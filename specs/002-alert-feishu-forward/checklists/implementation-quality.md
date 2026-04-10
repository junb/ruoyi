# Implementation Quality Checklist

**Purpose**: Validate spec/plan/tasks 中需求的完整性、清晰性、一致性，确保实现前所有需求明确无歧义
**Created**: 2026-04-06
**Focus**: 实现质量、API 契约质量、安全与权限
**Depth**: Standard
**Actor**: Author + Reviewer

---

## Requirement Completeness（需求完整性）

- [ ] CHK001 Alertmanager v2 Webhook payload 的完整字段映射是否在 spec 中明确列出？当前 spec FR-002 仅列出"告警名称、严重级别、实例地址、标签集合、摘要、描述"，但未涵盖 `generatorURL`、`groupLabels`、`commonLabels` 等字段是否需要持久化 [Completeness, Spec §FR-002]
- [ ] CHK002 飞书消息卡片的具体内容布局是否在 spec 中定义？FR-003 仅说明"纯展示卡片"和"区分红/绿风格"，但未指定卡片中应展示哪些字段及其排列顺序 [Completeness, Spec §FR-003]
- [ ] CHK003 Edge Case"同一 fingerprint 短时间重复推送时更新已有记录"是否与 FR-011 的"firing→resolved 状态流转更新同一条记录"在 spec 中明确区分为两种不同场景？ [Consistency, Spec §FR-011]
- [ ] CHK004 单次 Webhook 包含大量告警（如 100 条）的批量处理需求是否指定了处理策略（串行 vs 并行、超时控制、部分失败处理）？ [Completeness, Edge Case, Spec §Edge Cases]
- [ ] CHK005 FR-013 规定"最多 3 次重试，间隔递增"，但 spec 中未定义首次发送失败后重试的触发时机——是在 Webhook 接收时立即自动重试，还是仅由用户手动触发？ [Clarity, Spec §FR-013]
- [ ] CHK006 告警来源（`alert_source`）字段的取值范围是否明确？Spec 提到"通过 `datasource=loki` 区分来源"，但未定义当标签中无 `datasource` 时的默认值及其他可能值 [Clarity, Spec §Assumptions]

## API Contract Quality（API 契约质量）

- [ ] CHK007 Webhook 接口 `/webhook/alert/{token}` 的错误响应格式是否与 Alertmanager 的期望一致？Alertmanager 对非 200 响应会重试，spec 中未明确 400/401 响应是否会触发 Alertmanager 重试 [Completeness, Contracts §1]
- [ ] CHK008 告警记录列表 API 的 `beginTime`/`endTime` 参数格式是否明确为 `yyyy-MM-dd` 还是 `yyyy-MM-dd HH:mm:ss`？contracts 中标注为 `yyyy-MM-dd`，但前端使用 `YYYY-MM-DD HH:mm:ss` 格式 [Consistency, Contracts §2]
- [ ] CHK009 删除接口 `DELETE /inspection/alarm/{ids}` 中 `ids` 的分隔符（逗号 vs 路径数组）是否在 contracts 中明确？路径参数 `{ids}` 与前端 `Array<string|number>` 类型存在歧义 [Clarity, Contracts §2]
- [ ] CHK010 重试接口 `POST /inspection/alarm/retry` 的请求体是否需要校验告警记录当前状态？spec 中未定义对 `feishuSendStatus` 非 `failed` 的记录执行重试时的预期行为 [Completeness, Contracts §2]
- [ ] CHK011 Webhook 接口的请求体大小限制是否定义？单次推送包含 100 条告警时 payload 可能超过默认限制 [Gap, Non-Functional]
- [ ] CHK012 所有 CRUD 接口的分页参数默认值（`pageNum`、`pageSize`）是否与框架默认一致？contracts 中标注"默认 1 / 10"但未引用框架配置 [Traceability, Contracts §2]

## Security & Permissions（安全与权限）

- [ ] CHK013 Webhook token 的安全性要求是否明确？当前使用 URL 路径变量传递 token，spec 中未定义 token 的最小长度、复杂度要求或是否支持 token 轮换 [Clarity, Spec §FR-001]
- [ ] CHK014 `/webhook/alert/**` 路径排除 Sa-Token 认证拦截的配置是否在 spec 或 plan 中明确标注为必须步骤？当前仅在 plan.md 的 Constitution Check 中以"需注意"提及，未作为 FR 列出 [Completeness, Gap]
- [ ] CHK015 Webhook 接口的租户插件排除是否与多租户需求一致？plan 中提到"手动设置 tenant_id"，但未明确租户插件是否也需要排除该路径 [Clarity, Plan §Constitution Check]
- [ ] CHK016 飞书 Webhook URL 在字典配置中的存储安全性是否考虑？URL 包含敏感 hook token，spec 中未定义是否需要加密存储或脱敏展示 [Gap, Security]
- [ ] CHK017 权限点 `inspection:alarm:query/remove/retry` 与菜单按钮权限的对应关系是否完整？contracts 中定义了 3 个权限点，但 spec FR-014 提到"三个权限点：查询、删除、重试"，未包含导出权限 [Consistency, Spec §FR-014 vs Contracts §3]

## Requirement Clarity（需求清晰性）

- [ ] CHK018 FR-004"持久化与飞书发送互不依赖"是否明确为并行执行还是串行但容错？spec 和 plan 中的措辞在不同位置不一致 [Ambiguity, Spec §FR-004 vs Plan]
- [ ] CHK019 SC-001"5 秒内到达飞书"的测量起止点是否明确？是从 Alertmanager 发送 HTTP 请求开始计时，还是从系统收到 Webhook 开始计时？ [Measurability, Spec §SC-001]
- [ ] CHK020 飞书消息卡片中"firing 使用红色警告风格，resolved 使用绿色恢复风格"（FR-012）的具体实现标准是否明确？spec 中未定义卡片 template 值（如 `red`/`green`）和内容模板 [Clarity, Spec §FR-012]

## Scenario Coverage（场景覆盖）

- [ ] CHK021 已 resolved 的告警收到相同 fingerprint 的 firing 告警时，spec Edge Cases 定义了"状态更新回 firing"，但未明确是否需要重新发送飞书通知（当前实现会重发，spec 是否确认？） [Coverage, Edge Case]
- [ ] CHK022 Alertmanager Webhook 请求体 JSON 格式合法但 `alerts` 数组为空的场景是否在 spec 中定义？当前实现会静默返回，但 spec 未明确期望行为 [Coverage, Gap]
- [ ] CHK023 飞书 API 返回非零 code（如频率限制 9499）时，系统是否应将其视为发送成功还是失败？FR-013 的重试策略仅提到"发送失败"但未定义失败判定标准 [Clarity, Spec §FR-013]
- [ ] CHK024 告警记录的 `fingerprint` 字段是否可能为空？Alertmanager 文档中 fingerprint 始终存在，但 spec 未定义当 fingerprint 为空时的处理策略 [Edge Case, Gap]

## Dependencies & Assumptions（依赖与假设）

- [ ] CHK025 假设"飞书自定义机器人 Webhook 已创建"是否意味着系统不需要提供 Webhook 地址的连通性测试功能？ [Assumption, Spec §Assumptions]
- [ ] CHK026 假设"Loki 告警通过 Alertmanager 统一转发"是否排除了系统直接接收 Loki Alerting Rules 的场景？ [Assumption Validation, Spec §Assumptions]
