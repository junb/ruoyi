# Data Model: 告警飞书转发

**Date**: 2026-04-06
**Feature**: 002-alert-feishu-forward

## 实体关系

```
alert_record（告警记录） — 独立表，与其他表无外键关联
```

## 数据表

### alert_record（告警记录表）

| 字段名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | bigint | PK | 雪花ID（ASSIGN_ID） |
| fingerprint | varchar(64) | 是 | 告警指纹（来自 Alertmanager，16位十六进制） |
| alert_name | varchar(255) | 是 | 告警名称（labels.alertname） |
| status | varchar(20) | 是 | 告警状态：firing / resolved（suppressed 不会出现在 webhook 中） |
| severity | varchar(20) | 否 | 严重级别（labels.severity）：critical / warning / info |
| instance | varchar(255) | 否 | 实例地址（labels.instance） |
| labels | text | 否 | 标签集合 JSON（完整 labels 对象） |
| summary | varchar(1000) | 否 | 摘要（annotations.summary） |
| description | text | 否 | 描述（annotations.description） |
| raw_payload | text | 否 | 原始单条 alert JSON（非整个 Webhook message，避免批量推送时冗余） |
| alert_source | varchar(50) | 否 | 告警来源（labels.datasource 或默认 prometheus） |
| starts_at | datetime | 是 | 告警开始时间（alerts[].startsAt） |
| ends_at | datetime | 否 | 告警结束时间（resolved 时更新） |
| feishu_send_status | varchar(20) | 是 | 飞书发送状态：pending / success / failed |
| feishu_response | text | 否 | 飞书 API 响应结果 |
| feishu_send_time | datetime | 否 | 飞书发送成功时间 |
| tenant_id | varchar(20) | 是 | 租户 ID |
| create_dept | bigint | 否 | 创建部门 |
| create_by | bigint | 否 | 创建者 |
| create_time | datetime | 否 | 创建时间 |
| update_by | bigint | 否 | 更新者 |
| update_time | datetime | 否 | 更新时间 |
| del_flag | smallint | 否 | 删除标志（0 正常，1 删除） |

**索引**：

| 索引名 | 类型 | 字段 | 说明 |
|--------|------|------|------|
| idx_fingerprint | UNIQUE | fingerprint, tenant_id | 同一租户内 fingerprint 唯一，用于 firing→resolved 关联 |
| idx_status | NORMAL | status | 按状态查询 |
| idx_severity | NORMAL | severity | 按严重级别查询 |
| idx_create_time | NORMAL | create_time | 按时间范围查询和过期清理 |

**状态流转**：

```
firing ──→ resolved（通过 fingerprint 匹配更新已有记录）
  │
  └── feishu_send_status: pending → success / failed
```

**约束规则**：
- 同一 fingerprint + tenant_id 唯一，firing→resolved 状态变化更新同一条记录
- **resolved 后再 firing 策略**：当已 resolved 的记录收到相同 fingerprint 的 firing 告警时，将已有记录的 status 更新回 firing，重置 ends_at 为 NULL，保留原始 starts_at，更新 raw_payload 和 labels（告警可能产生新的标签）。这样一条 fingerprint 在数据库中始终只有一条记录，状态随 Alertmanager 推送流转
- `raw_payload` 存储该 alert 对应的单条 alert JSON（非整个 Webhook message），避免批量推送时大量冗余
- `raw_payload` 和 `labels` 使用 TEXT 类型（JSON 可能较大）
- `del_flag` 使用逻辑删除
- `tenant_id` 必填（多租户数据隔离）

## SQL DDL

```sql
CREATE TABLE alert_record (
    id              bigint       NOT NULL COMMENT '主键ID',
    fingerprint     varchar(64)  NOT NULL COMMENT '告警指纹',
    alert_name      varchar(255) NOT NULL COMMENT '告警名称',
    status          varchar(20)  NOT NULL DEFAULT 'firing' COMMENT '告警状态：firing/resolved（注：suppressed不会出现在webhook中）',
    severity        varchar(20)  DEFAULT NULL COMMENT '严重级别：critical/warning/info',
    instance        varchar(255) DEFAULT NULL COMMENT '实例地址',
    labels          text         DEFAULT NULL COMMENT '标签集合JSON',
    summary         varchar(1000) DEFAULT NULL COMMENT '告警摘要',
    description     text         DEFAULT NULL COMMENT '告警描述',
    raw_payload     text         DEFAULT NULL COMMENT '原始单条alert JSON（非整个Webhook message）',
    alert_source    varchar(50)  DEFAULT 'prometheus' COMMENT '告警来源',
    starts_at       datetime     NOT NULL COMMENT '告警开始时间',
    ends_at         datetime     DEFAULT NULL COMMENT '告警结束时间',
    feishu_send_status varchar(20) NOT NULL DEFAULT 'pending' COMMENT '飞书发送状态：pending/success/failed',
    feishu_response text         DEFAULT NULL COMMENT '飞书API响应结果',
    feishu_send_time datetime    DEFAULT NULL COMMENT '飞书发送成功时间',
    tenant_id       varchar(20)  DEFAULT '000000' COMMENT '租户ID',
    create_dept     bigint       DEFAULT NULL COMMENT '创建部门',
    create_by       bigint       DEFAULT NULL COMMENT '创建者',
    create_time     datetime     DEFAULT NULL COMMENT '创建时间',
    update_by       bigint       DEFAULT NULL COMMENT '更新者',
    update_time     datetime     DEFAULT NULL COMMENT '更新时间',
    del_flag        smallint     DEFAULT 0 COMMENT '删除标志',
    PRIMARY KEY (id),
    UNIQUE KEY idx_fingerprint (fingerprint, tenant_id),
    KEY idx_status (status),
    KEY idx_severity (severity),
    KEY idx_create_time (create_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='告警记录表';
```
