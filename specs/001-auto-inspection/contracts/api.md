# API Contracts: 服务器自动巡检

**Feature**: `001-auto-inspection` | **Date**: 2026-04-05
**Base Path**: `/inspection/report`

---

## 1. 分页查询报告列表

`GET /inspection/report/list`
权限：`inspection:report:query`

**请求参数（Query）**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| status | String | 否 | 状态筛选：pending/success/failed |
| reportDate | String | 否 | 报告日期（yyyy-MM-dd） |
| beginTime | String | 否 | 开始时间（yyyy-MM-dd） |
| endTime | String | 否 | 结束时间（yyyy-MM-dd） |
| pageNum | Integer | 是 | 页码，默认 1 |
| pageSize | Integer | 是 | 每页条数，默认 10 |
| orderByColumn | String | 否 | 排序字段，默认 createTime |
| isAsc | String | 否 | 排序方向：asc/desc，默认 desc |

**响应**：`TableDataInfo<InspectionReportVo>`

```json
{
  "code": 200,
  "msg": "查询成功",
  "rows": [
    {
      "id": 1234567890,
      "reportName": "2026年4月服务器巡检报告_20260405_0900",
      "reportDate": "2026-04-05",
      "status": "success",
      "instanceCount": 5,
      "generationDuration": 12,
      "triggerType": "manual",
      "createTime": "2026-04-05 09:00:00"
    }
  ],
  "total": 30
}
```

---

## 2. 报告详情

`GET /inspection/report/detail/{id}`
权限：`inspection:report:query`

**路径参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | Long | 是 | 报告 ID |

**响应**：`R<InspectionReportDetailVo>`

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "id": 1234567890,
    "reportName": "2026年4月服务器巡检报告_20260405_0900",
    "reportDate": "2026-04-05",
    "status": "success",
    "instanceCount": 5,
    "generationDuration": 12,
    "markdownContent": "# 服务器资源巡检报告\n...",
    "prometheusEndpoint": "http://192.168.2.161:9090",
    "triggerType": "manual",
    "errorMessage": null,
    "createTime": "2026-04-05 09:00:00",
    "updateTime": "2026-04-05 09:00:12"
  }
}
```

---

## 3. 手动触发巡检

`POST /inspection/report/generate`
权限：`inspection:report:create`

**请求体**：无

**响应**：`R<Void>`

```json
{
  "code": 200,
  "msg": "巡检任务已提交，请稍后查看报告"
}
```

**行为**：异步执行，立即返回。报告创建后状态为 `pending`，通过 SSE 推送状态变更通知。

---

## 4. 下载报告

`GET /inspection/report/download/{id}`
权限：`inspection:report:download`

**路径参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | Long | 是 | 报告 ID |

**响应**：文件流（`Content-Type: application/octet-stream`）

- `Content-Disposition: attachment; filename*=utf-8''<percentEncodedName>.md`
- `download-filename: <percentEncodedName>.md`

---

## 5. 批量删除报告

`DELETE /inspection/report/delete`
权限：`inspection:report:delete`

**请求体**：

```json
{
  "ids": [1234567890, 1234567891, 1234567892]
}
```

**响应**：`R<Void>`

```json
{
  "code": 200,
  "msg": "删除成功"
}
```

**行为**：级联删除关联的实例指标数据。

---

## 6. 获取实例指标列表

`GET /inspection/report/instances/{id}`
权限：`inspection:report:query`

**路径参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | Long | 是 | 报告 ID |

**响应**：`R<List<InspectionInstanceMetricVo>>`

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": [
    {
      "id": 1234567891,
      "instanceIp": "192.168.1.10",
      "instanceName": null,
      "cpuUsagePercent": 18.32,
      "memoryTotalBytes": 34359738368,
      "memoryUsedBytes": 15735674880,
      "memoryAvailableBytes": 18624063488,
      "memoryUsagePercent": 45.84,
      "diskPartitions": [
        {
          "mountPoint": "/",
          "device": "/dev/vda1",
          "totalBytes": 53687091200,
          "usedBytes": 13824925696,
          "availableBytes": 39862165504,
          "usagePercent": 25.76
        }
      ],
      "collectedTime": "2026-04-05 09:00:05"
    }
  ]
}
```

---

## 7. 重试失败报告

`POST /inspection/report/retry`
权限：`inspection:report:create`

**请求体**：

```json
{
  "reportId": 1234567890
}
```

**响应**：`R<Void>`

```json
{
  "code": 200,
  "msg": "重试任务已提交"
}
```

**行为**：将报告状态重置为 `pending`，清除旧的错误信息和指标数据，重新执行完整的采集和生成流程。通过 SSE 推送状态变更。

---

## SSE 推送消息格式

报告状态变更时，通过项目已有的 SSE 通道推送消息：

```json
{
  "type": "inspection_status",
  "reportId": 1234567890,
  "status": "success",
  "message": "巡检报告生成完成"
}
```
