# Implementation Quality Checklist: 服务器自动巡检

**Purpose**: 全面审查需求文档（spec、plan、data-model、api 契约）的完整性、清晰性、一致性和可测性
**Created**: 2026-04-05
**Feature**: [spec.md](../spec.md) | [plan.md](../plan.md) | [data-model.md](../data-model.md) | [api.md](../contracts/api.md)
**Scope**: 覆盖功能需求、数据模型、API 契约、异步流程、安全、前端

---

## Requirement Completeness（需求完整性）

- [ ] CHK001 spec 中是否为每种可能的 Prometheus 查询失败场景定义了明确的需求？如：网络超时、返回非 200、返回 status≠success、result 数组为空 [Completeness, Gap]
- [ ] CHK002 spec 中是否定义了百炼 AI 返回空内容或格式异常时的处理需求？[Completeness, Gap, FR-003]
- [ ] CHK003 spec 中"报告名称格式 `{年}年{月}月服务器巡检报告_{YYYYMMDD_HHmm}`"是否明确了同一天多次触发时名称重复的处理策略？[Completeness, Spec §Key Entities]
- [ ] CHK004 spec 中是否定义了定时巡检的 cron 表达式配置方式？是硬编码还是通过 SnailJob 动态配置？[Completeness, Gap, FR-009]
- [ ] CHK005 spec 中是否定义了磁盘分区 `device` 字段（如 `/dev/sda1`）的数据来源？Prometheus 原始数据中是否包含此字段？[Completeness, data-model disk_partitions]
- [ ] CHK006 spec 中 `LABEL_FILTERS` 字典配置项的定义是否明确？它如何影响 Prometheus 查询？是追加到 PromQL 还是过滤结果？[Completeness, Gap]
- [ ] CHK007 spec 中是否定义了"生成耗时"的起止时间点？是从报告创建开始还是从 Prometheus 采集开始？[Completeness, Clarity, data-model generationDuration]
- [ ] CHK008 spec 中是否定义了 SSE 推送失败时（如用户未连接 SSE）的回退策略？用户是否只能手动刷新？[Completeness, Gap, FR-015]
- [ ] CHK009 spec 中是否定义了并发巡检报告的数量上限？是否存在资源耗尽风险？[Completeness, Gap, Non-Functional]
- [ ] CHK010 spec 中是否定义了百炼 AI API 的超时时间？60 秒（HTTP client timeout）是否与 SC-002 的 60 秒总量目标矛盾？[Completeness, Consistency, SC-002]

## Requirement Clarity（需求清晰性）

- [ ] CHK011 spec 中 FR-002"CPU 使用率"查询语句 `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` 是否能返回正确的百分比？rate() 结果可能为小数（如 0.18），直接乘 100 是否与 DECIMAL(5,2) 精度匹配？[Clarity, 需求文档 §2.2]
- [ ] CHK012 spec 中"instance IP 从 Prometheus 的 `instance` 标签中提取，去掉端口部分"——这个规则是否处理了 IPv6 地址（含冒号）的情况？[Clarity, Edge Case]
- [ ] CHK013 data-model 中 `instance_name` 字段标注为"否"（非必填），但来源未定义。是来自 Prometheus 的 `instance` 标签还是其他数据源？[Clarity, Gap, data-model]
- [ ] CHK014 spec 中 FR-014"权限控制必须覆盖四个权限点：查询、创建（含重试）、下载、删除"——但 api.md 中重试接口使用的是 `inspection:report:create` 权限，这个合并是否在 spec 中有明确说明？[Clarity, Spec FR-014 vs api.md]
- [ ] CHK015 spec 中"状态变更必须立即持久化，不能等整体业务事务结束"——这个约束是否意味着每个状态变更需要独立事务？与 @Async 方法的结合方式是否已明确？[Clarity, data-model §状态流转]
- [ ] CHK016 spec 中 SC-002"单次巡检能在 60 秒内完成"是否包含重试时间？如果百炼重试 3 次（2s+4s+8s 等待 + 每次调用时间），总时间可能超过 60 秒 [Clarity, Consistency, SC-002 vs FR-013]

