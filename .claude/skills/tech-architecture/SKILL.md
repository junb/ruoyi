---
name: tech-architecture
description: |
  技术经理概念设计Skill。当需要进行系统概要设计、技术选型、架构设计、
  数据库设计、API设计、风险评估时触发。
  适配RuoYi-Vue-Plus 5.6.0技术栈。
---

# 技术经理概念设计 Skill

## 角色定位

作为技术经理，负责将PRD需求转化为技术方案，输出系统概要设计文档，确保技术方案可行、合理、可落地。

## 一、系统概要设计模板

```markdown
# 系统概要设计：{项目/模块名称}

## 1. 设计目标

### 1.1 业务目标
<!-- 本次设计要解决的业务问题 -->

### 1.2 技术目标
- 代码结构清晰，遵循RuoYi框架规范
- 支持多租户数据隔离
- 接口性能满足需求指标
- 安全性符合企业级标准

### 1.3 约束条件
- 技术栈：Spring Boot 3.5.12 + Java 17 + MyBatis-Plus 3.5.16 + Sa-Token 1.44.0
- 前端：Vue 3 + Element Plus 2.13.5 + Pinia + UnoCSS + TypeScript
- 框架版本：RuoYi-Vue-Plus 5.6.0

## 2. 技术架构

### 2.1 架构图（文字描述）

```
┌─────────────────────────────────────────────────────────┐
│                      前端 (Vue 3)                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ 页面组件  │ │ API请求  │ │ 状态管理  │ │ 路由管理  │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
├─────────────────────────────────────────────────────────┤
│                    网关 / Nginx                           │
├─────────────────────────────────────────────────────────┤
│                  后端 (Spring Boot 3.5)                   │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Controller 层                        │   │
│  │  （接口定义、参数校验、权限注解）                    │   │
│  ├──────────────────────────────────────────────────┤   │
│  │              Service 层                          │   │
│  │  （业务逻辑、事务管理、缓存处理）                    │   │
│  ├──────────────────────────────────────────────────┤   │
│  │              Mapper 层 (MyBatis-Plus)            │   │
│  │  （数据访问、SQL映射、分页查询）                    │   │
│  └──────────────────────────────────────────────────┘   │
├──────────────┬──────────────┬───────────────────────────┤
│   MySQL      │    Redis     │     OSS (文件存储)        │
│  (主数据)     │  (缓存/会话)  │    (附件/图片)           │
└──────────────┴──────────────┴───────────────────────────┘
```

### 2.2 技术选型及理由

| 技术组件 | 版本 | 选型理由 | 替代方案 |
|----------|------|----------|----------|
| Spring Boot | 3.5.12 | RuoYi框架基础，生态成熟 | - |
| Sa-Token | 1.44.0 | RuoYi集成，轻量级认证授权 | Spring Security |
| MyBatis-Plus | 3.5.16 | RuoYi集成，简化CRUD，代码生成 | JPA/MyBatis原生 |
| Redis | - | 缓存、会话、分布式锁 | Caffeine(本地缓存) |
| Vue 3 | - | RuoYi前端框架，Composition API | - |
| Element Plus | 2.13.5 | RuoYi UI组件库 | Ant Design Vue |

## 3. 功能模块划分

### 3.1 模块架构

```
ruoyi-modules/
├── ruoyi-{module}/                    # 新业务模块
│   ├── ruoyi-{module}-api/            # 模块API（接口定义、DTO）
│   └── ruoyi-{module}-biz/            # 模块业务实现
│       ├── src/main/java/com/ruoyi/{module}/
│       │   ├── controller/            # 控制器
│       │   ├── domain/                # 实体类
│       │   │   ├── entity/            # 数据库实体
│       │   │   ├── bo/                # 业务对象
│       │   │   └── vo/                # 视图对象
│       │   ├── mapper/                # 数据访问层
│       │   └── service/               # 服务层
│       │       ├── impl/              # 服务实现
│       │       └── {Service}.java     # 服务接口
│       └── src/main/resources/
│           └── mapper/{module}/       # MyBatis XML
```

### 3.2 模块依赖关系

| 模块 | 职责 | 依赖 | 被依赖 |
|------|------|------|--------|
| ruoyi-{module}-api | 接口定义、DTO、Feign客户端 | ruoyi-common-core | ruoyi-{module}-biz, 其他模块 |
| ruoyi-{module}-biz | 业务逻辑实现 | ruoyi-{module}-api, ruoyi-common | ruoyi-admin |
| ruoyi-common-redis | Redis工具封装 | ruoyi-common-core | 各业务模块 |
| ruoyi-common-mybatis | 数据库工具封装 | ruoyi-common-core | 各业务模块 |
| ruoyi-common-satoken | 认证授权工具 | ruoyi-common-core | 各业务模块 |

## 4. 数据流设计

### 4.1 核心业务数据流

```
用户操作 → 前端组件 → API请求(axios) → Controller(参数校验)
    → Service(业务逻辑) → Mapper(数据访问) → MySQL(持久化)
                                                    ↓
    响应 ← VO组装 ← 缓存处理(Redis) ←──────────────┘
