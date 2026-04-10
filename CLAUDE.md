# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

RuoYi-Vue-Plus v5.6.0 — Dromara 组织出品的多租户企业级管理系统。基于原版 RuoYi 框架全面重写，面向分布式集群与多租户场景。仓库包含两个顶层目录：

- `RuoYi-Vue-Plus/` — Java 后端（Spring Boot 3.5.12，JDK 17/21）
- `plus-ui/` — Vue 3 + TypeScript 前端（Vite 7，Element Plus，Pinia）

官方文档：https://plus-doc.dromara.org

## 构建与运行

### 后端

```bash
# 打包（默认 dev 环境）
cd RuoYi-Vue-Plus
mvn clean package -P dev

# 打包指定模块及其依赖
mvn clean package -pl ruoyi-admin -am

# 运行测试（默认跳过，skipTests=true）
mvn test -DskipTests=false

# 运行单个测试类
mvn test -DskipTests=false -pl ruoyi-modules/ruoyi-system -Dtest=SomeTestClass

# 启动应用（入口类：org.dromara.DromaraApplication）
mvn spring-boot:run -pl ruoyi-admin -P dev
```

环境 Profile：`local`、`dev`（默认）、`prod`。通过 Maven 资源过滤注入到 `application.yml`。

后端端口 **8080**，依赖 MySQL（数据库名 `ry-vue`）和 Redis。

### 前端

```bash
cd plus-ui
npm install --registry=https://registry.npmmirror.com
npm run dev              # 开发服务器端口 80，/dev-api 代理到 localhost:8080
npm run build:prod       # 生产构建
npm run build:dev        # 开发环境构建
npm run lint:eslint      # ESLint 检查
npm run lint:eslint:fix  # ESLint 检查并修复
```

要求 Node >= 20.19.0。

### 数据库初始化

SQL 脚本位于 `RuoYi-Vue-Plus/script/sql/`，默认 MySQL，同时提供 Oracle、PostgreSQL、SQLServer 脚本。

- 主库：`script/sql/ry_vue_5.X.sql`
- 定时任务：`script/sql/ry_job.sql`
- 工作流：`script/sql/ry_workflow.sql`

版本升级 SQL 在 `script/sql/update/` 目录。

### Docker 部署

`RuoYi-Vue-Plus/script/docker/docker-compose.yml` 包含 MySQL、Redis、Nginx 配置。

## 后端架构

### 模块结构

```
RuoYi-Vue-Plus/
├── ruoyi-admin/           # 应用入口（DromaraApplication）、配置文件
├── ruoyi-common/          # 通用库（每个为独立 Maven 模块）
│   ├── ruoyi-common-core/       # 核心工具类、基类、注解
│   ├── ruoyi-common-web/        # Web 层（控制器、响应包装）
│   ├── ruoyi-common-satoken/    # Sa-Token 鉴权集成
│   ├── ruoyi-common-security/   # 安全过滤器、数据权限
│   ├── ruoyi-common-tenant/     # 多租户支持
│   ├── ruoyi-common-mybatis/    # MyBatis-Plus 配置、基础 Mapper
│   ├── ruoyi-common-redis/      # Redis/Redisson 集成、Spring-Cache 扩展
│   ├── ruoyi-common-log/        # 操作日志
│   ├── ruoyi-common-oss/        # 对象存储（S3 协议）
│   ├── ruoyi-common-excel/      # Excel 导入导出
│   ├── ruoyi-common-encrypt/    # 接口加解密（AES + RSA）
│   ├── ruoyi-common-idempotent/ # 分布式幂等
│   ├── ruoyi-common-ratelimiter/# 限流
│   ├── ruoyi-common-sensitive/  # 数据脱敏
│   ├── ruoyi-common-translation/# 数据翻译（字典、用户ID→名称等）
│   ├── ruoyi-common-doc/        # SpringDoc OpenAPI 接口文档
│   ├── ruoyi-common-json/       # Jackson 序列化配置
│   ├── ruoyi-common-job/        # SnailJob 分布式任务调度
│   ├── ruoyi-common-mail/       # 邮件发送
│   ├── ruoyi-common-sms/        # 短信（sms4j）
│   ├── ruoyi-common-social/     # 第三方 OAuth 登录（JustAuth）
│   ├── ruoyi-common-sse/        # SSE 推送
│   └── ruoyi-common-websocket/  # WebSocket
├── ruoyi-modules/         # 业务模块
│   ├── ruoyi-system/      # 系统管理（用户、角色、菜单、部门、租户）
│   ├── ruoyi-generator/   # 代码生成器
│   ├── ruoyi-job/         # 定时任务
│   ├── ruoyi-demo/        # 演示示例
│   └── ruoyi-workflow/    # 工作流引擎（Warm-Flow）
├── ruoyi-extend/          # 独立扩展服务
│   ├── ruoyi-monitor-admin/    # Spring Boot Admin 监控中心
│   └── ruoyi-snailjob-server/  # SnailJob 调度服务器
└── script/                # SQL 脚本、Docker、部署脚本
```

