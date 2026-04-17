# Quickstart: 磁盘容量预测

## 前置条件

- Prometheus 已部署，正常采集 `node_filesystem_*` 指标
- 字典 `prometheus_inspection` 中已配置 `PROMETHEUS_ENDPOINT`
- 数据库中已执行建表 SQL（`disk_usage_history`）

## 使用步骤

### 1. 配置定时采集

在 SnailJob 管理台创建任务：
- 任务名称：`diskUsageCollectJob`
- CRON 表达式：`0 0 0/6 * * ?`（每 6 小时）
- 任务处理器：`diskUsageCollectJob`

### 2. 等待数据积累

首次采集后即可在预测页面看到数据，但需要至少 10 个数据点（约 2.5 天）才能生成预测。

### 3. 查看预测结果

巡检管理 → 资源预测 菜单：
- 概览表格查看所有磁盘风险等级
- 点击行展开详情抽屉查看趋势图

### 4. 巡检报告集成

磁盘预测摘要会在巡检报告生成时自动追加到 AI prompt 中，无需额外操作。