```

**详细流程：**

```
1. 用户在页面触发操作（点击按钮、提交表单）
2. 前端调用 API → 发送 HTTP 请求到后端
3. Controller 层：
   - @SaCheckPermission 权限校验
   - @Valid 参数校验
   - 调用 Service 层
4. Service 层：
   - 业务规则校验
   - 数据处理/转换
   - 事务管理（@Transactional）
   - 调用 Mapper 层
5. Mapper 层：
   - MyBatis-Plus 执行SQL
   - 数据权限过滤（@DataScope）
6. 返回数据：
   - Service 组装 VO
   - Controller 返回 R<T> 统一响应
```

### 4.2 外部接口数据流

```
后端服务 → HTTP/RPC调用 → 外部系统
    ↓
响应处理 → 数据转换 → 业务处理
    ↓
异常处理 → 重试/降级/告警
```

| 外部系统 | 通信协议 | 调用方式 | 超时设置 | 重试策略 |
|----------|----------|----------|----------|----------|
| {系统名} | HTTP/REST | 同步调用 | 5秒 | 3次，间隔1秒 |
| {系统名} | RPC | 同步调用 | 3秒 | 不重试 |

## 5. 数据库设计概要

### 5.1 ER图概要（实体和关系描述）

```
{主实体}
  ├── 1:N → {子实体1}（一对多关系）
  ├── N:N → {关联实体}（多对多关系，通过中间表）
  └── N:1 → {父实体}（多对一关系）

关系说明：
- {主实体} 与 {子实体}：一个{主}可以拥有多个{子}，通过 {主}_id 外键关联
- {实体A} 与 {实体B}：多对多关系，通过 {中间表} 关联
```

### 5.2 核心表清单

| 表名 | 说明 | 预估数据量 | 关键字段 | 索引策略 |
|------|------|------------|----------|----------|
| {module}_main | 主表 | 10万/年 | id, name, status, dept_id, create_by | idx_dept_id, idx_create_time |
| {module}_detail | 明细表 | 50万/年 | id, main_id, ... | idx_main_id |
| {module}_log | 日志表 | 100万/年 | id, main_id, action, ... | idx_main_id, idx_create_time |

### 5.3 表设计规范

**字段命名规范：**
- 表名：`{模块}_{实体名}`，全小写下划线
- 主键：`id`（BIGINT，雪花算法）
- 通用字段：
  ```sql
  tenant_id        VARCHAR(20)     COMMENT '租户编号'
  create_dept      BIGINT          COMMENT '创建部门'
  create_by        BIGINT          COMMENT '创建者'
  create_time      DATETIME        COMMENT '创建时间'
  update_by        BIGINT          COMMENT '更新者'
  update_time      DATETIME        COMMENT '更新时间'
  del_flag         CHAR(1)         COMMENT '删除标志（0代表存在 2代表删除）'
  ```

**索引设计原则：**
- 主键索引：每张表必须有
- 外键字段：建索引
- 高频查询条件：建组合索引
- 遵循最左前缀原则
- 单表索引数量不超过5个

### 5.4 SQL建表模板

```sql
CREATE TABLE {module}_{table} (
    id            BIGINT          NOT NULL                  COMMENT '主键ID',
    tenant_id     VARCHAR(20)     DEFAULT '000000'          COMMENT '租户编号',
    {business_field}  {type}      {constraint}              COMMENT '{业务字段说明}',
    create_dept   BIGINT          DEFAULT NULL              COMMENT '创建部门',
    create_by     BIGINT          DEFAULT NULL              COMMENT '创建者',
    create_time   DATETIME        DEFAULT NULL              COMMENT '创建时间',
    update_by     BIGINT          DEFAULT NULL              COMMENT '更新者',
    update_time   DATETIME        DEFAULT NULL              COMMENT '更新时间',
    del_flag      CHAR(1)         DEFAULT '0'               COMMENT '删除标志（0存在 2删除）',
    PRIMARY KEY (id)
) ENGINE=InnoDB COMMENT='{表说明}';

