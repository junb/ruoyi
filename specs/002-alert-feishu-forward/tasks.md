# Tasks: 告警飞书转发

**Input**: Design documents from `/specs/002-alert-feishu-forward/`
**Prerequisites**: plan.md, spec.md, data-model.md, contracts/api.md, research.md

**Tests**: 未显式要求，不生成测试任务。

**Organization**: 按 User Story 分组，每个 Story 可独立实现和测试。

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 可并行执行（不同文件，无依赖）
- **[Story]**: 所属用户故事（US1, US2, US3, US4）
- 包含精确文件路径

---

## Phase 1: Setup（数据库与配置）

**Purpose**: 创建数据库表、字典配置、菜单权限

- [x] T001 创建告警记录表和字典数据 SQL 脚本 in `RuoYi-Vue-Plus/script/sql/alert.sql`
  - CREATE TABLE alert_record（见 data-model.md DDL）
  - INSERT 字典类型 alert_config 及 4 个配置项（FEISHU_WEBHOOK_URL, WEBHOOK_TOKEN, RETENTION_DAYS, DEFAULT_TENANT_ID）
  - INSERT 菜单：告警记录（2076, parent=2070）及 3 个按钮权限（query/remove/retry, menu_id 2077-2079）
- [ ] T002 执行 alert.sql 初始化数据库（建表 + 字典 + 菜单权限）

---

## Phase 2: Foundational（基础代码骨架）

**Purpose**: 创建所有实体、BO、VO、Mapper 等基础类，为所有 User Story 提供共享基础设施

**⚠️ CRITICAL**: 所有 User Story 的实现必须在此阶段完成后才能开始

- [x] T003 [P] 创建 AlertRecord 实体类 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/domain/AlertRecord.java`
- [x] T004 [P] 创建 AlertRecordBo 查询 BO in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/domain/bo/AlertRecordBo.java`
- [x] T005 [P] 创建 AlertRecordVo 列表 VO in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/domain/vo/AlertRecordVo.java`
- [x] T006 [P] 创建 AlertRecordDetailVo 详情 VO in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/domain/vo/AlertRecordDetailVo.java`
- [x] T007 [P] 创建 AlertRecordMapper 接口 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/mapper/AlertRecordMapper.java`
- [x] T008 [P] 创建 AlertRecordMapper.xml in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/resources/mapper/inspection/AlertRecordMapper.xml`
- [x] T009 创建 IAlertRecordService 接口 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/service/IAlertRecordService.java`

**Checkpoint**: 基础类就绪，User Story 实现可以开始

---

## Phase 3: User Story 1 - 接收 Alertmanager 告警并转发飞书 (Priority: P1) 🎯 MVP

**Goal**: Alertmanager 通过 Webhook 推送告警，系统解析并异步转发飞书消息卡片，同时持久化告警记录

**Independent Test**: 配置 Alertmanager webhook_configs 指向本接口，触发告警规则，验证飞书群收到消息卡片且数据库中存在告警记录

### Implementation for User Story 1

- [x] T010 [US1] 创建 FeishuNotifyService 飞书消息发送服务 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/service/impl/FeishuNotifyService.java`
  - 构建飞书消息卡片 JSON（卡片 2.0 格式，firing 红色/resolved 绿色）
  - 使用 Hutool HttpRequest 发送 POST 请求到飞书 Webhook
  - 实现重试策略：最多 3 次，间隔 2s/3s/5s
  - 从字典配置（alert_config.FEISHU_WEBHOOK_URL）获取 Webhook 地址