### 核心设计模式

- **基础包名**：`org.dromara` — 所有 Mapper 扫描、实体扫描、组件扫描均基于此根包。
- **主键策略**：MyBatis-Plus 雪花ID（`ASSIGN_ID`），非自增。
- **认证鉴权**：Sa-Token + JWT，请求头 `Authorization`，配置在 `application.yml` 的 `sa-token` 节点。
- **多租户**：默认开启（`tenant.enable: true`），通过 MyBatis-Plus 租户插件实现数据隔离，排除表在配置中列出。
- **数据权限**：MyBatis-Plus 插件拦截查询自动拼接权限 SQL，通过 Mapper 接口上的注解配置。
- **统一响应**：所有接口返回 `R<T>` 标准包装类。
- **对象转换**：MapStruct-Plus 实现 DTO/VO/Entity 之间的转换。
- **缓存**：Spring-Cache + Redisson 后端，注解扩展支持 TTL/maxIdle/maxSize。
- **数据翻译**：`@Translation` 注解在 Jackson 序列化时自动将 ID 翻译为显示值。
- **数据脱敏**：`@Sensitive` 注解在序列化时自动脱敏（手机号、身份证等）。

### 配置文件

- `ruoyi-admin/src/main/resources/application.yml` — 主配置（服务器、Sa-Token、MyBatis-Plus、租户等）
- `ruoyi-admin/src/main/resources/application-dev.yml` — 开发环境（数据源、Redis、邮件、短信、OAuth）
- `ruoyi-admin/src/main/resources/application-prod.yml` — 生产环境

## 前端架构

```
plus-ui/src/
├── api/           # API 请求模块（与后端 Controller 一一对应）
├── views/         # 页面组件（按功能模块组织）
├── components/    # 公共组件
├── layout/        # 布局框架
├── router/        # 路由配置
├── store/         # Pinia 状态管理
├── utils/         # 工具函数（请求封装、鉴权等）
├── hooks/         # 组合式函数
├── directive/     # 自定义指令
├── lang/          # 国际化（中文/英文）
├── plugins/       # 插件注册
├── enums/         # TypeScript 枚举
└── types/         # TypeScript 类型定义
```

### 核心设计模式

- **API 代理**：开发模式通过 Vite 将 `/dev-api` 代理到 `localhost:8080`，配置在 `vite.config.ts`。
- **认证令牌**：通过 Cookie 存储，请求头 `Authorization` 传递（与后端 Sa-Token 对应）。
- **接口加密**：通过 `VITE_APP_ENCRYPT` 环境变量控制 RSA + AES 加密，前后端 RSA 密钥必须配套。
- **自动导入**：通过 `unplugin-auto-import` 和 `unplugin-vue-components` 实现组件和 API 自动导入。
- **UI 框架**：Element Plus 自动导入，无需手动注册组件。
- **CSS 方案**：UnoCSS attributify 模式。
- **表格组件**：VxeTable 处理复杂数据表格。

## 开发约定

- Java 代码遵循阿里巴巴编码规范，项目大量使用 Lombok。
- 后端 API 基础路径为 `/`（context-path），Controller 位于 `org.dromara.{module}.controller` 包下。
- 前端 API 模块与后端 Controller 一一对应。
- 多租户表使用 `tenant_id` 字段，非租户表在 `tenant.excludes` 配置中列出。
- 默认数据库名 `ry-vue`，MySQL 默认账号 root/root，Redis 默认密码 `ruoyi123`。
- 版本升级 SQL 在 `script/sql/update/` 目录下。

## Active Technologies
- Java 17（Spring Boot 3.5.12）+ TypeScript ~5.9.3（Vue 3.5.30） + MyBatis-Plus 3.5.16, Sa-Token 1.44.0, Redisson 3.52.0, SnailJob 1.9.0, Hutool HTTP, Element Plus 2.13.5 (001-auto-inspection)
- MySQL（`ry-vue`）+ Redis（Spring Cache/Redisson） (001-auto-inspection)
- Java 17（Spring Boot 3.5.12） + MyBatis-Plus 3.5.16, Sa-Token 1.44.0, Hutool HTTP, Jackson, Spring Async (002-alert-feishu-forward)
- MySQL（`ry-vue` 库），Redis（Spring Cache/Redisson） (002-alert-feishu-forward)

## Recent Changes
- 001-auto-inspection: Added Java 17（Spring Boot 3.5.12）+ TypeScript ~5.9.3（Vue 3.5.30） + MyBatis-Plus 3.5.16, Sa-Token 1.44.0, Redisson 3.52.0, SnailJob 1.9.0, Hutool HTTP, Element Plus 2.13.5