-- 索引
CREATE INDEX idx_{table}_{field} ON {module}_{table} ({field});
```

## 6. API接口概要

### 6.1 接口设计规范

**RESTful风格：**

| 操作 | HTTP方法 | URL格式 | 说明 |
|------|----------|---------|------|
| 列表查询 | GET | /api/{module}/{entity}/list | 分页查询，参数通过Query传递 |
| 详情查询 | GET | /api/{module}/{entity}/{id} | 获取单条详情 |
| 新增 | POST | /api/{module}/{entity} | RequestBody传参 |
| 修改 | PUT | /api/{module}/{entity} | RequestBody传参 |
| 删除 | DELETE | /api/{module}/{entity}/{ids} | 支持批量，逗号分隔 |
| 导出 | POST | /api/{module}/{entity}/export | 返回Excel文件流 |

**统一响应格式：**
```json
{
    "code": 200,
    "msg": "操作成功",
    "data": {}
}
```

### 6.2 接口清单

| 模块 | 接口 | 方法 | 说明 | 权限标识 |
|------|------|------|------|----------|
| {模块} | /api/{module}/{entity}/list | GET | 分页列表 | {module}:{entity}:list |
| {模块} | /api/{module}/{entity}/{id} | GET | 详情 | {module}:{entity}:query |
| {模块} | /api/{module}/{entity} | POST | 新增 | {module}:{entity}:add |
| {模块} | /api/{module}/{entity} | PUT | 修改 | {module}:{entity}:edit |
| {模块} | /api/{module}/{entity}/{ids} | DELETE | 删除 | {module}:{entity}:remove |
| {模块} | /api/{module}/{entity}/export | POST | 导出 | {module}:{entity}:export |

### 6.3 分页查询接口规范

**请求参数：**
```json
{
    "pageNum": 1,
    "pageSize": 10,
    "orderByColumn": "createTime",
    "isAsc": "desc",
    "{业务字段}": "{筛选值}"
}
```

**响应格式：**
```json
{
    "code": 200,
    "msg": "查询成功",
    "rows": [{...}],
    "total": 100
}
```

## 7. 安全设计

### 7.1 认证授权

**认证机制（Sa-Token）：**
- 登录认证：`StpUtil.login(userId)`
- Token校验：请求头携带 `Authorization: Bearer {token}`
- Token刷新：自动续期 / 双Token机制
- 注销登录：`StpUtil.logout()`

**授权机制：**
- 菜单权限：`@SaCheckPermission("module:entity:list")`
- 角色权限：`@SaCheckRole("admin")`
- 数据权限：`@DataScope` 注解 + SQL过滤

### 7.2 数据安全

| 安全措施 | 实现方式 | 适用场景 |
|----------|----------|----------|
| 密码加密 | BCrypt | 用户密码存储 |
| 敏感字段加密 | AES | 身份证号、银行卡号 |
| 数据脱敏 | @JsonSerialize | 接口返回手机号、身份证 |
| SQL注入防护 | MyBatis-Plus参数化查询 | 所有数据库操作 |
| XSS防护 | 全局过滤器 | 所有输入参数 |
| 文件上传校验 | 扩展名+大小+内容校验 | 所有文件上传 |

### 7.3 接口安全

| 措施 | 说明 |
|------|------|
| 接口幂等性 | 重复提交校验（@RepeatSubmit） |
| 限流控制 | 防刷限流（@RateLimiter） |
| 参数校验 | @Valid + 自定义校验注解 |
| 操作日志 | @Log 记录关键操作 |
| 防篡改 | 关键接口签名校验 |

## 8. 性能设计

### 8.1 性能指标

| 指标 | 目标值 | 测量方式 |
|------|--------|----------|
| 接口响应时间(P99) | < 500ms | APM监控 |
| 接口响应时间(P95) | < 200ms | APM监控 |
| 并发用户数 | ≥ 500 | 压测工具 |
| 吞吐量(TPS) | ≥ 1000 | 压测工具 |
| 页面加载时间 | < 3s | 浏览器DevTools |
| 数据库慢查询 | < 100ms | MySQL慢查询日志 |

### 8.2 缓存策略

| 缓存场景 | Key格式 | 过期时间 | 更新策略 |
|----------|---------|----------|----------|
| 字典数据 | dict:{dictType} | 30分钟 | 修改时主动删除 |
| 配置数据 | config:{configKey} | 30分钟 | 修改时主动删除 |
| 用户信息 | user:{userId} | 30分钟 | 修改时主动更新 |
| 业务数据 | {module}:{entity}:{id} | 按业务设定 | 写穿/失效 |

**缓存使用规范：**
- 使用Redis工具类（Spring Cache + RedisTemplate）
- 避免缓存穿透（空值缓存）
- 避免缓存雪崩（随机过期时间）
- 热点Key处理（本地缓存兜底）

### 8.3 数据库优化

| 优化措施 | 适用场景 | 说明 |
|----------|----------|------|
| 索引优化 | 高频查询 | 根据慢查询日志优化 |
| 分页优化 | 大数据量 | 使用游标分页替代深分页 |
| 批量操作 | 批量插入/更新 | MyBatis-Plus saveBatch |
| 读写分离 | 高并发读 | 配置主从数据源 |
| 表分区 | 大数据量表 | 按时间/范围分区 |
| SQL优化 | 复杂查询 | 避免SELECT *，合理使用JOIN |

## 9. 技术风险

### 9.1 风险评估矩阵

| 风险 | 影响 | 概率 | 风险等级 | 缓解措施 | 负责人 | 状态 |
|------|------|------|----------|----------|--------|------|
| {风险描述} | 高/中/低 | 高/中/低 | 高/中/低 | {具体措施} | {负责人} | 待处理/已缓解 |

**风险等级判定：**
```
         高概率
            │
     中风险  │  高风险
            │