- [x] T011 [US1] 实现 AlertRecordServiceImpl 核心逻辑 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/service/impl/AlertRecordServiceImpl.java`
  - 解析 Alertmanager Webhook payload（alerts 数组、fingerprint、labels、annotations）
  - 异步处理：持久化与飞书发送并行互不依赖
  - fingerprint 关联：firing→resolved 更新同一条记录
  - 多租户：从字典配置读取默认租户 ID
  - 批量处理：单次 Webhook 包含多条告警全部处理
- [x] T012 [US1] 创建 AlertWebhookController Webhook 接口 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/controller/AlertWebhookController.java`
  - POST /webhook/alert/{token}
  - token 验证（与字典配置 alert_config.WEBHOOK_TOKEN 比较）
  - 请求体非空校验
  - 无需 @SaCheckPermission（无登录态）
  - **⚠️ 重要**：此接口路径 `/webhook/alert/` 必须排除 Sa-Token 认证拦截和租户插件拦截（在 application.yml 的 sa-token.excludePathPatterns 或安全配置中添加排除路径）
  - 返回 200/400/401

**Checkpoint**: 此时可独立测试——Alertmanager 推送告警，飞书收到消息卡片，数据库有记录

---

## Phase 4: User Story 2 - 告警记录查询与管理 (Priority: P2)

**Goal**: 运维人员在管理界面查看历史告警记录，支持筛选、详情查看和删除

**Independent Test**: 通过已有告警数据验证列表查询（状态筛选、时间范围）、详情查看和删除操作

### Implementation for User Story 2

- [x] T013 [US2] 完善 AlertRecordServiceImpl 的查询/详情/删除方法 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/service/impl/AlertRecordServiceImpl.java`
  - selectPageList：分页查询，支持 status/severity/alertName/beginTime/endTime 筛选
  - selectDetailById：查询详情含完整 rawPayload 和飞书响应
  - deleteByIds：批量逻辑删除
- [ ] T014 [US2] 创建 AlertRecordController CRUD 接口 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/controller/AlertRecordController.java`
  - GET /inspection/alarm/list — @SaCheckPermission("inspection:alarm:query")
  - GET /inspection/alarm/{id} — @SaCheckPermission("inspection:alarm:query")
  - DELETE /inspection/alarm/{ids} — @SaCheckPermission("inspection:alarm:remove")
- [ ] T015 [P] [US2] 创建前端 TypeScript 类型定义 in `plus-ui/src/api/inspection/alarm/types.ts`
  - AlertRecordQuery、AlertRecordVo、AlertRecordDetailVo 接口定义
- [ ] T016 [P] [US2] 创建前端 API 模块 in `plus-ui/src/api/inspection/alarm/index.ts`
  - listAlarm、detailAlarm、deleteAlarm 函数
- [ ] T017 [US2] 创建告警记录列表页面 in `plus-ui/src/views/inspection/alarm/index.vue`
  - el-table 展示：告警名称、状态（firing 红色/resolved 绿色 tag）、严重级别、实例、摘要、飞书发送状态、接收时间
  - 筛选：状态下拉、严重级别下拉、告警名称搜索、时间范围
  - 操作：查看详情、删除
  - 分页
- [ ] T018 [US2] 创建告警详情页面 in `plus-ui/src/views/inspection/alarm/detail.vue`
  - el-descriptions 展示基本信息
  - 原始 payload JSON 展示（折叠/展开）
  - 飞书发送状态和响应结果

**Checkpoint**: 此时可独立测试——告警记录列表筛选、详情查看、删除操作

---

## Phase 5: User Story 3 - 告警转发失败重试 (Priority: P2)

**Goal**: 运维人员对飞书发送失败的告警点击"重试"重新发送

**Independent Test**: 对一条飞书发送失败的告警记录执行重试操作，验证飞书消息重新发送

### Implementation for User Story 3

- [ ] T019 [US3] 在 AlertRecordServiceImpl 添加 retrySend 方法 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/service/impl/AlertRecordServiceImpl.java`
  - 查询告警记录，验证飞书发送状态为 failed
  - 调用 FeishuNotifyService 重新发送
  - 更新发送状态
- [ ] T020 [US3] 在 AlertRecordController 添加重试接口 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/controller/AlertRecordController.java`
  - POST /inspection/alarm/retry — @SaCheckPermission("inspection:alarm:retry")
