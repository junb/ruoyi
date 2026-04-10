# Feature Specification: 告警飞书转发

**Feature Branch**: `002-alert-feishu-forward`
**Created**: 2026-04-05
**Status**: Draft
**Input**: User description: "增加一个功能接口，将 alertmanager 和 loki 的告警转发推送到飞书，同时将告警记录下来"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - 接收 Alertmanager 告警并转发飞书 (Priority: P1)

Alertmanager 通过 Webhook 将告警信息推送到系统，系统解析告警内容，格式化为飞书消息卡片后发送到指定的飞书群机器人，同时将告警记录持久化到数据库。运维人员在飞书群中实时收到告警通知。

**Why this priority**: 这是核心价值——将 Alertmanager 的告警实时推送到飞书，让运维人员第一时间感知系统异常。

**Independent Test**: 配置 Alertmanager 的 webhook_configs 指向本接口，触发一条告警规则，验证飞书群收到消息卡片且数据库中存在该告警记录。

**Acceptance Scenarios**:

1. **Given** Alertmanager 已配置 Webhook URL 指向本系统， **When** Alertmanager 触发一条 firing 告警， **Then** 系统接收告警数据，向飞书群发送一条包含告警名称、严重级别、实例、摘要的消息卡片，同时数据库中新增一条告警记录。
2. **Given** Alertmanager 发送一条 resolved 告警， **When** 系统接收到该告警， **Then** 飞书群收到"告警恢复"消息卡片，数据库中对应的告警记录状态更新为已恢复。
3. **Given** Alertmanager 推送告警时飞书 Webhook 调用失败， **When** 系统检测到发送失败， **Then** 告警记录仍然持久化到数据库（飞书发送状态标记为失败），不丢失告警数据。

---

### User Story 2 - 告警记录查询与管理 (Priority: P2)

运维人员在管理界面中查看所有历史告警记录，支持按状态（firing/resolved）、严重级别、告警名称、时间范围进行筛选。可以查看告警详情（包括完整的原始数据和飞书发送结果），以及删除过期告警记录。

**Why this priority**: 告警记录的查询和管理是运维日常使用场景，但依赖 US1 的数据积累。

**Independent Test**: 通过已有告警数据验证列表查询（状态筛选、时间范围）、详情查看和删除操作。

**Acceptance Scenarios**:

1. **Given** 系统中存在多条告警记录， **When** 用户按状态或严重级别筛选， **Then** 列表只显示符合条件的告警记录。
2. **Given** 一条告警记录， **When** 用户点击查看详情， **Then** 展示告警完整信息包括原始 payload、关联的标签、飞书发送状态和响应结果。

---

### User Story 3 - 告警转发失败重试 (Priority: P2)

运维人员对飞书发送失败的告警记录点击"重试"按钮，系统重新调用飞书 Webhook 发送消息卡片。

**Why this priority**: 重试机制保证告警通知的可靠性，是运维场景的刚需。

**Independent Test**: 对一条飞书发送失败的告警记录执行重试操作，验证飞书消息重新发送。

**Acceptance Scenarios**:

1. **Given** 一条飞书发送失败的告警记录， **When** 用户点击"重试"， **Then** 系统重新向飞书发送消息卡片，更新发送状态。

---

### User Story 4 - 定时清理过期告警 (Priority: P3)

系统自动清理超过保留天数的告警记录，防止数据无限增长。

**Why this priority**: 数据清理是运维保障功能，优先级低于核心告警转发。

**Independent Test**: 配置短保留期，等待定时任务执行后验证过期记录被清理。

**Acceptance Scenarios**:

1. **Given** 保留天数配置为 90 天， **When** 定时清理任务执行， **Then** 超过 90 天的告警记录被自动删除。

---

### Edge Cases