低概率 ─────┼───── 高概率
            │
     低风险  │  中风险
            │
         低概率
     低影响   高影响
```

### 9.2 常见技术风险清单

| 风险类别 | 风险描述 | 典型缓解措施 |
|----------|----------|-------------|
| 技术风险 | 新技术与现有框架不兼容 | 提前做POC验证 |
| 技术风险 | 第三方依赖版本冲突 | 锁定版本，统一管理 |
| 性能风险 | 数据量增长导致查询变慢 | 索引优化 + 分页 + 缓存 |
| 性能风险 | 并发写入导致数据不一致 | 乐观锁/分布式锁 |
| 安全风险 | 敏感数据泄露 | 加密存储 + 脱敏展示 |
| 安全风险 | 越权访问 | 权限校验 + 数据权限 |
| 进度风险 | 需求变更频繁 | 敏捷迭代，分期交付 |
| 进度风险 | 技术难题阻塞 | 预留buffer，提前攻关 |
```

---

## 二、RuoYi-Vue-Plus架构适配要点

### 2.1 新模块创建规范

**后端模块结构：**
```
ruoyi-modules/
└── ruoyi-{module}/
    ├── ruoyi-{module}-api/          # API模块（可被其他模块依赖）
    │   └── pom.xml
    └── ruoyi-{module}-biz/          # 业务实现模块
        └── pom.xml
```