- [ ] T021 [US3] 在前端 API 模块添加 retryAlarm 函数 in `plus-ui/src/api/inspection/alarm/index.ts`
- [ ] T022 [US3] 在告警记录列表页添加重试按钮 in `plus-ui/src/views/inspection/alarm/index.vue`
  - 仅 feishuSendStatus 为 failed 时显示"重试"按钮
  - 重试成功后刷新列表

**Checkpoint**: 此时可独立测试——对失败告警点击重试，飞书重新发送消息

---

## Phase 6: User Story 4 - 定时清理过期告警 (Priority: P3)

**Goal**: 系统自动清理超过保留天数的告警记录

**Independent Test**: 配置短保留期，等待定时任务执行后验证过期记录被清理

### Implementation for User Story 4

- [x] T023 [US4] 在 AlertRecordServiceImpl 添加 cleanExpiredAlerts 方法 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/service/impl/AlertRecordServiceImpl.java`
  - 从字典配置（alert_config.RETENTION_DAYS）获取保留天数
  - 逻辑删除超过保留天数的记录
- [x] T024 [US4] 创建 AlertCleanupJob 定时任务 in `RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/job/AlertCleanupJob.java`
  - @JobExecutor(name="alertCleanupJob")
  - 调用 cleanExpiredAlerts

**Checkpoint**: 此时可独立测试——配置短保留期，定时任务清理过期记录

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: 编译验证和收尾

- [x] T025 编译验证：运行 `mvn clean compile -pl ruoyi-admin -am` 确保后端编译通过
- [x] T026 [P] ESLint 检查前端代码：运行 `cd plus-ui && npm run lint:eslint`
- [ ] T027 验证菜单权限：确认"巡检管理 → 告警记录"菜单和按钮权限正确显示

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 无依赖，立即开始
- **Foundational (Phase 2)**: 依赖 Phase 1 完成 — BLOCKS 所有 User Story
- **User Story 1 (Phase 3)**: 依赖 Phase 2 完成 — MVP 核心
- **User Story 2 (Phase 4)**: 依赖 Phase 2 完成（US2 前端查询依赖 US1 产出的数据）
- **User Story 3 (Phase 5)**: 依赖 Phase 2 + Phase 4 完成（重试按钮在列表页上）
- **User Story 4 (Phase 6)**: 依赖 Phase 2 完成
- **Polish (Phase 7)**: 依赖所有 User Story 完成

### User Story Dependencies

- **US1 (P1)**: Phase 2 后即可开始，无其他 Story 依赖 — 🎯 MVP
- **US2 (P2)**: Phase 2 后可开始后端部分，前端页面独立
- **US3 (P2)**: 依赖 US2 的列表页（重试按钮在列表页上）
- **US4 (P3)**: Phase 2 后即可开始，完全独立

### Parallel Opportunities

- Phase 2 中 T003-T008 全部可并行（不同文件）
- Phase 4 中 T015/T016 可并行（前端类型和 API）
- Phase 5 和 Phase 6 可并行（完全不同文件）

---

## Parallel Example: Phase 2

```text
# 同时创建所有基础类：
T003 AlertRecord.java
T004 AlertRecordBo.java
T005 AlertRecordVo.java
T006 AlertRecordDetailVo.java
T007 AlertRecordMapper.java
T008 AlertRecordMapper.xml
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: 执行 SQL 建表
2. Phase 2: 创建基础类
3. Phase 3: 实现 Webhook 接收 + 飞书转发 + 持久化
4. **STOP and VALIDATE**: curl 模拟 Alertmanager Webhook 调用，验证飞书收到消息 + 数据库有记录

### Incremental Delivery

1. US1 → 告警接收和飞书转发可用（MVP）
2. US2 → 管理界面可查看/筛选/删除告警
3. US3 → 重试失败的飞书发送
4. US4 → 定时清理过期记录

---

## Notes

- [P] 任务 = 不同文件，无依赖
- [Story] 标签将任务映射到具体 User Story
- 每个 User Story 应可独立完成和测试
- 每个 Checkpoint 停下来验证
- 提交信息格式：`巡检管理: 简要描述`
