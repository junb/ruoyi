---
name: ai-test-generation
description: |
  AI 自动生成测试用例 Skill。给测试工程师 Agent 使用，在代码变更后自动分析变更范围、生成边界测试用例和安全测试用例。覆盖 API 变更、查询变更、状态变更、权限变更的自动用例生成规则，支持模糊测试和优先级排序。
---

# AI 自动生成测试用例 Skill

> 版本：v1.0 | 适配 RuoYi-Vue-Plus 5.6.0
> 后端：Spring Boot 3.5.12 + Java 17 + MyBatis-Plus + Sa-Token + 多租户

---

## 一、变更分析

### 1.1 变更检测

```bash
# 获取变更文件
cd {BACKEND_DIR}/
git diff --name-only HEAD~1

# 获取变更内容
git diff HEAD~1

# 按模块分类
git diff --name-only HEAD~1 | grep "ruoyi-modules/" | cut -d'/' -f2 | sort | uniq

# 获取变更的类和方法
git diff HEAD~1 -- '*.java' | grep "^@@" | head -20
```

### 1.2 变更类型识别

```
变更文件
  │
  ├─ *Controller.java → API 变更
  ├─ *Service.java / *ServiceImpl.java → 业务逻辑变更
  ├─ *Mapper.java / *Mapper.xml → 数据访问变更
  ├─ *Entity.java / Bo.java / Vo.java → 数据模型变更
  ├─ *Config.java / application*.yml → 配置变更
  └─ *.sql → 数据库变更
```

### 1.3 影响范围分析

| 变更类型 | 分析方法 | 影响范围 |
|----------|----------|----------|
| Controller 变更 | 检查方法签名、参数、权限注解 | 前端 API 调用 |
| Service 变更 | 检查调用链（谁调用了这个方法） | 其他 Service、Controller |
| Mapper 变更 | 检查 SQL 变更（条件/字段/表） | 对应 Service 的查询结果 |
| Entity 变更 | 检查关联的 Bo/Vo/Mapper | 数据映射、前端显示 |
| 配置变更 | 检查哪些模块使用该配置 | 所有引用该配置的模块 |

### 1.4 边界条件识别

对每个变更方法，分析以下边界：

```
输入参数：
  ├── 空值（null）
  ├── 空集合（[]）
  ├── 空字符串（""）
  ├── 超长字符串
  ├── 特殊字符（SQL注入/XSS/路径穿越）
  ├── 超大数值
  ├── 负数（不应为负的字段）
  └── 类型不匹配

业务逻辑：
  ├── 数据不存在
  ├── 数据已删除（del_flag='2'）
  ├── 数据属于其他租户
  ├── 数据属于其他部门
  ├── 状态不允许操作
  ├── 重复操作
  └── 并发操作

环境依赖：
  ├── Redis 不可用
  ├── 数据库连接超时
  ├── 文件系统满
  └── 外部接口超时
```

---

## 二、边界用例自动生成规则

### 2.1 API 变更

**触发条件：** Controller 方法新增/修改/删除

| # | 用例名称 | 测试输入 | 预期结果 | 类型 |
|---|---------|----------|----------|------|
| 1 | 空参数请求 | `{}`（Body 为空对象） | 400 或校验提示 | 边界 |
| 2 | null 值参数 | `{"name": null}` | 400 或"不能为空" | 边界 |
| 3 | 超长字符串 | `{"name": "a".repeat(10000)}` | 400 或长度限制提示 | 边界 |
| 4 | 特殊字符 - SQL注入 | `{"name": "' OR 1=1 --"}` | 正常返回（不注入成功） | 安全 |
| 5 | 特殊字符 - XSS | `{"name": "<script>alert(1)</script>"}` | 内容被转义或清洗 | 安全 |
| 6 | 未登录访问 | 无 Token | 401 未认证 | 安全 |
| 7 | 无权限访问 | 普通用户 Token | 403 无权限 | 安全 |
| 8 | 参数类型错误 | `{"id": "abc"}` | 400 参数格式错误 | 边界 |
| 9 | 额外字段 | `{"name": "test", "extra": "xxx"}` | 额外字段被忽略 | 边界 |
| 10 | 重复提交 | 快速连续请求两次 | 第二次返回防重提示 | 边界 |