**创建步骤：**
1. 在 `ruoyi-modules/` 下创建新模块目录
2. 编写各子模块的 `pom.xml`
3. 在 `ruoyi-admin/pom.xml` 中添加新模块依赖
4. 在根 `pom.xml` 的 `<modules>` 中注册
5. 在 Nacos 配置中添加模块配置

**ruoyi-admin pom.xml 添加依赖：**
```xml
<dependency>
    <groupId>com.ruoyi</groupId>
    <artifactId>ruoyi-{module}-biz</artifactId>
</dependency>
```

### 2.2 复用ruoyi-common公共能力

| 公共模块 | 提供能力 | 使用方式 |
|----------|----------|----------|
| ruoyi-common-core | 基础类、工具类、统一响应 | 直接依赖 |
| ruoyi-common-redis | Redis操作、分布式锁 | 直接依赖 |
| ruoyi-common-mybatis | 数据库操作、分页、数据权限 | 直接依赖 |
| ruoyi-common-satoken | 认证授权、权限校验 | 直接依赖 |
| ruoyi-common-web | Web配置、全局异常处理 | 通过ruoyi-admin间接依赖 |
| ruoyi-common-oss | 文件上传下载 | 直接依赖 |
| ruoyi-common-log | 操作日志 | 直接依赖 |
| ruoyi-common-excel | Excel导入导出 | 直接依赖 |

### 2.3 三层架构规范

**Controller层：**
```java
@Validated
@RequiredArgsConstructor
@RestController
@RequestMapping("/{module}/{entity}")
public class {Entity}Controller extends BaseController {

    private final I{Entity}Service {entity}Service;

    @SaCheckPermission("{module}:{entity}:list")
    @GetMapping("/list")
    public TableDataInfo<{Entity}Vo> list({Entity}Bo bo, PageQuery pageQuery) {
        return {entity}Service.queryPageList(bo, pageQuery);
    }

    @SaCheckPermission("{module}:{entity}:query")
    @GetMapping("/{id}")
    public R<{Entity}Vo> getInfo(@PathVariable Long id) {
        return R.ok({entity}Service.queryById(id));
    }

    @SaCheckPermission("{module}:{entity}:add")
    @Log(title = "{实体名称}", businessType = BusinessType.INSERT)
    @PostMapping()
    public R<Void> add(@Valid @RequestBody {Entity}Bo bo) {
        return toAjax({entity}Service.insertByBo(bo));
    }

    @SaCheckPermission("{module}:{entity}:edit")
    @Log(title = "{实体名称}", businessType = BusinessType.UPDATE)
    @PutMapping()
    public R<Void> edit(@Valid @RequestBody {Entity}Bo bo) {
        return toAjax({entity}Service.updateByBo(bo));
    }

    @SaCheckPermission("{module}:{entity}:remove")
    @Log(title = "{实体名称}", businessType = BusinessType.DELETE)
    @DeleteMapping("/{ids}")
    public R<Void> remove(@PathVariable List<Long> ids) {
        return toAjax({entity}Service.deleteWithValidByIds(ids));
    }
}
```

**Service层：**
```java
public interface I{Entity}Service {
    TableDataInfo<{Entity}Vo> queryPageList({Entity}Bo bo, PageQuery pageQuery);
    {Entity}Vo queryById(Long id);
    Boolean insertByBo({Entity}Bo bo);
    Boolean updateByBo({Entity}Bo bo);
    Boolean deleteWithValidByIds(List<Long> ids);
}
```

**Mapper层：**
```java
public interface {Entity}Mapper extends BaseMapperPlus<{Entity}, {Entity}Vo> {
}
```

### 2.4 多租户架构适配

**多租户实现方式：**
- 使用 `tenant_id` 字段实现数据隔离
- MyBatis-Plus 拦截器自动添加租户条件
- 部分表可配置忽略租户过滤

