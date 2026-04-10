# API Contracts: 告警飞书转发

**Date**: 2026-04-06
**Feature**: 002-alert-feishu-forward

## 1. Webhook 接口（Alertmanager 调用，无登录态）

### POST /webhook/alert/{token}

接收 Alertmanager Webhook 推送的告警数据。

**认证**：URL 路径参数 token，与字典配置 `alert_config.WEBHOOK_TOKEN` 匹配。

**Request**：

```
POST /webhook/alert/your-secret-token
Content-Type: application/json
```

```json
{
  "receiver": "webhook",
  "status": "firing",
  "alerts": [
    {
      "status": "firing",
      "labels": { "alertname": "HighCPU", "severity": "critical", "instance": "10.0.0.1:9090" },
      "annotations": { "summary": "CPU过高", "description": "CPU已达95%" },
      "startsAt": "2026-04-06T08:30:00.123Z",
      "endsAt": "0001-01-01T00:00:00Z",
      "generatorURL": "http://prometheus:9090/...",
      "fingerprint": "a1b2c3d4e5f6a1b2"
    }
  ],
  "groupLabels": {},
  "commonLabels": {},
  "commonAnnotations": {},
  "externalURL": "http://alertmanager:9093",
  "version": "4"
}
```

**Response（成功）**：

```
HTTP 200 OK
```

```json
{ "code": 200, "msg": "操作成功" }
```

**Response（token 无效）**：

```
HTTP 401 Unauthorized
```

```json
{ "code": 401, "msg": "无效的 Webhook Token" }
```

**Response（请求体为空或格式异常）**：

```
HTTP 400 Bad Request
```

```json
{ "code": 400, "msg": "告警数据为空或格式异常" }
```

**处理逻辑**：
1. 验证 token
2. 验证请求体非空且包含 alerts 数组
3. 立即返回 200
4. 异步处理每条 alert：持久化 + 发送飞书

---

## 2. 告警记录 CRUD 接口（需登录态）

### GET /inspection/alarm/list

分页查询告警记录列表。

**权限**：`inspection:alarm:query`

**Request Query Parameters**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNum | int | 否 | 页码（默认 1） |
| pageSize | int | 否 | 每页条数（默认 10） |
| status | string | 否 | 告警状态筛选：firing / resolved |
| severity | string | 否 | 严重级别筛选：critical / warning / info |
| alertName | string | 否 | 告警名称模糊搜索 |
| beginTime | string | 否 | 开始时间（yyyy-MM-dd HH:mm:ss） |
| endTime | string | 否 | 结束时间（yyyy-MM-dd HH:mm:ss） |

**Response**：

```json
{
  "code": 200,
  "msg": "操作成功",
  "rows": [
    {
      "id": "1895123456789",
      "fingerprint": "a1b2c3d4e5f6a1b2",
      "alertName": "HighCPU",
      "status": "firing",
      "severity": "critical",
      "instance": "10.0.0.1:9090",
      "summary": "CPU过高",
      "alertSource": "prometheus",
      "startsAt": "2026-04-06 08:30:00",
      "endsAt": null,
      "feishuSendStatus": "success",
      "feishuSendTime": "2026-04-06 08:30:01",
      "createTime": "2026-04-06 08:30:00"
    }
  ],
  "total": 100
}
```

### GET /inspection/alarm/{id}

查询告警记录详情。

**权限**：`inspection:alarm:query`

**Response**：

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "id": "1895123456789",
    "fingerprint": "a1b2c3d4e5f6a1b2",
    "alertName": "HighCPU",
    "status": "firing",
    "severity": "critical",
    "instance": "10.0.0.1:9090",
    "labels": "{\"alertname\":\"HighCPU\",\"severity\":\"critical\",\"instance\":\"10.0.0.1:9090\"}",
    "summary": "CPU过高",
    "description": "CPU已达95%，超过阈值90%",
    "rawPayload": "{...}",
    "alertSource": "prometheus",
    "startsAt": "2026-04-06 08:30:00",
    "endsAt": null,
    "feishuSendStatus": "success",
    "feishuResponse": "{\"code\":0,\"msg\":\"success\"}",
    "feishuSendTime": "2026-04-06 08:30:01",
    "createTime": "2026-04-06 08:30:00",
    "updateTime": "2026-04-06 08:30:01"
  }
}
```

### DELETE /inspection/alarm/{ids}

删除告警记录（支持批量删除）。

**权限**：`inspection:alarm:remove`

**Request Path**：`ids` 为逗号分隔的 ID 列表。

**Response**：

```json
{ "code": 200, "msg": "操作成功" }
```

### POST /inspection/alarm/retry

重试飞书发送。仅允许对 `feishuSendStatus` 为 `failed` 的记录执行重试，其他状态返回 400 错误。

**权限**：`inspection:alarm:retry`

**Request Body**：

```json
{ "id": "1895123456789" }
```

**Response**：

```json
{ "code": 200, "msg": "操作成功" }
```

---

## 3. 菜单与权限结构

```
巡检管理（2070）
├── 巡检报告（2071）
└── 告警记录（2076）              ← 新增二级菜单
    ├── 告警记录查询（2077）       ← inspection:alarm:query
    ├── 告警记录删除（2078）       ← inspection:alarm:remove
    └── 告警重试（2079）           ← inspection:alarm:retry
```

---

## 4. 字典配置

### 字典类型：alert_config

| 字典标签 | 字典键值 | 默认值 | 说明 |
|----------|----------|--------|------|
| 飞书 Webhook 地址 | FEISHU_WEBHOOK_URL | - | 飞书自定义机器人 URL |
| Webhook Token | WEBHOOK_TOKEN | - | 告警 Webhook 鉴权 token |
| 保留天数 | RETENTION_DAYS | 90 | 告警记录保留天数 |
| 默认租户ID | DEFAULT_TENANT_ID | 000000 | Webhook 使用的租户 ID |
