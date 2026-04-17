# Tasks: 磁盘容量预测

**Input**: Design documents from `/specs/003-disk-prediction/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/api.md

**Organization**: Tasks grouped by user story — each independently implementable and testable.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 数据库建表、字典配置、菜单权限

- [x] T001 Create SQL script `RuoYi-Vue-Plus/script/sql/disk_prediction.sql` — disk_usage_history 建表（含审计字段）、字典类型 prometheus_inspection 新增 DISK_RETENTION_DAYS、菜单 2080 资源预测及权限按钮（inspection:prediction:query）
- [ ] T002 Execute disk_prediction.sql against database to create table and menu entries

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 核心实体和 Mapper，所有 User Story 都依赖

**⚠️ CRITICAL**: 此阶段完成前不可开始任何 User Story

- [x] T003 Create `DiskUsageHistory` entity in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/domain/DiskUsageHistory.java` — extends TenantEntity, @TableName("disk_usage_history"), fields: id, instanceIp, mountPoint, totalBytes, usedBytes, usagePercent, collectedTime, delFlag
- [x] T004 [P] Create `DiskUsageHistoryMapper` in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/mapper/DiskUsageHistoryMapper.java` — extends BaseMapperPlus<DiskUsageHistory, DiskUsageHistoryVo>
- [x] T005 [P] Create mapper XML `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/resources/mapper/inspection/DiskUsageHistoryMapper.xml`
- [x] T006 [P] Create `DiskUsageHistoryVo` in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/domain/vo/DiskUsageHistoryVo.java` — with @AutoMapper(target=DiskUsageHistory.class)

**Checkpoint**: Foundation ready — user story implementation can begin

---

## Phase 3: User Story 1 - 定时采集磁盘历史数据 (Priority: P1) 🎯 MVP

**Goal**: 系统每 6 小时自动采集磁盘快照并落库，自动清理过期数据

**Independent Test**: 手动触发定时任务后，检查 disk_usage_history 表中是否有新数据写入

### Implementation for User Story 1

- [x] T007 [US1] Create `IDiskUsageHistoryService` interface in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/service/IDiskUsageHistoryService.java` — methods: collectAndSave(), cleanExpiredHistory()
- [x] T008 [US1] Implement `DiskUsageHistoryServiceImpl` in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/service/impl/DiskUsageHistoryServiceImpl.java` — collectAndSave: 复用 PrometheusCollectorService.collectDiskPartitions() 遍历构建 DiskUsageHistory 批量插入；cleanExpiredHistory: 根据字典配置的保留天数清理旧数据
- [x] T009 [US1] Create `DiskUsageCollectJob` in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/job/DiskUsageCollectJob.java` — @JobExecutor(name="diskUsageCollectJob"), 调用 collectAndSave() + cleanExpiredHistory()
- [x] T010 [US1] Compile and verify — run `mvn compile -pl ruoyi-modules/ruoyi-inspection -am` to ensure no compilation errors

**Checkpoint**: 定时任务可独立运行，数据库中产生磁盘历史数据

---

## Phase 4: User Story 2 - 查看磁盘容量预测概览 (Priority: P2)

**Goal**: 独立预测看板页面，展示所有磁盘的预测概览、趋势图和风险等级

**Independent Test**: 打开资源预测页面，查看概览表格和详情抽屉中的趋势图

### Implementation for User Story 2

- [x] T011 [P] [US2] Create `PredictionPointVo` in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/domain/vo/PredictionPointVo.java` — fields: daysAhead, predictedUsedBytes, predictedUsagePercent
- [x] T012 [P] [US2] Create `DiskPredictionVo` in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/domain/vo/DiskPredictionVo.java` — fields: instanceIp, mountPoint, totalBytes, currentUsedBytes, currentUsagePercent, dailyGrowthBytes, dailyGrowthMB, estimatedFullDate, remainingDays, riskLevel, dataPoints, predictions list
- [x] T013 [US2] Create `IDiskPredictionService` interface in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/service/IDiskPredictionService.java` — methods: predictAll(instanceIp, mountPoint), predictDetail(instanceIp, mountPoint), queryHistory(instanceIp, mountPoint, days)
- [x] T014 [US2] Implement `DiskPredictionServiceImpl` in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/service/impl/DiskPredictionServiceImpl.java` — 线性回归算法（最小二乘法）、边界处理（数据不足/斜率<=0）、查询历史数据。风险等级严格按阈值：high（使用率>85%且剩余<30天）、medium（使用率>70%或剩余<90天）、low（其他）、insufficient_data（数据点<10）。挂载点已卸载（最近7天无新数据）标记 riskLevel="expired"
- [x] T015 [US2] Create `DiskPredictionController` in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/controller/DiskPredictionController.java` — @RequestMapping("/inspection/prediction"), 三个接口: /list, /detail, /history, @SaCheckPermission("inspection:prediction:query")
- [x] T016 [P] [US2] Create frontend types `plus-ui/src/api/inspection/prediction/types.ts` — DiskPredictionVo, PredictionPointVo, DiskUsageHistoryVo, DiskPredictionQuery interfaces
- [x] T017 [P] [US2] Create frontend API `plus-ui/src/api/inspection/prediction/index.ts` — listPredictions, detailPrediction, historyPrediction functions
- [x] T018 [US2] Create prediction page `plus-ui/src/views/inspection/prediction/index.vue` — 概览表格（实例/挂载点/使用率/日均增长/满载日期/风险等级 tag：high红色/medium黄色/low绿色/insufficient_data灰色"数据不足"/expired灰色"已过期"）+ 详情抽屉（ECharts 趋势图：实际数据点+拟合线+预测虚线 + 预测节点表格 + 关键指标卡片）
- [x] T019 [US2] Compile and verify — backend `mvn compile`, frontend `npm run build:dev`

