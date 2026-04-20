<!--
  Sync Impact Report:
  - Version change: N/A → 1.0.0 (initial creation)
  - Modified principles: N/A (first version)
  - Added sections: All sections (Core Principles, 技术栈约束, 开发流程, Governance)
  - Removed sections: N/A
  - Templates status:
    - .specify/templates/plan-template.md ✅ 兼容（Constitution Check 节点已预留）
    - .specify/templates/spec-template.md ✅ 兼容（无冲突）
    - .specify/templates/tasks-template.md ✅ 兼容（任务结构与原则一致）
  - Follow-up TODOs: 无
-->

# RuoYi-Vue-Plus Constitution

## Core Principles

### I. 分层架构

系统必须严格遵循 Controller → Service → Mapper 三层架构，层间职责不得交叉。

- Controller 层：仅负责请求路由、参数校验（`@Validated`）、权限控制（`@SaCheckPermission`）、操作日志（`@Log`）。不允许包含业务逻辑。
- Service 层：接口（`I{Module}Service`）与实现（`{Module}ServiceImpl`）必须分离。事务（`@Transactional`）、缓存（`@Cacheable`/`@CacheEvict`）、业务逻辑均在此层。
- Mapper 层：继承 `BaseMapperPlus<Entity, Vo>`，数据权限通过 `@DataPermission` + `@DataColumn` 注解在 Mapper 接口上声明。

**原因**：分层隔离确保职责单一，便于代码生成器自动生成标准模板代码，降低维护成本。

### II. 统一响应与异常处理

所有 API 必须返回 `R<T>` 标准包装类；分页查询必须返回 `TableDataInfo<T>`。

- 业务异常必须通过 `ServiceException` 抛出，由全局异常处理器统一捕获并转换为 `R<T>` 响应。
- 前端 HTTP 请求通过 `src/utils/request.ts` 统一封装，响应拦截器自动处理错误码（401 跳转登录、500 提示错误等）。
- 禁止在 Controller 中使用 `try-catch` 处理业务异常。

**原因**：统一的响应格式和异常处理机制保证前后端交互的一致性，减少重复代码。

### III. 多租户数据隔离

默认开启多租户（`tenant.enable: true`），新增业务表必须包含 `tenant_id` 字段。

- 非租户表必须在 `application.yml` 的 `tenant.excludes` 中显式声明。
- 实体类继承体系：`BaseEntity` → `TenantEntity`，新增业务实体必须继承 `TenantEntity`。
- **建表字段对齐**：继承 `TenantEntity` 的实体对应的数据表必须包含其全部字段：`tenant_id`、`create_dept`、`create_by`、`create_time`、`update_by`、`update_time`、`del_flag`（含 `@TableLogic` 逻辑删除）。遗漏任何字段将导致 MyBatis-Plus 生成 SQL 时报 `Unknown column` 错误。
- 租户隔离由 MyBatis-Plus 租户插件自动处理，禁止在业务代码中手动拼接租户条件。

**原因**：多租户是本项目的核心架构特性，数据隔离是安全性底线。

### IV. 注解驱动开发

利用框架提供的注解减少模板代码，优先使用注解而非编码实现横切关注点。

- 对象转换：BO/VO/Entity 之间使用 `@AutoMapper`（MapStruct-Plus），禁止手动编写转换代码。
- 数据脱敏：敏感字段使用 `@Sensitive` 注解，如手机号、身份证号。
- 数据翻译：ID 到名称的映射使用 `@Translation` 注解，如部门ID→部门名称。
- 参数校验：使用 Jakarta Validation 注解（`@NotBlank`、`@Size`、`@Email`）+ 自定义 `@Xss` 注解。
- 主键策略：统一使用雪花ID（`ASSIGN_ID`），禁止使用数据库自增。

**原因**：注解驱动减少重复代码，保持代码风格一致，便于代码生成器生成标准代码。

### V. 安全与权限

所有接口必须有明确的权限控制，遵循最小权限原则。

- 接口权限：Controller 方法必须标注 `@SaCheckPermission` 注解。
- 数据权限：通过 Mapper 上的 `@DataPermission` 注解声明，支持部门、个人等多种粒度。
- API 加密：敏感接口使用 `@ApiEncrypt` 注解，前后端 RSA 密钥必须配套。
- 防重复提交：写操作接口使用 `@RepeatSubmit` 注解。

**原因**：企业级系统对安全性要求严格，权限控制必须在代码层面显式声明。

### VI. 缓存策略

缓存统一使用 Spring-Cache + Redisson 后端，通过注解声明。

- 查询方法使用 `@Cacheable`，缓存 key 使用 `CacheNames` 常量类中的预定义值。
- 增删改方法必须使用 `@CacheEvict` 清除关联缓存。
- 缓存注解支持 TTL、maxIdle、maxSize 扩展参数。

**原因**：统一的缓存策略避免缓存穿透、雪崩等问题，注解方式降低使用门槛。

### VII. 前后端协作规范

前端 API 模块与后端 Controller 必须一一对应，数据结构保持同步。

