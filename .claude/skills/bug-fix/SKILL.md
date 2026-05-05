---
name: bug-fix
description: |
  Bug 定位与修复 Skill。给测试工程师 Agent 使用，在发现 Bug 后进行系统化定位、分析和修复。覆盖日志分析、代码追踪、数据验证、接口测试等方法，以及 RuoYi-Vue-Plus 常见 Bug 模式和修复规范。
---

# Bug 定位与修复 Skill

> 版本：v1.0 | 适配 RuoYi-Vue-Plus 5.6.0
> 后端：Spring Boot 3.5.12 + Java 17 + MyBatis-Plus + Sa-Token
> 前端：Vue 3 + Element Plus + TypeScript

---

## 一、Bug 定位方法

### 1.1 定位流程（严格执行）

```
Bug 报告
  │
  ▼
复现问题（确认是真实 Bug）
  │
  ▼
判断 Bug 类型（前端/后端/接口/配置）
  │
  ├─ 前端 Bug ──→ 浏览器 DevTools 定位
  ├─ 后端 Bug ──→ 日志 + 代码追踪定位
  ├─ 接口 Bug ──→ curl/Postman 验证 API
  └─ 配置 Bug ──→ 检查配置文件
  │
  ▼
根因分析
  │
  ▼
制定修复方案
  │
  ▼
实施修复
  │
  ▼
验证修复
  │
  ▼
回归测试
```

### 1.2 日志分析

**日志文件位置：**

```bash
# 后端日志目录
ls -la {BACKEND_DIR}/logs/

# 查看最新日志
tail -f logs/sys-info.log        # 信息日志
tail -f logs/sys-error.log       # 错误日志

# 搜索关键字
grep -rn "NullPointerException" logs/
grep -rn "ServiceException" logs/
grep -rn "ERROR" logs/sys-error.log | tail -50
```

**日志分析要点：**

| 关键信息 | 说明 |
|----------|------|
| 异常类名 | `NullPointerException`、`ServiceException` 等 |
| 异常消息 | 业务异常的具体描述 |
| 堆栈跟踪 | 从上到下找到自己写的代码行 |
| 时间戳 | 确认是否为最新请求 |
| 请求路径 | 确认是哪个接口出错 |

### 1.3 代码追踪（后端）

从报错入口逐步追踪：

```
Controller（入口）
  ↓ 接收请求参数
  ↓ 调用 Service 方法
Service（业务逻辑）
  ↓ 业务校验
  ↓ 调用 Mapper 方法
Mapper（数据访问）
  ↓ SQL 执行
  ↓ 返回结果
```

**追踪命令：**

```bash
# 在后端项目中搜索相关类
cd {BACKEND_DIR}/

# 搜索 Controller
find . -name "*Controller.java" -exec grep -l "报错路径" {} \;

# 搜索 Service
find . -name "*Service*.java" -exec grep -l "相关方法名" {} \;

# 搜索 Mapper XML
find . -name "*.xml" -exec grep -l "相关SQL" {} \;
```

### 1.4 接口测试

```bash
# 登录获取 Token
TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H 'Content-Type: application/json' \
  -H 'clientid: e5cd7e4891bf95d1d19206ce24a7b32e' \
  -d '{"tenantId":"000000","username":"admin","password":"admin123"}' \
  | jq -r '.data.access_token')

# 测试接口
curl -v http://localhost:8080/{module}/{feature}/list?pageNum=1&pageSize=10 \
  -H "Authorization: Bearer $TOKEN" \
  -H "clientid: e5cd7e4891bf95d1d19206ce24a7b32e"
```

### 1.5 前端调试

| 调试方法 | 操作 |
|----------|------|
| 网络请求 | F12 → Network → 查看请求/响应 |
| 控制台错误 | F12 → Console → 查看红色报错 |
| 断点调试 | F12 → Sources → 设置断点 |
| Vue DevTools | 安装 Vue DevTools 浏览器扩展 |
| 状态检查 | Pinia DevTools 查看 Store 状态 |

### 1.6 数据验证