### 2.2 查询变更

**触发条件：** 查询条件修改、SQL 变更、Mapper 修改

| # | 用例名称 | 测试输入 | 预期结果 | 类型 |
|---|---------|----------|----------|------|
| 1 | 空结果集 | 不存在的查询条件 | 返回空列表（rows=[]） | 边界 |
| 2 | 单条结果 | 精确匹配一条 | 返回 1 条数据 | 正常 |
| 3 | 大量数据 | 返回超过 1000 条 | 分页正常，性能可接受 | 性能 |
| 4 | 分页边界 - 第一页 | pageNum=1, pageSize=10 | 正常返回 | 边界 |
| 5 | 分页边界 - 超大页码 | pageNum=99999 | 返回空列表 | 边界 |
| 6 | 分页边界 - pageSize=1 | pageSize=1 | 正常返回 1 条 | 边界 |
| 7 | 排序字段为空 | 不传排序参数 | 使用默认排序 | 正常 |
| 8 | 排序字段非法 | orderByColumn=nonexistent | 忽略或使用默认排序 | 安全 |
| 9 | 时间范围查询 | beginTime > endTime | 返回空列表或提示 | 边界 |
| 10 | 模糊查询空值 | keyword="" | 返回全部或忽略条件 | 边界 |

### 2.3 状态变更

**触发条件：** 状态流转逻辑修改

| # | 用例名称 | 测试输入 | 预期结果 | 类型 |
|---|---------|----------|----------|------|
| 1 | 非法状态转换 | 已完成 → 修改为待支付 | 拒绝操作，提示状态错误 | 业务 |
| 2 | 重复状态设置 | 已完成 → 再次设为已完成 | 幂等处理或提示 | 业务 |
| 3 | 并发状态修改 | 两个请求同时修改 | 仅一个成功（乐观锁） | 并发 |
| 4 | 已删除数据操作 | del_flag='2' 的数据修改 | 提示"记录不存在" | 边界 |
| 5 | 状态为空 | status=null | 默认状态或校验提示 | 边界 |

### 2.4 权限变更

**触发条件：** 权限注解修改、角色配置变更、数据权限变更

| # | 用例名称 | 测试输入 | 预期结果 | 类型 |
|---|---------|----------|----------|------|
| 1 | 无权限用户 | 无对应权限的 Token | 403 Forbidden | 安全 |
| 2 | 未登录 | 无 Token | 401 Unauthorized | 安全 |
| 3 | 越权访问 - 跨部门 | A 部门用户访问 B 部门数据 | 数据不可见 | 安全 |
| 4 | 越权访问 - 跨租户 | A 租户访问 B 租户数据 | 数据不可见 | 安全 |
| 5 | 超级管理员 | admin 账号 | 可访问所有数据 | 正常 |
| 6 | 权限字符串错误 | perms 值与数据库不匹配 | 403 | 安全 |

---

## 三、模糊测试方法

### 3.1 随机参数组合

对接口参数进行随机组合，发现未覆盖的边界：

```
参数1: [null, "", "a", "a"*100, "特殊字符", "SQL注入", 正常值]
参数2: [null, 0, -1, 999999, 正常值]
参数3: [null, "0", "1", "2", "999", 正常值]

随机组合测试 → 发现异常组合
```

### 3.2 边界值组合

```markdown
| 组合 | 参数1 | 参数2 | 预期 |
|------|-------|-------|------|
| 最小+最小 | pageSize=1 | pageNum=1 | 正常 |
| 最小+最大 | pageSize=1 | pageNum=MAX | 空 |
| 最大+最小 | pageSize=500 | pageNum=1 | 正常 |
| 最大+最大 | pageSize=500 | pageNum=MAX | 空 |
| 零值 | pageSize=0 | pageNum=0 | 校验提示 |
| 负数 | pageSize=-1 | pageNum=-1 | 校验提示 |
```