## Requirement Consistency（需求一致性）

- [ ] CHK017 spec FR-006 要求"下载报告为 Markdown 文件"，但 api.md 的 download 接口返回的是原始 markdownContent。是否需要包含实例指标数据？下载的文件内容与 AI 生成的原始 Markdown 是否一致？[Consistency, Spec FR-006 vs api.md]
- [ ] CHK018 data-model 中 `prometheus_inspection_instance_metric` 表缺少 `create_dept` 字段，但父表 `prometheus_inspection_report` 包含此字段。两张表都继承 TenantEntity，是否一致？[Consistency, data-model]
- [ ] CHK019 spec FR-011 定义三种状态 `pending/success/failed`，但 sql/inspection.sql 中 status 默认值为 `'pending'`。FR-011 中是否应明确说明初始状态？[Consistency, Spec FR-011 vs data-model]
- [ ] CHK020 spec §Edge Cases 提到"某台服务器没有返回指标数据时，该实例的指标字段为空，不影响其他实例"——但这与 prometheus_inspection_instance_metric 表中 `instance_ip` 为 NOT NULL 的约束矛盾。未返回指标的实例是否应创建空记录？[Consistency, Conflict, Spec §Edge Cases vs data-model]
- [ ] CHK021 plan.md 中 XML 文件路径为 `ruoyi-admin/src/main/resources/mapper/inspection/`，但实际代码中 XML 文件放在 `ruoyi-inspection/src/main/resources/mapper/inspection/`。需求文档中的路径约定是否已统一？[Consistency, plan.md vs 实际实现]
- [ ] CHK022 spec 中提到"下载文件名包含中文字符时需正确编码（RFC 5987）"，api.md 的 download 响应头中同时使用 `filename*=utf-8''` 和 `download-filename` 两个头。这两个头的使用场景和浏览器兼容性需求是否明确？[Consistency, Spec §Edge Cases vs api.md]

## Acceptance Criteria Quality（验收标准质量）

- [ ] CHK023 US1 场景 2"指标采集和 AI 生成全部完成"——能否定义一个可度量的时间阈值？如"30 秒内"或"状态从 pending 变为 success" [Measurability, Spec US1]
- [ ] CHK024 US2 场景 2"浏览器下载一个 Markdown 文件"——是否需要定义文件内容的验证标准？如"文件内容与报告中 markdownContent 字段完全一致" [Measurability, Spec US2]
- [ ] CHK025 US4 场景 2"超过保留月数的报告被自动清理"——是否定义了清理操作的执行时机？是报告生成成功后立即清理还是批量定时清理？[Measurability, Clarity, Spec US4]
- [ ] CHK026 SC-003"重试成功率不低于 95%"的统计周期和统计口径是否定义？是基于单次会话还是长期统计？[Measurability, SC-003]

## Scenario Coverage（场景覆盖）

- [ ] CHK027 是否定义了 Prometheus 返回部分实例数据（某些 instance 有 CPU 但无内存数据）时的需求？[Coverage, Exception Flow]
- [ ] CHK028 是否定义了百炼 AI 返回的 Markdown 内容包含 HTML 标签或恶意脚本时的安全过滤需求？[Coverage, Security, Gap]
- [ ] CHK029 是否定义了报告处于 `pending` 状态时，用户尝试删除该报告的需求？应允许还是拒绝？[Coverage, Edge Case, Gap]
- [ ] CHK030 是否定义了同一用户在短时间内多次点击"立即巡检"按钮（绕过 @RepeatSubmit）的需求？后端是否有防抖/限流策略？[Coverage, Gap, FR-001]
- [ ] CHK031 是否定义了 Prometheus endpoint 配置为无效地址（如非 HTTP 协议、不可达 IP）时的错误信息展示需求？[Coverage, Exception Flow]
- [ ] CHK032 是否定义了多租户场景下的字典配置隔离需求？`prometheus_inspection` 字典数据是否需要按租户区分不同的 Prometheus 地址和 API Key？[Coverage, Multi-tenant, Gap]

