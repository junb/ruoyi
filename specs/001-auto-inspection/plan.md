# Implementation Plan: 服务器自动巡检

**Branch**: `001-auto-inspection` | **Date**: 2026-04-05 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-auto-inspection/spec.md`

## Summary

通过 Prometheus HTTP API 采集服务器资源指标（CPU、内存、磁盘），将指标格式化为结构化文本后发送给阿里云百炼智能体生成 Markdown 巡检报告。支持手动触发和定时触发两种方式，通过 SSE 实时推送状态变更，提供报告的查询、详情查看、下载和批量删除功能。

## Technical Context

**Language/Version**: Java 17（Spring Boot 3.5.12）+ TypeScript ~5.9.3（Vue 3.5.30）
**Primary Dependencies**: MyBatis-Plus 3.5.16, Sa-Token 1.44.0, Redisson 3.52.0, SnailJob 1.9.0, Hutool HTTP, Element Plus 2.13.5
**Storage**: MySQL（`ry-vue`）+ Redis（Spring Cache/Redisson）
**Testing**: JUnit 5 + Spring Boot Test（项目默认 skipTests=true）
**Target Platform**: Web 应用（后端 Undertow 8080，前端 Vite 80）
**Project Type**: 企业管理系统功能模块（前后端分离）
**Performance Goals**: 手动触发 2 秒内响应，报告生成 60 秒内完成（≤50 台服务器）
**Constraints**: 异步执行，SSE 实时推送，RFC 5987 中文文件名，多租户支持
**Scale/Scope**: 约 50 台服务器，月度巡检，报告保留 12 个月

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 原则 | 合规 | 说明 |
|------|------|------|
| I. 分层架构 | ✅ | Controller → Service → Mapper 三层分离，新模块独立包结构 |
| II. 统一响应与异常处理 | ✅ | 所有接口返回 `R<T>`，分页返回 `TableDataInfo<T>`，异常通过 ServiceException 抛出 |
| III. 多租户数据隔离 | ✅ | 两张业务表均含 `tenant_id`，实体继承 `TenantEntity` |
| IV. 注解驱动开发 | ✅ | 使用 `@AutoMapper` 做 BO/VO/Entity 转换，`@SaCheckPermission` 权限控制 |
| V. 安全与权限 | ✅ | 四个权限点：query/create/download/delete，Controller 方法标注 `@SaCheckPermission` |
| VI. 缓存策略 | ⚠️ | 本功能以写操作为主，暂不引入缓存。字典配置通过 Redis 缓存的 DictService 读取 |
| VII. 前后端协作规范 | ✅ | 前端 API 模块与 Controller 一一对应，类型定义同步 |

**设计后复查**：设计完成后重新检查上述原则，确认无违反。

## Project Structure

### Documentation (this feature)

```text
specs/001-auto-inspection/
├── plan.md              # 本文件
├── research.md          # Phase 0 研究产出
├── data-model.md        # Phase 1 数据模型
├── quickstart.md        # Phase 1 快速开始
├── contracts/
│   └── api.md           # Phase 1 接口契约
└── tasks.md             # Phase 2 任务列表（/speckit.tasks 生成）
```

### Source Code (repository root)

```text
RuoYi-Vue-Plus/
├── ruoyi-modules/
│   └── ruoyi-inspection/                    # 新增巡检模块
│       ├── pom.xml
│       └── src/main/java/org/dromara/inspection/
│           ├── controller/
│           │   └── InspectionReportController.java
│           ├── domain/
│           │   ├── InspectionReport.java              # 报告实体（继承 TenantEntity）
│           │   ├── InspectionInstanceMetric.java       # 实例指标实体（继承 TenantEntity）
│           │   ├── bo/
│           │   │   ├── InspectionReportBo.java
│           │   │   └── InspectionReportRetryBo.java
│           │   └── vo/
│           │       ├── InspectionReportVo.java
│           │       ├── InspectionReportDetailVo.java
│           │       └── InspectionInstanceMetricVo.java
│           ├── mapper/
│           │   ├── InspectionReportMapper.java
│           │   └── InspectionInstanceMetricMapper.java
│           ├── service/
│           │   ├── IInspectionReportService.java
│           │   └── impl/
│           │       ├── InspectionReportServiceImpl.java
│           │       ├── PrometheusCollectorService.java  # Prometheus 指标采集
│           │       └── BailianReportService.java         # 百炼 AI 报告生成
│           └── job/
│               └── InspectionScheduledJob.java           # SnailJob 定时任务
├── ruoyi-admin/src/main/resources/
│   └── mapper/inspection/                               # MyBatis XML
│       ├── InspectionReportMapper.xml
│       └── InspectionInstanceMetricMapper.xml

plus-ui/src/
├── api/inspection/report/
│   ├── index.ts                        # API 接口函数
│   └── types.ts                        # TypeScript 类型定义
└── views/inspection/report/
    ├── index.vue                       # 报告列表页
    ├── detail.vue                      # 报告详情页
    └── components/
        └── ReportContent.vue           # Markdown 内容渲染组件
```

**Structure Decision**: 新建独立模块 `ruoyi-inspection` 遵循项目现有的模块化架构（与 `ruoyi-system`、`ruoyi-demo` 平级），前端新建 `inspection` 目录与后端模块对应。需要在 `ruoyi-admin/pom.xml` 中添加模块依赖。

## Design Decisions

### D1: 异步执行方案

使用 `@Async` + Spring Boot 3.5 内置线程池。手动触发和重试时，Controller 立即返回，Service 层异步执行采集和生成逻辑。状态变更后通过 SSE 推送通知前端。

### D2: Prometheus 指标采集

使用 Hutool `HttpRequest` 调用 Prometheus HTTP API（`/api/v1/query`），按实例 IP 聚合 CPU、内存、磁盘指标。衍生计算（使用率）在 Service 层完成，不在 SQL 层。

### D3: 百炼 AI 报告生成

将指标格式化为 Markdown 文本作为 Prompt，通过 Hutool HTTP POST 调用百炼智能体应用。失败时指数退避重试 3 次（2s → 4s → 8s）。API Key 和 App ID 从系统字典读取。

### D4: SSE 实时推送

复用 `ruoyi-common-sse` 模块的 `SseMessageUtils.publishMessage()`，向触发用户推送报告状态变更通知。

### D5: 定时任务

使用 SnailJob 的 `@JobExecutor(name = "inspectionScheduledJob")` 注解定义定时任务。生成成功后自动清理超过保留月数的历史报告。

### D6: 文件下载

复用 `FileUtils.setAttachmentResponseHeader()` 设置 RFC 5987 编码的中文文件名，直接写入 response 流。

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| VI. 缓存策略（未引入 @Cacheable/@CacheEvict） | 巡检功能以写操作为主（生成报告），读操作（列表查询）数据量小且实时性要求高，缓存命中率低 | 缓存层会增加代码复杂度且收益不明显；字典配置已通过 DictService 自带 Redis 缓存覆盖 |
