# Research: 告警飞书转发

**Date**: 2026-04-06
**Feature**: 002-alert-feishu-forward

## R1: Alertmanager v2 Webhook Payload 格式

**Decision**: 使用 Alertmanager v2 API 的标准 Webhook payload 格式进行解析。

**Payload 结构**：

```json
{
  "receiver": "webhook",
  "status": "firing",
  "alerts": [
    {
      "status": "firing",
      "labels": { "alertname": "HighCPU", "severity": "critical", "instance": "10.0.0.1:9090" },
      "annotations": { "summary": "CPU过高", "description": "CPU已达95%" },
      "startsAt": "2026-04-06T08:30:00.123456789Z",
      "endsAt": "0001-01-01T00:00:00Z",
      "generatorURL": "http://prometheus:9090/...",
      "fingerprint": "a1b2c3d4e5f6a1b2"
    }
  ],
  "groupLabels": {},
  "commonLabels": {},
  "commonAnnotations": {},
  "externalURL": "http://alertmanager:9093",
  "version": "4",
  "groupKey": "{alertname=\"HighCPU\"}",
  "truncatedAlerts": 0
}
```

**关键字段说明**：

| 字段 | 说明 |
|------|------|
| `alerts[].status` | `"firing"` 或 `"resolved"`（webhook 中不会出现 `suppressed`） |
| `alerts[].fingerprint` | 16 位十六进制字符串，基于 labels 计算，同一告警 firing→resolved 指纹不变 |
| `alerts[].labels` | 告警标签（参与 fingerprint 计算），含 alertname、severity、instance 等 |
| `alerts[].annotations` | 告警注解（不参与 fingerprint），含 summary、description |
| `alerts[].startsAt` | RFC3339 格式，告警开始时间 |
| `alerts[].endsAt` | 未结束时为零值 `0001-01-01T00:00:00Z` |

**Rationale**: Alertmanager webhook payload 是标准格式，版本固定为 "4"。fingerprint 可用于关联同一告警的 firing→resolved 状态变化。suppressed 告警不会发送到 webhook，所以系统只需处理 firing 和 resolved 两种状态。

**Alternatives considered**:
- 使用 Alertmanager v1 API（已弃用，不推荐）
- 直接对接 Prometheus Alerting Rules（需要自己实现分组/路由逻辑，过于复杂）

## R2: 飞书自定义机器人消息卡片格式

**Decision**: 使用飞书卡片 2.0 格式，通过 `msg_type: "interactive"` 发送消息卡片。告警用红色主题，恢复用绿色主题。

**请求体结构**：

```json
{
  "msg_type": "interactive",
  "card": {
    "schema": "2.0",
    "config": { "update_multi": true },
    "header": {
      "title": { "tag": "plain_text", "content": "告警通知" },
      "template": "red"
    },
    "body": {
      "direction": "vertical",
      "elements": [
        { "tag": "markdown", "content": "**告警名称：** HighCPU\n**实例：** 10.0.0.1" }
      ]
    }
  }
}
```

**颜色映射**：

| 告警状态 | header.template | 含义 |
|----------|----------------|------|
| firing (critical) | `red` | 严重告警 |
| firing (warning) | `orange` | 警告 |
| resolved | `green` | 已恢复 |

**Markdown 支持的语法**：粗体 `**text**`、斜体 `*text*`、链接 `[text](url)`、彩色文本 `<font color='red'>text</font>`、分割线 `---`、无序列表 `- item`。

**频率限制**：单机器人 100 次/分钟，5 次/秒。请求体不超过 20KB。

**Rationale**: 卡片 2.0 格式是最新的飞书消息卡片标准，支持丰富的 markdown 排版和颜色控制。使用 `update_multi: true` 配置允许多端同步更新。纯展示卡片不含交互按钮，符合需求。

**Alternatives considered**:
- 飞书开放平台 API（需要 App ID/Secret，复杂度高，自定义机器人已满足需求）
- 纯文本消息（`msg_type: "text"`）（信息展示能力弱，不推荐）

## R3: 异步处理与多租户处理

**Decision**: Webhook 接口使用 `@Async` 异步处理告警。持久化和飞书发送并行执行，互不依赖。多租户通过字典配置默认租户 ID。

**处理流程**：

```
Alertmanager → POST /webhook/alert/{token}
  → 验证 token（从字典配置获取）
  → 立即返回 200 OK
  → @Async 异步处理：
      → 遍历 alerts 数组
      → 对每条 alert：
          1. 查找已有记录（by fingerprint + tenant_id）
          2. 如存在且新状态为 resolved → 更新记录
          3. 如不存在 → 新建记录
          4. 异步发送飞书通知（失败不影响持久化）
```

**多租户处理**：Webhook 接口无登录态，Service 层从字典配置读取默认租户 ID，手动设置到告警记录中。CRUD 接口走正常 Sa-Token + 租户插件隔离。

**Rationale**: 异步处理确保 Webhook 快速响应（Alertmanager 有超时机制），持久化和飞书发送互不依赖保证数据不丢失。多租户通过配置化处理，避免 Webhook 接口需要认证的复杂度。

**Alternatives considered**:
- 同步处理（阻塞 Alertmanager，可能导致超时重试）
- 使用消息队列（RabbitMQ/Kafka）（过度设计，当前规模不需要）

## R4: 飞书发送失败重试策略

**Decision**: 使用内存重试，最多 3 次，间隔递增 2s/3s/5s。重试失败后标记发送状态为 failed，支持用户手动重试。

**Rationale**: 与项目中 BailianReportService 的重试策略保持一致（相同的 MAX_RETRIES=3 和 RETRY_DELAYS={2000,3000,5000}）。手动重试通过用户在界面点击触发。

## R5: Webhook 接口鉴权方案

**Decision**: 使用 URL 路径变量 token 进行简单鉴权，token 通过字典配置管理。

**URL 格式**：`POST /webhook/alert/{token}`

**验证逻辑**：
1. 从字典配置（`alert_config` 字典类型，key `WEBHOOK_TOKEN`）获取有效 token
2. 比较请求路径中的 token 与配置的 token
3. 匹配则处理，不匹配返回 401

**Rationale**: 简单有效，Alertmanager 直接在 webhook URL 中配置完整 token 即可。无需 OAuth/JWT 等复杂认证。

## R6: 字典配置项

**Decision**: 新增字典类型 `alert_config`，包含以下配置项：

| 字典标签 | 字典键值 | 说明 |
|----------|----------|------|
| 飞书 Webhook 地址 | `FEISHU_WEBHOOK_URL` | 飞书自定义机器人 Webhook URL |
| Webhook Token | `WEBHOOK_TOKEN` | 告警 Webhook 鉴权 token |
| 保留天数 | `RETENTION_DAYS` | 告警记录保留天数（默认 90） |
| 默认租户ID | `DEFAULT_TENANT_ID` | Webhook 接口使用的默认租户 ID |

**Rationale**: 与巡检模块的 `prometheus_inspection` 字典配置模式一致，通过 DictService 获取配置。