## API Contract Quality（API 契约质量）

- [ ] CHK033 api.md 中 delete 接口的请求体格式为 `{ "ids": [1,2,3] }`，但这与 RuoYi 框架的 `DELETE /{ids}` 路径参数惯例不一致。是否需要在契约中说明选择请求体的原因？[Clarity, api.md §5]
- [ ] CHK034 api.md 中 generate 接口响应的 `msg` 为"巡检任务已提交，请稍后查看报告"——这个文案是否应定义为 i18n 可配置？[Clarity, api.md §3]
- [ ] CHK035 api.md 中 list 接口的 `reportDate` 参数（精确日期）与 `beginTime/endTime` 参数（日期范围）是否互斥？同时传入时的优先级是否定义？[Clarity, Consistency, api.md §1]
- [ ] CHK036 api.md 中 SSE 推送消息格式 `inspection_status` 是否与项目现有 SSE 消息类型冲突？命名空间是否需要加上模块前缀？[Clarity, Consistency, api.md §SSE]

## Non-Functional Requirements（非功能性需求）

- [ ] CHK037 是否定义了 Prometheus API 调用的超时时间（目前代码中为 30 秒）？是否需要作为可配置项？[Completeness, Non-Functional]
- [ ] CHK038 是否定义了 markdown_content 字段的大小上限？百炼 AI 返回的报告长度是否有上限约束？[Completeness, Non-Functional, data-model LONGTEXT]
- [ ] CHK039 是否定义了报告列表页面的默认排序方式？api.md 默认 `createTime desc`，但 spec FR-004 只说"支持排序"未指定默认值 [Completeness, Spec FR-004 vs api.md]
- [ ] CHK040 是否定义了 @Async 线程池的配置需求？如核心线程数、队列大小、拒绝策略？[Completeness, Non-Functional, Gap]

## Dependencies & Assumptions（依赖与假设）

- [ ] CHK041 spec 假设"Prometheus 服务已部署并正常运行"——是否需要在功能中增加 Prometheus 连通性检测/健康检查的需求？[Assumption, Spec §Assumptions]
- [ ] CHK042 spec 假设"阿里云百炼智能体应用已创建"——是否需要定义 API Key 过期或配额耗尽时的处理需求？[Assumption, Dependency, Gap]
- [ ] CHK043 spec 假设"前端页面遵循 Element Plus 组件风格"——是否需要定义巡检报告页面与现有系统页面风格一致性的具体标准？[Assumption, Spec §Assumptions]
- [ ] CHK044 plan.md 中依赖 SnailJob 1.9.0 的 @JobExecutor——是否需要在 SnailJob 管理界面中手动注册任务的步骤文档？[Dependency, plan.md §D5]

## Ambiguities & Conflicts（歧义与冲突）

- [ ] CHK045 spec §Edge Cases 提到"同一时刻多次触发巡检时，每条报告独立生成"——但前端有 @RepeatSubmit 防抖。这是否意味着后端也应允许多次并发触发？还是 spec 允许但前端限制？[Ambiguity, Spec §Edge Cases vs FR-001]
- [ ] CHK046 data-model 中 `tenant_id` 默认值为 0，但 RuoYi 框架的多租户机制通常由拦截器自动填充。默认值 0 是否会导致未登录场景下的数据错误？[Ambiguity, data-model]
- [ ] CHK047 需求文档 §2.2 中磁盘查询的 `fstype!~"tmpfs|fuse.*"` 正则表达式使用了 `|` 分隔，但 PromQL 中 `!~` 操作符使用的是 RE2 正则语法。`tmpfs|fuse.*` 是否匹配预期？[Ambiguity, 需求文档 §2.2]

## Notes

- 标记完成：`[x]`
- 发现问题时在条目后追加注释
- 优先处理标记为 `[Gap]` 和 `[Conflict]` 的条目
- CHK020（空实例记录 vs NOT NULL 约束）和 CHK016（重试时间 vs 60秒目标）为高优先级冲突
