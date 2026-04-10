# 磁盘容量预测功能设计

## 概述

基于 Prometheus 采集的磁盘使用历史数据，通过线性回归预测未来磁盘容量趋势，计算预计满载日期，帮助运维人员提前发现磁盘空间不足的风险。

功能包含：独立的资源预测看板页面 + 巡检报告中的磁盘预测摘要。

## 背景

- Prometheus 默认保留数据时间短（通常 15-30 天），不适合做长期预测
- 磁盘容量增长近似线性，适合线性回归拟合
- 已有 `PrometheusCollectorService` 采集磁盘即时数据，可复用

## 数据模型

新增 `disk_usage_history` 表，存储定时采集的磁盘快照：

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT | 主键 |
| instance_ip | VARCHAR(100) | 服务器 IP |
| mount_point | VARCHAR(255) | 挂载点 |
| total_bytes | BIGINT | 总容量 |
| used_bytes | BIGINT | 已用容量 |
| usage_percent | DECIMAL(5,2) | 使用率% |
| collected_time | DATETIME | 采集时间 |

索引：`(instance_ip, mount_point, collected_time)`

审计字段（tenant_id, create_by, create_time 等）后续补充。

数据量估算：10 台服务器 × 3 挂载点 × 4 次/天 × 365 天 = ~43,800 条/年。

## 定时采集

- **任务名**：`diskUsageCollectJob`（SnailJob）
- **频率**：每 6 小时
- **逻辑**：
  1. 复用 `PrometheusCollectorService.collectDiskPartitions()` 获取磁盘快照
  2. 遍历结果，构建 `DiskUsageHistory` 实体批量插入
  3. 自动清理超过保留天数的历史数据（字典配置，默认 365 天）

## 预测算法

新建 `DiskPredictionService`，基于最小二乘法线性回归。

### 输入

- instance_ip
- mount_point
- 历史数据序列（默认取全部）

### 计算步骤

1. 从 `disk_usage_history` 查询 `(collected_time, used_bytes)` 序列
2. 时间转相对天数（x轴），已用字节数（y轴），做最小二乘法线性回归
3. 得到斜率 `k`（每天增长字节数）和截距 `b`
4. 用回归方程计算：
   - 未来 7/15/30/60/90 天的预测使用量和使用率
   - 预计满载日期：`total_bytes = k * x + b` → `x = (total - b) / k`（k > 0 时）
5. 附加指标：日均增长量（MB/天）、剩余天数、当前增长率

### 边界处理

- 数据点 < 10 个：返回"数据不足，无法预测"
- 斜率 k ≤ 0：不计算满载日期，remainingDays = null
- 已使用 > 85% 且剩余 < 30 天：标记为"高风险"

### 输出 VO

```java
DiskPredictionVo {
    instanceIp;          // 实例IP
    mountPoint;          // 挂载点
    totalBytes;          // 总容量
    currentUsedBytes;    // 当前已用
    currentUsagePercent; // 当前使用率
    dailyGrowthBytes;    // 日均增长(字节)
    dailyGrowthMB;       // 日均增长(MB)
    estimatedFullDate;   // 预计满载日期(可能为null)
    remainingDays;       // 剩余天数(可能为null)
    riskLevel;           // high/medium/low
    predictions: [       // 预测节点列表
        { daysAhead: 7,  predictedUsedBytes, predictedUsagePercent },
        { daysAhead: 15, predictedUsedBytes, predictedUsagePercent },
        { daysAhead: 30, predictedUsedBytes, predictedUsagePercent },
        { daysAhead: 60, predictedUsedBytes, predictedUsagePercent },
        { daysAhead: 90, predictedUsedBytes, predictedUsagePercent }
    ]
}
```

## API 设计

| 接口 | 方法 | 说明 |
|---|---|---|
| `/inspection/prediction/list` | GET | 所有实例的预测概览（含风险标记） |
| `/inspection/prediction/detail` | GET | 单个实例+挂载点的详细预测（含预测节点） |
| `/inspection/prediction/history` | GET | 历史数据（用于前端趋势图） |

参数：`instanceIp`、`mountPoint` 筛选。

## 前端页面

### 独立预测页面

巡检管理下新增"资源预测"菜单：

1. **概览表格**：列出所有实例+挂载点的当前使用率、日均增长、预计满载日期、剩余天数、风险等级（红/黄/绿 tag）
2. **详情抽屉**：点击行展开
   - 趋势图：实际历史数据点 + 线性回归拟合线 + 未来预测虚线（ECharts）
   - 预测节点表格：7/15/30/60/90 天的预测值
   - 关键指标卡片：日均增长、剩余天数、预计满载日期

### 巡检报告集成

- `BailianReportService.generateReport()` 的 prompt 中追加磁盘预测摘要
- 格式："磁盘预测：192.168.2.161 /data 分区当前使用率 72%，日均增长 50MB，预计 45 天后满载"

## 技术约束

- 线性回归纯 Java 实现，不引入第三方数学库
- 前端趋势图使用 ECharts
- 定时任务使用 SnailJob（与现有巡检任务一致）
- 新增代码放在 `ruoyi-inspection` 模块
