# Data Model: 服务器自动巡检

**Feature**: `001-auto-inspection` | **Date**: 2026-04-05

## 实体关系

```
InspectionReport (1) ────< (N) InspectionInstanceMetric
```

## 表结构

### 1. 巡检报告表 `prometheus_inspection_report`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | BIGINT PK | 是 | 雪花ID主键 |
| report_name | VARCHAR(255) | 是 | 报告名称，格式：`{年}年{月}月服务器巡检报告_{YYYYMMDD_HHmm}` |
| report_date | DATE | 是 | 报告日期 |
| status | VARCHAR(20) | 是 | 状态：`pending`/`success`/`failed`，默认 `pending` |
| instance_count | INT | 否 | 本次巡检的服务器数量 |
| generation_duration | INT | 否 | 生成耗时（秒） |
| markdown_content | LONGTEXT | 否 | AI 生成的 Markdown 报告 |
| prometheus_endpoint | VARCHAR(500) | 否 | 本次使用的 Prometheus 地址 |
| trigger_type | VARCHAR(20) | 是 | 触发方式：`manual`/`scheduled` |
| error_message | TEXT | 否 | 失败时的错误信息 |
| tenant_id | BIGINT | 是 | 多租户字段（继承 TenantEntity） |
| create_dept | BIGINT | 否 | 创建部门（继承 BaseEntity） |
| create_by | BIGINT | 否 | 创建人（继承 BaseEntity） |
| create_time | DATETIME | 是 | 创建时间（继承 BaseEntity） |
| update_by | BIGINT | 否 | 更新人（继承 BaseEntity） |
| update_time | DATETIME | 是 | 更新时间（继承 BaseEntity） |
| del_flag | CHAR(1) | 是 | 逻辑删除标志（继承 BaseEntity） |

**索引**：`idx_report_date(report_date)`、`idx_status(status)`、`idx_create_time(create_time)`

### 2. 实例指标表 `prometheus_inspection_instance_metric`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | BIGINT PK | 是 | 雪花ID主键 |
| report_id | BIGINT FK | 是 | 关联报告 ID，ON DELETE CASCADE |
| instance_ip | VARCHAR(100) | 是 | 服务器 IP |
| instance_name | VARCHAR(255) | 否 | 实例名称 |
| cpu_usage_percent | DECIMAL(5,2) | 否 | CPU 使用率（%） |
| memory_total_bytes | BIGINT | 否 | 内存总量（字节） |
| memory_used_bytes | BIGINT | 否 | 已用内存（字节） |
| memory_available_bytes | BIGINT | 否 | 可用内存（字节） |
| memory_usage_percent | DECIMAL(5,2) | 否 | 内存使用率（%） |
| disk_partitions | JSON | 否 | 磁盘分区列表（结构见下文） |
| collected_time | DATETIME | 是 | 指标采集时间 |
| tenant_id | BIGINT | 是 | 多租户字段 |
| create_by | BIGINT | 否 | 创建人 |
| create_time | DATETIME | 是 | 创建时间 |
| update_by | BIGINT | 否 | 更新人 |
| update_time | DATETIME | 是 | 更新时间 |
| del_flag | CHAR(1) | 是 | 逻辑删除标志 |

**索引**：`idx_report_id(report_id)`、`idx_instance_ip(instance_ip)`

### disk_partitions JSON 结构

```json
[
  {
    "mount_point": "/",
    "device": "/dev/sda1",
    "total_bytes": 1000000000,
    "used_bytes": 500000000,
    "available_bytes": 500000000,
    "usage_percent": 50.0
  }
]
```

## 状态流转

```
触发 → [pending] → [success]
                  → [failed] → 重试 → [pending] → ...
```

- **pending**：正在生成中（前端灰色）
- **success**：生成完成（前端绿色）
- **failed**：生成失败，保留 error_message（前端红色）

关键约束：状态变更必须立即持久化，不能等整体业务事务结束。

## 字典配置

字典类型：`prometheus_inspection`

| 配置键 | 默认值 | 说明 |
|--------|--------|------|
| PROMETHEUS_ENDPOINT | http://192.168.2.161:9090 | Prometheus API 地址 |
| LABEL_FILTERS | （空） | 实例标签过滤（可选） |
| RETENTION_MONTHS | 12 | 报告保留月数 |
| BAILIAN_API_KEY | （需配置） | 百炼 API Key |
| BAILIAN_APP_ID | e579a36217c64c219ca64d1099c98299 | 百炼智能体应用 ID |