### 3.3 异常输入组合

| 输入类型 | 示例 | 用途 |
|----------|------|------|
| Unicode | `\u0000`、emoji | 编码问题 |
| 超长 Base64 | 10000 字符的 Base64 | 内存/长度 |
| 路径穿越 | `../../etc/passwd` | 文件操作安全 |
| JSON 注入 | `{"__proto__": {}}` | 原型链污染 |
| XML 注入 | `<?xml version...` | XXE 攻击 |
| 换行符 | `test\r\nX-Header: hack` | HTTP 头注入 |

---

## 四、AI 生成用例模板

### 4.1 生成报告模板

```markdown
# AI 自动生成测试用例

## 变更分析

### 变更文件
| 文件 | 变更类型 | 变更方法 |
|------|----------|----------|
| FeatureController.java | 修改 | add(), edit() |
| FeatureServiceImpl.java | 修改 | insertByBo() |

### 影响范围
- 直接影响：Feature 模块的 CRUD 接口
- 间接影响：引用 Feature 服务的其他模块
- 安全影响：涉及新增/修改接口的权限和输入校验

### 边界条件
- 新增接口：空参数、超长名称、重复名称
- 修改接口：修改不存在的记录、修改其他租户数据
- 状态变更：停用的记录是否可编辑

## 自动生成用例

### API 测试用例

| # | 用例名称 | 测试输入 | 预期结果 | 类型 | 优先级 |
|---|---------|----------|----------|------|--------|
| 1 | 空参数新增 | `{}` | 400 | 边界 | P1 |
| 2 | SQL注入 - 名称 | `{"name":"' OR 1=1"}` | 正常返回 | 安全 | P0 |
| 3 | XSS - 名称 | `{"name":"<script>"}` | 内容转义 | 安全 | P0 |
| 4 | 未登录新增 | 无 Token | 401 | 安全 | P0 |
| 5 | 无权限新增 | 普通用户 Token | 403 | 安全 | P0 |
| 6 | 越权修改 | 其他租户数据的 ID | 提示不存在 | 安全 | P0 |
| 7 | 重复名称新增 | 已存在的 name | 提示重复或成功 | 业务 | P1 |
| 8 | 超长名称 | 1000字符 | 长度校验提示 | 边界 | P2 |
| 9 | 修改不存在记录 | id=999999 | 提示不存在 | 边界 | P1 |
| 10 | 并发新增 | 同时提交相同数据 | 幂等或唯一约束 | 并发 | P2 |

### 查询测试用例

| # | 用例名称 | 测试输入 | 预期结果 | 类型 | 优先级 |
|---|---------|----------|----------|------|--------|
| 11 | 空结果查询 | name=不存在的值 | rows=[] | 边界 | P1 |
| 12 | 大量数据分页 | pageSize=500 | 正常返回 | 性能 | P2 |
| 13 | 非法排序字段 | orderByColumn=xxx | 忽略或默认排序 | 安全 | P1 |
| 14 | 跨租户查询 | 用其他租户 Token | 数据不可见 | 安全 | P0 |

## 用例统计

| 类型 | 数量 |
|------|------|
| 安全测试（P0） | N |
| 业务测试（P1） | N |
| 边界测试（P2） | N |
| 性能测试（P2） | N |
| **合计** | **N** |
```

### 4.2 生成的测试代码模板