```bash
# 连接 MySQL 验证数据
mysql -u root -p -e "SELECT * FROM {table_name} WHERE id = {id};"

# 检查数据状态
mysql -u root -p -e "SELECT id, status, del_flag, tenant_id FROM {table_name} LIMIT 10;"
```

---

## 二、Bug 分析报告模板

```markdown
# Bug 分析报告

## Bug 信息
- 标题：{一句话描述 Bug}
- 优先级：P0（阻塞）/ P1（严重）/ P2（一般）/ P3（轻微）
- 分类：前端 / 后端 / 接口 / 配置
- 发现时间：{YYYY-MM-DD HH:mm}
- 报告人：{Agent 名称}

## 问题定位

### 复现步骤
1. 登录系统，使用 {账号}
2. 进入 {菜单路径}
3. 执行 {操作}
4. 出现 {异常现象}

### 错误现象
- 前端表现：{页面显示什么}
- 接口响应：{API 返回什么}
- 后端日志：{关键日志片段}

### 根因分析
{分析过程和结论}

### 影响范围
- 影响模块：{哪些模块受影响}
- 影响用户：{哪些用户受影响}
- 影响数据：{是否造成数据错误}

## 修复方案

### 修改文件清单
| 文件 | 修改内容 | 修改原因 |
|------|----------|----------|
| | | |

### 修复代码
\```java
// 关键代码片段
\```

## 验证方案
- 验证步骤：
  1. {步骤1}
  2. {步骤2}
- 回归范围：{需要回归测试的模块}
- 验证结果：PASS / FAIL
```

---

## 三、RuoYi 常见 Bug 模式

### 3.1 权限相关

| # | Bug 模式 | 表现 | 排查方法 | 修复方式 |
|---|---------|------|----------|----------|
| 1 | @SaCheckPermission 遗漏 | 接口返回 403 或所有人可访问 | 检查 Controller 方法注解 | 添加权限注解 |
| 2 | 权限字符串不匹配 | 有权限注解但仍然 403 | 对比 `sys_menu.perms` 和注解值 | 统一权限字符串 |
| 3 | @SaIgnore 误用 | 需要登录的接口变成公开 | 检查是否有误加 @SaIgnore | 移除误加的注解 |
| 4 | v-hasPermi 前后端不一致 | 按钮显示但接口 403 | 对比前端指令和后端注解 | 统一权限字符串 |

### 3.2 数据相关

| # | Bug 模式 | 表现 | 排查方法 | 修复方式 |
|---|---------|------|----------|----------|
| 5 | Bo/Vo 字段缺失 | 前端显示空值 | 对比 Entity、Vo、前端字段名 | 补充缺失字段 |
| 6 | LambdaQueryWrapper 条件错误 | 查询结果不对 | 打印 SQL 日志查看实际 SQL | 检查条件构建逻辑 |
| 7 | 多租户数据越权 | A 租户看到 B 租户数据 | 检查 Entity 是否继承 TenantEntity | 确认租户隔离配置 |
| 8 | 逻辑删除未生效 | 已删除的数据仍显示 | 检查 @TableLogic 注解 | 添加注解 |
| 9 | 分页参数未传递 | 前端不分页或分页异常 | 检查 Bo 是否继承 BaseEntity | 确认分页参数传递 |

### 3.3 缓存相关

| # | Bug 模式 | 表现 | 排查方法 | 修复方式 |
|---|---------|------|----------|----------|
| 10 | 缓存与数据库不一致 | 页面显示旧数据 | 直接查数据库对比 | 更新后清除缓存 |
| 11 | 缓存 Key 未包含租户ID | 跨租户缓存串数据 | 检查 @Cacheable key | key 中加入 tenant_id |
| 12 | 缓存穿透 | 大量请求打到数据库 | 检查空值是否缓存 | 空值也缓存（短时间） |

### 3.4 前端相关

| # | Bug 模式 | 表现 | 排查方法 | 修复方式 |
|---|---------|------|----------|----------|
| 13 | API 路径不匹配 | 接口 404 | 对比前端 API 文件和后端路径 | 统一 API 路径 |
| 14 | 表单校验规则缺失 | 非法数据提交成功 | 检查 :rules 和 prop | 补充校验规则 |
| 15 | 字典值未回显 | 下拉框或标签显示 code | 检查字典类型和 @Translation | 确认字典配置 |
| 16 | 分页未重置 pageNum | 翻页后搜索结果不对 | 检查 handleQuery 方法 | 搜索时重置 pageNum=1 |