- Alertmanager Webhook 请求体为空或格式异常时，系统返回 400 + 明确错误信息，不创建告警记录。
- 请求体 JSON 格式合法但 `alerts` 数组为空时，系统返回 200，不创建告警记录，日志记录 WARN 级别。
- 飞书 Webhook 地址无效或超时时，告警记录仍需持久化，飞书发送状态标记为失败。
- 同一告警（相同 fingerprint）短时间重复推送时，系统应更新已有记录而非重复创建。
- 已 resolved 的告警收到相同 fingerprint 的 firing 告警时，系统应将已有记录的状态更新回 firing，保留原始 starts_at，重置 ends_at，并**重新发送飞书通知**（因为这是一次新的告警触发事件）。
- Alertmanager 的 fingerprint 字段在正常情况下始终存在（16 位十六进制字符串），但若收到 fingerprint 为空的 alert，系统应跳过该条告警不持久化，日志记录 WARN。
- 单次 Webhook 包含大量告警（如 100 条）时，系统应串行遍历 alerts 数组逐条处理，每条独立持久化和飞书发送，单条失败不影响其他告警。

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: 系统必须提供一个 Webhook 接口接收 Alertmanager 的告警推送（兼容 Alertmanager v0.25+ 的 v2 API 格式）。
- **FR-002**: 系统必须解析 Alertmanager Webhook payload 中的每条 alert，提取以下字段：fingerprint、status、labels.alertname（告警名称）、labels.severity（严重级别）、labels.instance（实例地址）、labels.datasource（告警来源，默认 prometheus）、完整 labels 对象、annotations.summary（摘要）、annotations.description（描述）、startsAt、endsAt。不持久化 generatorURL、groupLabels、commonLabels 等非核心字段（仅存入 raw_payload）。
- **FR-003**: 系统必须将告警信息格式化为飞书消息卡片（纯展示，无交互按钮），通过飞书自定义机器人 Webhook 发送到指定群。卡片标题格式：`告警通知 - {alertname}`。firing 使用红色主题（header template: `red`），resolved 使用绿色主题（header template: `green`）。卡片 body 使用 Markdown 元素，按以下顺序展示：告警名称（加粗）、状态（firing=告警触发/resolved=已恢复）、严重级别、实例地址、摘要、告警开始时间、恢复时间（仅 resolved 时显示）。
- **FR-004**: 系统必须将每条告警记录持久化，包含告警 fingerprint、名称、状态、严重级别、实例、标签、摘要、原始 payload、接收时间、恢复时间、飞书发送状态。持久化与飞书发送在 Webhook 接口返回 200 后**异步并行执行**（非串行），两者互不依赖：持久化失败不阻塞飞书发送，飞书发送失败不阻塞持久化。
- **FR-005**: 系统必须支持告警记录的分页查询，支持按状态、严重级别、告警名称、时间范围筛选。
- **FR-006**: 系统必须支持查看告警详情，包括完整的原始 payload 和飞书发送结果。
- **FR-007**: 系统必须支持删除告警记录。
- **FR-008**: 系统必须支持对飞书发送失败的告警记录进行重试。
- **FR-009**: 系统必须支持定时清理超过保留天数的告警记录。
- **FR-010**: 告警状态必须包含两种：firing（告警中）、resolved（已恢复）。注：Alertmanager 的 suppressed 告警不会发送到 Webhook，因此系统无需处理 suppressed 状态。
- **FR-011**: 同一 fingerprint 的告警 firing → resolved 状态流转必须更新同一条记录而非新建记录。
- **FR-012**: 飞书消息卡片必须区分 firing（红色警告风格）和 resolved（绿色恢复风格）。
- **FR-013**: 飞书首次发送失败后，系统自动按递增间隔（2s/3s/5s）最多重试 3 次。自动重试全部失败后标记为 `failed`，用户可通过界面的"重试"按钮手动触发重新发送（手动重试也会执行完整的 3 次自动重试策略）。
- **FR-014**: 权限控制必须覆盖三个权限点：查询、删除、重试。
- **FR-015**: 告警记录页面必须作为"巡检管理"菜单下的子菜单"告警记录"呈现。
- **FR-016**: 飞书 Webhook 地址和保留天数通过系统字典配置管理。
- **FR-017**: Webhook 接口路径 `/webhook/alert/**` 必须在系统安全配置（`application.yml` 的 `security.excludes`）中排除 Sa-Token 认证拦截和多租户插件拦截，确保 Alertmanager 无需登录态即可调用。
- **FR-018**: 飞书 API 返回非零 code（如频率限制 9499、签名失败 19021）时，系统必须将发送状态标记为 `failed`，并在 `feishu_response` 中记录完整的错误码和消息。

### Key Entities

- **告警记录（Alert Record）**: 一次告警事件的完整记录，包含 fingerprint、告警名称、状态（firing/resolved）、严重级别、实例地址、标签集合（JSON）、摘要、描述、原始 payload（JSON）、接收时间、恢复时间、飞书发送状态、飞书响应结果。同一 fingerprint 的 firing → resolved 状态流转更新同一条记录。
- **飞书消息卡片（Feishu Card）**: 由系统根据告警信息动态构建的飞书交互式卡片消息,firing 告警使用红色警告风格,resolved 使用绿色恢复风格,包含告警标题、状态标签、实例信息、标签详情。
- **系统配置（字典配置）**: 飞书 Webhook 地址、保留天数、消息模板 ID 等运行参数。

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 从系统 Webhook 接口收到 Alertmanager HTTP 请求并返回 200 开始计时，飞书群中显示消息卡片的端到端延迟不超过 5 秒（不含 Alertmanager 到系统的网络延迟）。
- **SC-002**: 告警数据 100% 持久化,即使飞书发送失败也不丢失任何告警记录。
- **SC-003**: 告警记录列表查询响应时间在 2 秒以内（数据量不超过 10 万条）。
- **SC-004**: 飞书消息发送失败后,自动重试能在 30 秒内完成（3 次重试含等待时间）。
- **SC-005**: 同一告警的 firing → resolved 状态流转能正确关联,无重复记录。

## Clarifications

### Session 2026-04-05

- Q: 飞书消息发送使用哪种集成方式？ → A: 飞书自定义机器人 Webhook（向一个固定 URL POST JSON，简单无认证）
- Q: 告警 Webhook 接口的处理模式？ → A: 异步（接收告警 → 立即返回 → 后台并行执行持久化和飞书发送）。持久化和飞书发送互不依赖，任一失败不影响另一个。
- Q: 告警记录放在哪个模块？ → A: 放在已有 `ruoyi-inspection` 模块中，与巡检功能共享基础设施，作为"巡检管理"菜单下的子功能。
- Q: Webhook 接口的鉴权方式？ → A: URL 路径变量（如 `/webhook/alert/{token}`），token 通过字典配置，Alertmanager 直接配在 webhook URL 中即可。
- Q: 飞书消息卡片是否需要交互按钮？ → A: 纯展示卡片，只展示告警信息和状态颜色，不含交互按钮。

## Assumptions

- Alertmanager 版本为 v0.25+,使用 v2 API 格式的 Webhook payload。
- 飞书自定义机器人 Webhook 已创建,URL 已获取。
- Loki 告警通过 Alertmanager 统一转发,系统只需根据告警标签（如 `datasource=loki`）区分来源。
- 本功能运行在 RuoYi-Vue-Plus 框架内,复用现有的用户认证、权限管理、多租户等基础设施。
- 告警记录的保留策略通过字典配置保留天数。
- Webhook 接口使用 URL 路径变量 token 进行简单鉴权（如 `/webhook/alert/{token}`），Alertmanager 直接将 token 嵌入 webhook URL 配置即可。