```java
/**
 * AI 自动生成的安全测试用例
 * 针对：FeatureController.add()
 */
@SpringBootTest
@ActiveProfiles("dev")
@Transactional
class FeatureSecurityTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    @DisplayName("安全 - SQL注入防护")
    void testSqlInjection() {
        Map<String, String> body = Map.of("name", "' OR 1=1 --");
        HttpEntity<Map<String, String>> request = new HttpEntity<>(body, authHeaders());

        ResponseEntity<String> res = restTemplate.postForEntity(
            "/{module}/{feature}", request, String.class);

        // 应正常处理，不执行注入
        assertTrue(res.getBody().contains("\"code\":200") ||
                   res.getBody().contains("\"code\":400"));
    }

    @Test
    @DisplayName("安全 - XSS 防护")
    void testXssProtection() {
        Map<String, String> body = Map.of("name", "<script>alert(1)</script>");
        HttpEntity<Map<String, String>> request = new HttpEntity<>(body, authHeaders());

        ResponseEntity<String> res = restTemplate.postForEntity(
            "/{module}/{feature}", request, String.class);

        // 返回的数据中 script 标签应被转义
        if (res.getBody().contains("\"code\":200")) {
            assertFalse(res.getBody().contains("<script>alert(1)</script>"));
        }
    }

    @Test
    @DisplayName("安全 - 未登录访问应401")
    void testUnauthorizedAccess() {
        ResponseEntity<String> res = restTemplate.getForEntity(
            "/{module}/{feature}/list", String.class);

        assertTrue(res.getBody().contains("\"code\":401"));
    }

    @Test
    @DisplayName("安全 - 跨租户数据隔离")
    void testTenantIsolation() {
        // 使用租户A的 Token 查询
        ResponseEntity<String> res = restTemplate.exchange(
            "/{module}/{feature}/list",
            HttpMethod.GET,
            new HttpEntity<>(tenantAHeaders()),
            String.class);

        // 确认结果中不包含租户B的数据
        // 具体验证逻辑取决于业务
    }
}
```

---

## 五、用例优先级排序

### 5.1 优先级定义

| 优先级 | 定义 | 必须测试 | 说明 |
|--------|------|----------|------|
| **P0** | 安全相关 | ✅ 是 | 越权、注入、XSS、未认证 |
| **P1** | 核心业务 | ✅ 是 | 主流程、必填校验、数据完整性 |
| **P2** | 边界条件 | ⚠️ 建议测试 | 超长、空值、并发、异常输入 |
| **P3** | 异常场景 | 📋 按需 | 性能、兼容性、极端情况 |

### 5.2 自动排序规则

```
生成的用例 → 按以下规则排序：

1. P0 安全用例排最前（越权、注入、XSS）
2. P1 核心业务用例次之（主流程、校验）
3. P2 边界用例再次（超长、空值、并发）
4. P3 异常用例最后（性能、兼容）
```

### 5.3 用例筛选建议

| 可用时间 | 建议范围 |
|----------|----------|
| 5 分钟 | 仅 P0 安全用例 |
| 15 分钟 | P0 + P1 核心用例 |
| 30 分钟 | P0 + P1 + P2 边界用例 |
| 1 小时+ | P0 + P1 + P2 + P3 全量 |

---

## 六、执行流程

### 6.1 完整流程

```
1. git diff 分析变更文件
     │
     ▼
2. 识别变更类型（API/查询/状态/权限）
     │
     ▼
3. 读取变更代码，分析业务逻辑
     │
     ▼
4. 根据变更类型 + 边界规则生成用例
     │
     ▼
5. 按优先级排序
     │
     ▼
6. 输出用例报告
     │
     ▼
7. （可选）生成测试代码并执行
     │
     ▼
8. 输出测试结果
```

### 6.2 与其他 Skill 的协作

| Skill | 协作方式 |
|-------|----------|
| test-planning | 生成的用例可作为测试计划的补充 |
| integration-testing | P0/P1 用例可转为集成测试代码 |
| regression-testing | 变更分析结果用于确定回归范围 |
| bug-fix | 修复后的变更重新生成用例验证 |

---

## 七、经验库写入规范

```markdown
## AI 测试生成
- [{日期}] {原则性描述}。{具体场景说明}
```

**示例：**
```markdown
## AI 测试生成
- [260430] 新增/修改接口必须生成越权测试用例（跨租户、跨部门）。曾因缺少越权测试导致上线后数据泄露
- [260430] 涉及状态流转的代码变更必须覆盖所有非法状态转换路径。曾因遗漏状态检查导致已删除的数据被修改
```