- 前端 API 模块结构：每个功能模块独立目录，包含 `index.ts`（接口函数）和 `types.ts`（类型定义）。
- 前端类型定义：请求类型（`XxxQuery`、`XxxForm`）和响应类型（`XxxVO`）与后端 BO/VO 对应。
- 前端使用 `<script setup lang="ts">` 组合式 API 风格，组件名称通过 `name` 属性声明。
- HTTP 请求统一使用 `AxiosPromise<T>` 返回类型。

**原因**：前后端一一对应降低沟通成本，类型同步减少运行时错误。

## 技术栈约束

### 后端（强制）

- **语言**：Java 17（兼容 JDK 21）
- **框架**：Spring Boot 3.5.12
- **ORM**：MyBatis-Plus 3.5.16，主键策略 `ASSIGN_ID`（雪花ID）
- **认证**：Sa-Token 1.44.0 + JWT
- **缓存**：Redisson 3.52.0 + Spring-Cache
- **数据库**：MySQL（默认），支持 Oracle/PostgreSQL/SQLServer
- **对象映射**：MapStruct-Plus 1.5.0
- **序列化**：Jackson（Spring Boot 内置）
- **代码规范**：阿里巴巴 Java 编码规范，Lombok 减少模板代码
- **基础包名**：`org.dromara`

### 前端（强制）

- **语言**：TypeScript ~5.9.3
- **框架**：Vue 3.5.30 + Composition API
- **UI 库**：Element Plus 2.13.5（自动导入）
- **构建**：Vite 7.3.1
- **状态管理**：Pinia 3.0.4
- **CSS**：UnoCSS（attributify 模式）
- **表格**：VxeTable 4.18.1
- **Node**：>= 20.19.0
- **路径别名**：`@/` 指向 `src/`

### 禁止事项

- 后端禁止使用 `fastjson`，统一使用 Jackson。
- 后端禁止在 Controller 中编写业务逻辑。
- 前端禁止手动注册 Element Plus 组件（已配置自动导入）。
- 前后端禁止硬编码 RSA 密钥（使用环境变量或配置文件）。

## 开发流程

### 新增业务模块标准流程

1. **数据库**：建表时必须包含 `tenant_id`、`create_by`、`create_time`、`update_by`、`update_time`、`del_flag` 等审计字段。
2. **后端代码生成**：使用代码生成器（`ruoyi-generator`）生成 Entity/Mapper/Service/Controller/BO/VO 模板代码。
3. **前端代码生成**：代码生成器同步生成前端 API 模块和页面组件。
4. **权限配置**：在菜单管理中配置功能权限，权限标识与 `@SaCheckPermission` 注解值对应。

### 代码提交规范

- 提交信息使用中文，格式：`模块名: 简要描述`。
- 示例：`系统管理: 新增用户导出功能`、`代码生成: 修复模板空指针问题`。

### 构建环境（强制）

- **JDK**：Azul Zulu 17.0.15（路径：`/Users/jun/Library/Java/JavaVirtualMachines/azul-17.0.15/Contents/Home`），禁止使用 JDK 21 编译。
- **Maven**：Apache Maven 3.9.14（路径：`/Users/jun/Documents/tools/maven/apache-maven-3.9.14`）。
- **打包方式**：必须通过终端使用上述 Maven CLI 执行 `mvn clean package -P dev -pl ruoyi-admin -am -q`，禁止依赖 IntelliJ IDEA 内部 Maven 构建生产 JAR。
- **启动方式**：必须使用 `java -jar ruoyi-admin/target/ruoyi-admin.jar` 启动，禁止通过 IDE 直接运行或使用 `mvn spring-boot:run`。IDE 会持续污染 `target` 目录导致 class 文件损坏。
- **原因**：IntelliJ 的 Eclipse JDT 编译器会生成损坏的 class 文件（`super_class` 被解析为 `Object` 而非实际父类），导致运行时 `ClassNotFoundException` 和方法找不到错误。即使打包后，IDE 的自动构建也会覆盖 target 中的 class 文件。

### 环境与配置

- Maven Profile：`local`（本地）、`dev`（开发，默认）、`prod`（生产）。
- 后端端口 8080，前端开发端口 80，代理 `/dev-api` → `localhost:8080`。
- 默认数据库名 `ry-vue`，MySQL root/root，Redis 密码 `ruoyi123`。

## Governance

本 constitution 是项目开发的最高准则，所有代码变更必须遵守上述原则。

- **修订流程**：原则变更必须附带说明文档、影响分析和迁移方案，经项目负责人批准后生效。
- **版本策略**：遵循语义版本（MAJOR.MINOR.PATCH）— 原则删除或重定义为 MAJOR，新增原则为 MINOR，措辞优化为 PATCH。
- **合规检查**：所有代码审查必须验证是否符合本文件规定的原则。使用 `CLAUDE.md` 作为运行时开发指导文件。
- **复杂度豁免**：如需违反某项原则，必须在实现计划的 Complexity Tracking 表中记录：违反项、必要性、被否决的更简方案及理由。

**Version**: 1.2.0 | **Ratified**: 2026-04-05 | **Last Amended**: 2026-04-19
