# Tasks: 服务器自动巡检

**Input**: Design documents from `/specs/001-auto-inspection/`
**Prerequisites**: plan.md (required), spec.md (required), data-model.md, contracts/api.md, research.md

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Backend module**: `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/`
- **Backend base package**: `org.dromara.inspection`
- **Mapper XML**: `RuoYi-Vue-Plus/ruoyi-admin/src/main/resources/mapper/inspection/`
- **Frontend API**: `plus-ui/src/api/inspection/report/`
- **Frontend views**: `plus-ui/src/views/inspection/report/`

---

## Phase 1: Setup (项目初始化)

**Purpose**: 创建后端模块骨架和前端目录结构

- [x] T001 创建 Maven 模块 `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/pom.xml`，继承 ruoyi-modules 父 POM，依赖 ruoyi-common-core、ruoyi-common-web、ruoyi-common-mybatis、ruoyi-common-sse、ruoyi-common-job、ruoyi-common-log
- [x] T002 [P] 在 `RuoYi-Vue-Plus/ruoyi-admin/pom.xml` 中添加 ruoyi-inspection 模块依赖
- [x] T003 [P] 创建建表 SQL 脚本 `RuoYi-Vue-Plus/script/sql/inspection.sql`，包含 prometheus_inspection_report 表和 prometheus_inspection_instance_metric 表（含审计字段和索引）
- [x] T004 [P] 创建字典初始化 SQL，向 sys_dict_type 和 sys_dict_data 插入 `prometheus_inspection` 字典类型及其配置项（PROMETHEUS_ENDPOINT、BAILIAN_API_KEY、BAILIAN_APP_ID、RETENTION_MONTHS）
- [x] T005 [P] 创建菜单和权限 SQL，向 sys_menu 插入一级菜单"巡检管理"和二级菜单"巡检报告"，以及四个按钮权限（query/create/download/delete）

---

## Phase 2: Foundational (基础设施 — 阻塞所有用户故事)

**Purpose**: 创建所有用户故事共用的实体、Mapper 和基础服务

**⚠️ CRITICAL**: 所有用户故事的工作必须在此阶段完成后才能开始

- [x] T006 创建报告实体 `InspectionReport.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/domain/`，继承 TenantEntity，使用 @AutoMapper 注解，包含所有业务字段（reportName、reportDate、status、instanceCount、generationDuration、markdownContent、prometheusEndpoint、triggerType、errorMessage）
- [x] T007 [P] 创建实例指标实体 `InspectionInstanceMetric.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/domain/`，继承 TenantEntity，使用 @AutoMapper 注解，diskPartitions 字段使用 String 类型存储 JSON
- [x] T008 [P] 创建报告 BO 类 `InspectionReportBo.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/domain/bo/`，包含分页查询参数（status、reportDate、beginTime、endTime）
- [x] T009 [P] 创建重试 BO 类 `InspectionReportRetryBo.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/domain/bo/`，包含 reportId 字段并标注 @NotNull
- [x] T010 [P] 创建报告列表 VO `InspectionReportVo.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/domain/vo/`
- [x] T011 [P] 创建报告详情 VO `InspectionReportDetailVo.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/domain/vo/`，额外包含 markdownContent 和 errorMessage
- [x] T012 [P] 创建实例指标 VO `InspectionInstanceMetricVo.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/domain/vo/`
- [x] T013 创建 Mapper 接口 `InspectionReportMapper.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/mapper/`，继承 BaseMapperPlus<InspectionReport, InspectionReportVo>
- [x] T014 [P] 创建 Mapper 接口 `InspectionInstanceMetricMapper.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/mapper/`，继承 BaseMapperPlus<InspectionInstanceMetric, InspectionInstanceMetricVo>
- [x] T015 创建 MyBatis XML `InspectionReportMapper.xml` 在 `RuoYi-Vue-Plus/ruoyi-admin/src/main/resources/mapper/inspection/`，包含分页查询 SQL（支持 status、日期范围筛选，按 createTime 降序）
- [x] T016 [P] 创建 MyBatis XML `InspectionInstanceMetricMapper.xml` 在 `RuoYi-Vue-Plus/ruoyi-admin/src/main/resources/mapper/inspection/`，包含按 reportId 查询实例指标列表的 SQL

**Checkpoint**: 基础设施就绪 — 用户故事实现可以并行开始

---

## Phase 3: User Story 1 - 手动触发巡检并查看报告 (Priority: P1) 🎯 MVP