**忽略租户的表：**
```yaml
mybatis-plus:
  tenant:
    ignore-tables:
      - sys_dict_type
      - sys_dict_data
      - sys_config
      # 添加需要忽略租户的表
```

**手动忽略租户过滤：**
```java
// 临时忽略租户过滤
TenantHelper.ignore(() -> {
    // 这里的查询不会自动添加 tenant_id 条件
    return mapper.selectList();
});
```

### 2.5 Sa-Token权限体系适配

**权限校验注解：**
```java
// 需要特定权限
@SaCheckPermission("module:entity:list")

// 需要特定角色
@SaCheckRole("admin")

// 需要登录
@SaCheckLogin
```

**在代码中获取当前用户：**
```java
LoginUser loginUser = LoginHelper.getLoginUser();
Long userId = loginUser.getUserId();
Long deptId = loginUser.getDeptId();
```

### 2.6 MyBatis-Plus ORM策略

**代码生成：**
- 使用 RuoYi 代码生成器生成 Entity/Mapper/Service/Controller
- 生成后根据业务需求调整

**常用特性：**
- 自动分页：`PageQuery` + `TableDataInfo`
- 逻辑删除：`@TableLogic` + `del_flag`
- 自动填充：`create_by`, `create_time`, `update_by`, `update_time`
- 乐观锁：`@Version` + `version` 字段
- 字典翻译：`@DictFormat` 注解

### 2.7 Redis缓存策略

**使用Spring Cache注解：**
```java
@Cacheable(cacheNames = "{module}:{entity}", key = "#id")
public {Entity}Vo queryById(Long id) { ... }

@CacheEvict(cacheNames = "{module}:{entity}", key = "#bo.id")
public Boolean updateByBo({Entity}Bo bo) { ... }
```

**使用Redis工具类：**
```java
// 设置缓存
RedisUtils.setCacheObject(key, value, timeout, timeUnit);

// 获取缓存
RedisUtils.getCacheObject(key);

// 删除缓存
RedisUtils.deleteObject(key);
```

### 2.8 文件存储（OSS）策略

**上传接口：**
```java
// 使用RuoYi封装的OSS服务
MultipartFile file;
String url = OssHelper.upload(file);
```

**文件类型限制：**
| 类型 | 允许的扩展名 | 最大大小 |
|------|-------------|----------|
| 图片 | jpg, jpeg, png, gif, webp | 5MB |
| 文档 | pdf, doc, docx, xls, xlsx | 20MB |
| 压缩包 | zip, rar | 50MB |

---

## 三、技术选型评估框架

### 3.1 评估维度

| 维度 | 权重 | 评估标准 |
|------|------|----------|
| **成熟度** | 25% | 版本稳定性、生产环境验证、文档完善度 |
| **性能** | 20% | 响应时间、吞吐量、资源消耗 |
| **社区活跃度** | 15% | GitHub Star数、更新频率、Issue响应速度 |
| **学习成本** | 15% | 上手难度、文档质量、社区教程 |
| **兼容性** | 25% | 与现有技术栈集成难度、版本兼容性 |

### 3.2 技术选型评估模板

```markdown
# 技术选型评估：{技术名称}

## 1. 评估背景
<!-- 为什么需要引入这个技术？解决什么问题？ -->

## 2. 候选方案
| 方案 | 说明 | 官网 |
|------|------|------|
| 方案A | {描述} | {链接} |
| 方案B | {描述} | {链接} |
| 方案C | {描述} | {链接} |

## 3. 评估打分（每项1-5分）

| 评估维度 | 权重 | 方案A | 方案B | 方案C |
|----------|------|-------|-------|-------|
| 成熟度 | 25% | | | |
| 性能 | 20% | | | |
| 社区活跃度 | 15% | | | |
| 学习成本 | 15% | | | |
| 兼容性 | 25% | | | |
| **加权总分** | **100%** | | | |

## 4. POC验证
<!-- 如果需要，进行概念验证 -->

## 5. 最终结论
- **推荐方案：** {方案名}
- **推荐理由：** {详细理由}
- **风险提示：** {潜在风险}
```

---

## 四、风险评估清单

