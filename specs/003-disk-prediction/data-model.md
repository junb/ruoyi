# Data Model: 磁盘容量预测

## 新增表

### disk_usage_history — 磁盘使用历史记录

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| id | BIGINT | YES | — | 主键（雪花ID） |
| instance_ip | VARCHAR(100) | YES | — | 服务器 IP |
| mount_point | VARCHAR(255) | YES | — | 挂载点（如 /data） |
| total_bytes | BIGINT | YES | — | 总容量（字节） |
| used_bytes | BIGINT | YES | — | 已用容量（字节） |
| usage_percent | DECIMAL(5,2) | YES | — | 使用率（%） |
| collected_time | DATETIME | YES | — | 采集时间 |
| tenant_id | VARCHAR(20) | NO | '000000' | 租户ID |
| create_dept | BIGINT | NO | NULL | 创建部门 |
| create_by | BIGINT | NO | NULL | 创建者 |
| create_time | DATETIME | YES | — | 创建时间 |
| update_by | BIGINT | NO | NULL | 更新者 |
| update_time | DATETIME | YES | — | 更新时间 |
| del_flag | CHAR(1) | YES | '0' | 删除标志 |

**索引**：
- PRIMARY KEY (`id`)
- INDEX `idx_instance_mount_time` (`instance_ip`, `mount_point`, `collected_time`)
- INDEX `idx_collected_time` (`collected_time`)

**估算容量**：10 服务器 × 3 挂载点 × 4 次/天 × 365 天 ≈ 43,800 行/年

## 非持久化实体

### DiskPredictionVo — 预测结果（API 返回）

| 字段 | 类型 | 说明 |
|---|---|---|
| instanceIp | String | 实例 IP |
| mountPoint | String | 挂载点 |
| totalBytes | Long | 总容量 |
| currentUsedBytes | Long | 当前已用 |
| currentUsagePercent | BigDecimal | 当前使用率 |
| dailyGrowthBytes | Long | 日均增长（字节） |
| dailyGrowthMB | BigDecimal | 日均增长（MB） |
| estimatedFullDate | Date | 预计满载日期（可 null） |
| remainingDays | Integer | 剩余天数（可 null） |
| riskLevel | String | high / medium / low / insufficient_data |
| dataPoints | Integer | 实际使用的数据点数 |
| predictions | List<PredictionPoint> | 预测节点列表 |

### PredictionPoint — 预测节点

| 字段 | 类型 | 说明 |
|---|---|---|
| daysAhead | Integer | 未来天数（7/15/30/60/90） |
| predictedUsedBytes | Long | 预测已用量 |
| predictedUsagePercent | BigDecimal | 预测使用率 |

### DiskUsageHistoryVo — 历史数据查询结果

| 字段 | 类型 | 说明 |
|---|---|---|
| collectedTime | Date | 采集时间 |
| usedBytes | Long | 已用量 |
| usagePercent | BigDecimal | 使用率 |

## 风险等级规则

| 等级 | 条件 |
|---|---|
| high | 使用率 > 85% 且 剩余天数 < 30 |
| medium | 使用率 > 70% 或 剩余天数 < 90 |
| low | 其他情况 |
| insufficient_data | 数据点 < 10 |