**Goal**: 用户点击"立即巡检"按钮，系统异步采集 Prometheus 指标，调用百炼 AI 生成 Markdown 报告，通过 SSE 推送状态通知

**Independent Test**: 手动触发一次巡检，等待报告从 pending → success，查看报告详情和实例指标

### Implementation for User Story 1

- [x] T017 [US1] 创建 Prometheus 采集服务 `PrometheusCollectorService.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/service/impl/`，使用 Hutool HttpRequest 调用 Prometheus /api/v1/query 接口，采集 CPU 使用率、内存总量/可用量、磁盘各分区总量/可用量，按 instance IP 聚合，衍生计算内存使用率和磁盘使用率
- [x] T018 [US1] 创建百炼 AI 报告服务 `BailianReportService.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/service/impl/`，将指标格式化为 Markdown 文本作为 Prompt，通过 Hutool HTTP POST 调用百炼智能体应用，失败时指数退避重试 3 次（2s→4s→8s），API Key 和 App ID 从 DictService 读取（字典类型 prometheus_inspection）
- [x] T019 [US1] 创建服务接口 `IInspectionReportService.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/service/`，定义方法：generateReport（异步生成）、selectPageList、selectDetailById、selectInstancesByReportId、deleteByIds、retryReport、downloadReport
- [x] T020 [US1] 创建服务实现 `InspectionReportServiceImpl.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/service/impl/`，实现 generateReport 方法：创建 pending 状态报告 → @Async 异步执行 → 调用 PrometheusCollectorService 采集指标 → 保存实例指标 → 调用 BailianReportService 生成报告 → 更新状态为 success/failed → 通过 SseMessageUtils.publishMessage 推送状态变更通知
- [x] T021 [US1] 创建 Controller `InspectionReportController.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/controller/`，实现 POST /inspection/report/generate（@SaCheckPermission inspection:report:create，@Log，@RepeatSubmit）、GET /list（inspection:report:query）、GET /detail/{id}（inspection:report:query）、GET /instances/{id}（inspection:report:query）

**Checkpoint**: 此时手动触发巡检完整可用 — 触发 → 采集 → AI 生成 → SSE 通知 → 查看报告

---

## Phase 4: User Story 2 - 报告管理与下载 (Priority: P2)

**Goal**: 用户可以按状态筛选报告列表、查看报告详情、下载 Markdown 文件、批量删除报告

**Independent Test**: 通过已有报告数据验证列表查询（状态筛选）、详情查看、下载（中文文件名）、批量删除

### Implementation for User Story 2

- [x] T022 [US2] 在 `InspectionReportController.java` 中实现 DELETE /inspection/report/delete 接口（@SaCheckPermission inspection:report:delete，@Log），接收 ids 数组，Service 层级联删除关联的 InspectionInstanceMetric 数据
- [x] T023 [US2] 在 `InspectionReportController.java` 中实现 GET /inspection/report/download/{id} 接口（@SaCheckPermission inspection:report:download），使用 FileUtils.setAttachmentResponseHeader 设置 RFC 5987 中文文件名，将 markdownContent 写入 response 输出流
- [x] T024 [P] [US2] 创建前端 TypeScript 类型定义 `plus-ui/src/api/inspection/report/types.ts`，定义 InspectionReportQuery、InspectionReportVo、InspectionReportDetailVo、InspectionInstanceMetricVo 等接口类型
- [x] T025 [P] [US2] 创建前端 API 模块 `plus-ui/src/api/inspection/report/index.ts`，实现 listReport、detailReport、generateReport、downloadReport、deleteReport、listInstances、retryReport 七个接口函数
- [x] T026 [US2] 创建前端报告列表页 `plus-ui/src/views/inspection/report/index.vue`，实现：顶部操作栏（立即巡检按钮 + 状态筛选下拉框）、VxeTable 表格（报告名称、日期、状态标签、实例数量、耗时、触发类型、创建时间、操作列）、操作按钮（查看详情、下载、删除、重试）、SSE 监听自动刷新列表状态
- [x] T027 [P] [US2] 创建前端 Markdown 渲染组件 `plus-ui/src/views/inspection/report/components/ReportContent.vue`，使用 v-html 渲染 Markdown 内容（可使用 marked.js 或类似库）
- [x] T028 [US2] 创建前端报告详情页 `plus-ui/src/views/inspection/report/detail.vue`，展示报告元信息（名称、日期、状态、耗时、触发类型、Prometheus 地址）+ 实例指标表格（IP、CPU 使用率、内存使用率、磁盘分区）+ Markdown 报告内容渲染组件