### 4.1 风险评估矩阵模板

```markdown
# 技术风险评估：{项目/模块名称}

## 风险矩阵

| # | 风险类别 | 风险描述 | 影响 | 概率 | 风险等级 | 缓解措施 | 应急方案 | 负责人 | 状态 |
|---|----------|----------|------|------|----------|----------|----------|--------|------|
| 1 | 技术风险 | | 高/中/低 | 高/中/低 | 高/中/低 | | | | 待处理 |

## 风险类别说明

### 技术风险
| 检查项 | 是/否 | 说明 |
|--------|-------|------|
| 是否引入新技术？ | | 需要POC验证 |
| 是否有第三方依赖？ | | 评估版本兼容性 |
| 是否涉及复杂算法？ | | 评估开发难度 |
| 是否有集成风险？ | | 接口联调可能出问题 |

### 性能风险
| 检查项 | 是/否 | 说明 |
|--------|-------|------|
| 是否有大数据量查询？ | | 需要索引优化 |
| 是否有并发写入？ | | 需要锁机制 |
| 是否有定时任务？ | | 评估对系统的影响 |
| 是否有文件上传下载？ | | 评估带宽和存储 |

### 安全风险
| 检查项 | 是/否 | 说明 |
|--------|-------|------|
| 是否涉及敏感数据？ | | 加密存储+脱敏展示 |
| 是否有外部接口调用？ | | 接口签名+限流 |
| 是否有文件上传？ | | 类型+大小校验 |
| 是否有批量操作？ | | 权限校验+操作日志 |

### 进度风险
| 检查项 | 是/否 | 说明 |
|--------|-------|------|
| 需求是否明确？ | | 模糊需求需要提前澄清 |
| 是否依赖外部系统？ | | 外部依赖可能延期 |
| 是否有技术难点？ | | 需要预留攻关时间 |
| 人员是否充足？ | | 评估工作量分配 |
```

---

## 五、问题清单模板（技术可行性疑问）

```markdown
# 技术问题清单：{项目/模块名称}

| # | 问题 | 问题来源 | 优先级 | 可能方案A | 可能方案B | 推荐方案 | 理由 | 需要确认人 | 状态 |
|---|------|----------|--------|----------|----------|----------|------|-----------|------|
| 1 | {技术问题} | {来源} | {级别} | {方案A} | {方案B} | {推荐} | {理由} | {人员} | 待确认 |

**问题来源分类：**
| 来源 | 说明 | 示例 |
|------|------|------|
| PRD需求 | 需求描述的技术实现不确定 | "实时推送"用WebSocket还是SSE？ |
| 架构设计 | 架构方案有多个选择 | 新建独立服务还是模块化？ |
| 性能要求 | 性能目标可能难以达成 | 百万级数据查询优化方案？ |
| 安全合规 | 安全要求需要额外技术投入 | 数据加密方案选择？ |
| 第三方集成 | 外部系统对接的技术细节 | 对方接口的认证方式？ |

**问题优先级：**
| 级别 | 含义 | 处理时限 |
|------|------|----------|
| 🔴 阻塞 | 不确认无法开始开发 | 架构设计阶段必须确认 |
| 🟡 重要 | 影响核心功能实现 | 开发前确认 |
| 🟢 建议 | 优化或替代方案选择 | 开发过程中确认 |
```

---

## 六、设计文档检查清单

完成概要设计后，逐项检查：

- [ ] 设计目标是否明确且可衡量？
- [ ] 架构图是否清晰描述了各层关系？
- [ ] 技术选型是否有评估依据？
- [ ] 模块划分是否合理，职责是否清晰？
- [ ] 数据库表设计是否完整？
- [ ] API接口是否全部定义？
- [ ] 权限方案是否覆盖所有操作？
- [ ] 性能指标是否明确？
- [ ] 缓存策略是否合理？
- [ ] 安全措施是否充分？
- [ ] 风险是否识别并有缓解措施？
- [ ] 是否符合RuoYi框架规范？
- [ ] 多租户方案是否考虑？
- [ ] 问题清单是否整理？