**Checkpoint**: 资源预测页面完整可用，展示预测概览和趋势图

---

## Phase 5: User Story 3 - 巡检报告集成磁盘预测摘要 (Priority: P3)

**Goal**: 巡检报告 AI prompt 中追加磁盘预测摘要

**Independent Test**: 触发巡检报告生成，检查报告 markdown 中是否包含磁盘预测章节

### Implementation for User Story 3

- [x] T020 [US3] Modify `BailianReportService.generateReport()` in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/service/impl/BailianReportService.java` — 注入 IDiskPredictionService，在构建 prompt 前调用 predictAll() 获取预测摘要，追加到 prompt 中，格式："磁盘预测摘要：\n实例 192.168.2.161 /data 使用率 72%，日均增长 50MB，预计 45 天后满载\n..."。若无预测数据则追加"磁盘预测：数据不足，暂无法预测"
- [x] T021 [US3] Compile and verify — `mvn compile -pl ruoyi-modules/ruoyi-inspection -am`

**Checkpoint**: 巡检报告中包含磁盘预测摘要

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: 收尾和验证

- [x] T022 Run quickstart.md validation — verify SQL executed, SnailJob task created, page accessible
- [x] T023 ESLint check frontend code — `cd plus-ui && npm run lint:eslint`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 — BLOCKS all user stories
- **US1 (Phase 3)**: Depends on Phase 2 — 数据采集是基础
- **US2 (Phase 4)**: Depends on Phase 2 + Phase 3 — 预测页面需要历史数据
- **US3 (Phase 5)**: Depends on Phase 2 + Phase 3 — 报告集成需要预测服务
- **Polish (Phase 6)**: Depends on all stories complete

### User Story Dependencies

- **US1 (P1)**: Depends on Foundational only — 独立可测
- **US2 (P2)**: Depends on US1（需要历史数据才有预测结果）— 但代码层面可并行开发
- **US3 (P3)**: Depends on US2（需要 DiskPredictionService）— 串行

### Within Each User Story

- Models/VOs before Services
- Services before Controllers
- Backend before Frontend
- Compile/verify as final step

### Parallel Opportunities

- Phase 2: T004, T005, T006 can run in parallel (different files)
- Phase 4: T011, T012 can run in parallel; T016, T017 can run in parallel

---

## Parallel Example: Phase 4 (User Story 2)

```bash
# Launch VO creation in parallel:
Task: "Create PredictionPointVo in .../domain/vo/PredictionPointVo.java"
Task: "Create DiskPredictionVo in .../domain/vo/DiskPredictionVo.java"

# Launch frontend files in parallel (after backend APIs done):
Task: "Create frontend types in plus-ui/src/api/inspection/prediction/types.ts"
Task: "Create frontend API in plus-ui/src/api/inspection/prediction/index.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (SQL)
2. Complete Phase 2: Foundational (Entity + Mapper)
3. Complete Phase 3: User Story 1 (定时采集)
4. **STOP and VALIDATE**: 检查数据库中是否有磁盘历史数据
5. 等待数据积累后再开发预测页面

### Incremental Delivery

1. Setup + Foundation → 表和基础代码就绪
2. Add US1 → 定时采集运行 → **MVP（数据基础就位）**
3. Add US2 → 预测看板页面 → 核心价值交付
4. Add US3 → 巡检报告集成 → 增值功能
5. Polish → 收尾验证

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story
- US2 依赖 US1 的数据但代码可并行开发（用 mock 数据测试）
- 数据库 SQL 需用户手动执行（T002）
- 前端 ECharts 已在项目中引入，无需额外安装