**Checkpoint**: 报告管理和下载功能完整可用 — 列表筛选、详情查看、下载、删除

---

## Phase 5: User Story 3 - 失败报告重试 (Priority: P2)

**Goal**: 用户对失败报告点击"重试"，系统重新执行采集和生成流程

**Independent Test**: 对一条失败报告执行重试操作，验证状态回退到 pending 和重新生成

### Implementation for User Story 3

- [x] T029 [US3] 在 `InspectionReportServiceImpl.java` 中实现 retryReport 方法：校验报告状态为 failed → 重置状态为 pending → 清除旧的 error_message 和实例指标数据 → @Async 异步重新执行采集和生成流程 → SSE 推送状态变更
- [x] T030 [US3] 在 `InspectionReportController.java` 中实现 POST /inspection/report/retry 接口（@SaCheckPermission inspection:report:create，@Log），接收 reportId 参数调用 retryReport

**Checkpoint**: 重试功能完整可用 — 失败报告可重试，状态正确流转

---

## Phase 6: User Story 4 - 定时自动巡检 (Priority: P3)

**Goal**: 系统按 cron 表达式自动执行巡检，成功后清理过期报告

**Independent Test**: 配置短间隔定时规则，等待自动触发后验证报告生成和过期清理

### Implementation for User Story 4

- [x] T031 [US4] 创建 SnailJob 定时任务 `InspectionScheduledJob.java` 在 `ruoyi-inspection/src/main/java/org/dromara/inspection/job/`，使用 @Component + @JobExecutor(name = "inspectionScheduledJob") 注解，调用 IInspectionReportService.generateReport（triggerType = scheduled），生成成功后查询并删除超过 RETENTION_MONTHS 的历史报告及关联指标
- [x] T032 [US4] 在 `InspectionReportServiceImpl.java` 中实现 cleanExpiredReports 方法，查询 create_time 早于 RETENTION_MONTHS 个月前的报告，级联删除报告和关联指标

**Checkpoint**: 定时巡检和过期清理功能完整可用

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: 最终完善和配置验证

- [x] T033 [P] 执行建表 SQL（T003）和字典 SQL（T004），验证表结构和索引创建正确
- [ ] T034 [P] 执行菜单权限 SQL（T005），验证菜单层级和权限标识正确，在角色管理中分配权限
- [ ] T035 [P] 验证前端路由配置，确保"巡检管理"一级菜单和"巡检报告"二级菜单正确显示
- [ ] T036 端到端验证：登录系统 → 巡检管理 → 点击立即巡检 → 等待报告生成 → 查看详情 → 下载报告 → 验证 SSE 推送

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-6)**: All depend on Foundational phase completion
  - US1 (Phase 3): Can start after Foundational - No dependencies on other stories
  - US2 (Phase 4): Can start after Foundational - Frontend tasks depend on US1 Controller being defined but can be developed in parallel
  - US3 (Phase 5): Depends on US1's generateReport logic being complete
  - US4 (Phase 6): Depends on US1's generateReport logic being complete
- **Polish (Phase 7)**: Depends on all user stories being complete

### Within Each User Story

- Service impls before Controller
- PrometheusCollectorService and BailianReportService before InspectionReportServiceImpl
- Frontend types.ts before index.ts before views
- API module before view components

### Parallel Opportunities

- T001 → T002/T003/T004/T005 (setup tasks can run in parallel after module created)
- T006/T007/T008/T009/T010/T011/T012 (all BO/VO/Entity tasks can run in parallel)
- T013/T014 (Mapper interfaces can run in parallel)
- T015/T016 (Mapper XML can run in parallel)
- T024/T025/T027 (frontend types/API/component can run in parallel)

---

## Parallel Example: Phase 2 (Foundational)

```bash
# 并行创建所有实体和 VO
Task: "创建 InspectionReport.java 实体"
Task: "创建 InspectionInstanceMetric.java 实体"
Task: "创建所有 BO/VO 类"

# 然后并行创建 Mapper
Task: "创建 InspectionReportMapper.java + XML"
Task: "创建 InspectionInstanceMetricMapper.java + XML"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: 手动触发巡检 → 查看报告生成 → SSE 推送
5. 验证通过后可交付 MVP

### Incremental Delivery

1. Setup + Foundational → 基础就绪
2. Add US1 → 手动巡检可用（MVP!）
3. Add US2 → 报告管理+下载+前端页面
4. Add US3 → 重试功能
5. Add US4 → 定时自动巡检 + 过期清理
6. Polish → SQL 执行 + 端到端验证

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