---

## 四、修复规范

### 4.1 修复原则

| 原则 | 说明 |
|------|------|
| 最小化修改 | 只改必要的代码，不做无关重构 |
| 根因修复 | 不治标不治本（如：表单校验失败 → 加校验规则，而非前端弹提示） |
| 不引入新 Bug | 修复一个 Bug 不能制造另一个 |
| 保持风格一致 | 新代码与项目现有代码风格一致 |

### 4.2 修复流程

```
1. 确认 Bug → 复现问题
2. 定位根因 → 日志/代码/数据分析
3. 制定方案 → 评估影响范围
4. 实施修复 → 最小化修改
5. 自测验证 → 跑单元测试 + 手动验证
6. 回归测试 → 相关模块测试
7. 更新文档 → 经验库记录
```

### 4.3 修复后必须做的事

- [ ] 跑修复模块的单元测试
- [ ] 手动验证 Bug 已修复
- [ ] 检查是否引入新 Bug
- [ ] 大改动需技术经理审核
- [ ] 更新经验库（有参考价值的 Bug）

### 4.4 代码修复模板

**后端修复：**
```java
// 修复前（❌）
public FeatureDetailVo selectDetailById(Long id) {
    return featureMapper.selectVoById(id, FeatureDetailVo.class);
    // 缺少空值判断，id 不存在时返回 null，后续 NPE
}

// 修复后（✅）
public FeatureDetailVo selectDetailById(Long id) {
    FeatureDetailVo vo = featureMapper.selectVoById(id, FeatureDetailVo.class);
    if (vo == null) {
        throw new ServiceException("记录不存在");
    }
    return vo;
}
```

**前端修复：**
```typescript
// 修复前（❌）
const handleSubmit = () => {
  addFeature(formData.value).then(res => {
    // 没有处理错误
  });
};

// 修复后（✅）
const handleSubmit = async () => {
  try {
    await addFeature(formData.value);
    proxy?.$modal.msgSuccess("新增成功");
    getList();
  } catch (e) {
    // 错误已由全局拦截器处理
  }
};
```

---

## 五、Bug 修复检查清单

### 5.1 修复前检查

- [ ] Bug 已确认复现（非操作错误）
- [ ] 已收集完整的错误信息（日志/截图/请求响应）
- [ ] 已判断 Bug 类型（前端/后端/接口/配置）
- [ ] 已定位到根因（不是猜测）

### 5.2 修复中检查

- [ ] 修改范围最小化（不改动无关代码）
- [ ] 新代码符合项目编码规范
- [ ] 不引入新的依赖
- [ ] 不破坏现有接口契约

### 5.3 修复后检查

- [ ] 单元测试通过
- [ ] 手动验证 Bug 已修复
- [ ] 回归测试相关功能正常
- [ ] 经验库已更新（值得记录的 Bug）
- [ ] 修复代码已提交

---

## 六、Bug 优先级定义

| 优先级 | 定义 | 响应时间 | 示例 |
|--------|------|----------|------|
| P0 | 阻塞 | 立即 | 系统无法启动、核心流程完全不可用 |
| P1 | 严重 | 2小时内 | 数据丢失、安全漏洞、主要功能异常 |
| P2 | 一般 | 当天内 | 次要功能异常、UI 显示问题 |
| P3 | 轻微 | 下个迭代 | 样式瑕疵、文案错误、优化建议 |

---

## 七、经验库写入规范

```markdown
## Bug 修复
- [{日期}] {原则性描述}。{具体场景说明}
```

**示例：**
```markdown
## Bug 修复
- [260430] 修改/删除操作前必须校验记录是否存在且属于当前租户。曾因直接根据前端 ID 操作导致越权修改其他租户数据
- [260430] Bo 继承 BaseEntity 后 getParams() 返回的 Map 需判空。null Map 在 put 操作时会 NPE
```
