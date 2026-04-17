# Research: 磁盘容量预测

## 1. 线性回归算法选型

**Decision**: 纯 Java 实现最小二乘法线性回归，不引入第三方库

**Rationale**: 磁盘容量预测只需要简单的 y = kx + b 拟合，计算量极小（通常 < 500 个数据点），手写几十行代码即可完成。引入 Apache Commons Math 等库为单个方法增加不必要的依赖。

**Alternatives considered**:
- Apache Commons Math OLSMultipleLinearRegression：功能强大但过重
- PromQL `predict_linear()`：依赖 Prometheus 数据保留时长，不适合长期预测

## 2. 历史数据采集方式

**Decision**: 新建 SnailJob 定时任务 `diskUsageCollectJob`，复用现有 `PrometheusCollectorService.collectDiskPartitions()`

**Rationale**: 已有成熟的 Prometheus 采集代码，新表只存原始快照，不需要修改现有逻辑。采集频率 6h/次，一年约 4.4 万条记录，MySQL 完全胜任。

**Alternatives considered**:
- 复用巡检任务的采集结果：巡检报告的指标是 JSON 存在 `disk_partitions` 字段中，解析困难且与报告耦合
- 用 Prometheus range_query 拉历史：受限于 Prometheus 保留时长

## 3. 前端趋势图

**Decision**: 使用 ECharts（已在 package.json 中，版本 6.0.0）

**Rationale**: 项目已引入 ECharts，无需新增依赖。ECharts 的折线图支持实际数据线 + 预测虚线 + 标注点，完全满足需求。

**Alternatives considered**:
- VxeTable 内置图表：功能较弱
- 自绘 Canvas：开发成本高

## 4. 巡检报告集成方式

**Decision**: 在 BailianReportService 的 prompt 中追加磁盘预测摘要文本

**Rationale**: 百炼 AI 生成报告时，prompt 中追加一段磁盘预测的文本摘要即可，无需修改 AI 调用逻辑。格式如："磁盘预测：192.168.2.161 /data 当前 72%，日均增长 50MB，预计 45 天后满载"。

## 5. 审计字段策略

**Decision**: 新表 `disk_usage_history` 继承 `TenantEntity`，包含完整审计字段（tenant_id, create_dept, create_by, create_time, update_by, update_time, del_flag）

**Rationale**: 遵循项目 constitution 的"多租户数据隔离"和"新增业务模块标准流程"原则，所有表必须包含审计字段。定时任务无登录态时，租户ID 使用字典配置的默认值（与 alert 模块一致）。
