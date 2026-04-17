# Implementation Plan: 磁盘容量预测

**Branch**: `003-disk-prediction` | **Date**: 2026-04-10 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/003-disk-prediction/spec.md`

## Summary

基于 Prometheus 采集的磁盘使用历史数据，通过线性回归预测未来磁盘容量趋势。系统每 6 小时采集磁盘快照存入 `disk_usage_history` 表，后端基于最小二乘法线性回归计算增长趋势、预计满载日期和风险等级。提供独立预测看板页面（含 ECharts 趋势图），并在巡检报告中追加预测摘要。

## Technical Context

**Language/Version**: Java 17（Spring Boot 3.5.12）+ TypeScript ~5.9.3（Vue 3.5.30）
**Primary Dependencies**: MyBatis-Plus 3.5.16, Sa-Token 1.44.0, ECharts 6.0.0（已引入）, SnailJob
**Storage**: MySQL（`ry-vue` 库），Redis（Spring Cache/Redisson）
**Testing**: 手动测试 + SnailJob 任务日志验证
**Target Platform**: Web 管理后台
**Project Type**: Web application（前后端分离）
**Performance Goals**: 预测计算 < 1s，页面加载 < 3s
**Constraints**: 线性回归纯 Java 实现，不引入第三方数学库
**Scale/Scope**: 100 台以内服务器，每台 3-5 个挂载点

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 原则 | 合规 | 说明 |
|---|---|---|
| I. 分层架构 | PASS | Controller → Service → Mapper 标准三层 |
| II. 统一响应与异常处理 | PASS | 使用 R<T> 包装，ServiceException 抛异常 |
| III. 多租户数据隔离 | PASS | 新表含 tenant_id，实体继承 TenantEntity |
| IV. 注解驱动开发 | PASS | @AutoMapper 做 VO 转换，@SaCheckPermission 权限 |
| V. 安全与权限 | PASS | 所有接口 @SaCheckPermission 注解 |
| VI. 缓存策略 | PASS | 预测结果不缓存（实时计算），字典配置走已有缓存 |
| VII. 前后端协作规范 | PASS | API 模块 + types.ts 与后端一一对应 |

**无违规项**，Complexity Tracking 表无需填写。

## Project Structure

### Documentation (this feature)

```text
specs/003-disk-prediction/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   └── api.md           # Phase 1 output
└── tasks.md             # Phase 2 output (/speckit.tasks)
```

### Source Code (repository root)

```text
RuoYi-Vue-Plus/
├── script/sql/
│   └── disk_prediction.sql                          # 建表 + 字典 + 菜单
├── ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/
│   ├── domain/
│   │   ├── DiskUsageHistory.java                    # 历史实体（extends TenantEntity）
│   │   └── vo/
│   │       ├── DiskPredictionVo.java                # 预测结果 VO
│   │       ├── PredictionPointVo.java               # 预测节点 VO
│   │       └── DiskUsageHistoryVo.java              # 历史查询 VO
│   ├── mapper/
│   │   └── DiskUsageHistoryMapper.java              # Mapper
│   ├── service/
│   │   ├── IDiskUsageHistoryService.java            # 历史数据 Service 接口
│   │   ├── IDiskPredictionService.java              # 预测 Service 接口
│   │   └── impl/
│   │       ├── DiskUsageHistoryServiceImpl.java     # 历史数据采集+清理
│   │       └── DiskPredictionServiceImpl.java       # 线性回归预测
│   ├── controller/
│   │   └── DiskPredictionController.java            # 预测 API
│   └── job/
│       └── DiskUsageCollectJob.java                 # 定时采集任务

plus-ui/src/
├── api/inspection/prediction/
│   ├── index.ts                                     # API 函数
│   └── types.ts                                     # 类型定义
└── views/inspection/prediction/
    └── index.vue                                    # 预测看板页面
```

**Structure Decision**: 所有后端代码放在现有 `ruoyi-inspection` 模块，前端在 `inspection/prediction/` 目录下新增页面。

## Complexity Tracking

无违规项。
