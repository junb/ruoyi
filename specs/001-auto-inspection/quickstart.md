# Quickstart: 服务器自动巡检

**Feature**: `001-auto-inspection` | **Date**: 2026-04-05

## 前置条件

1. MySQL 数据库 `ry-vue` 可访问
2. Redis 服务运行中
3. Prometheus 服务运行中，HTTP API 可访问（默认 `http://192.168.2.161:9090`）
4. 服务器节点已安装 Node Exporter 并被 Prometheus 采集
5. 阿里云百炼智能体应用已创建，API Key 已获取

## 快速验证步骤

### 1. 数据库初始化

执行建表 SQL（在 `RuoYi-Vue-Plus/script/sql/` 下新增）：

- `prometheus_inspection_report` — 巡检报告表
- `prometheus_inspection_instance_metric` — 实例指标表

插入字典数据（字典类型 `prometheus_inspection`）：
- `PROMETHEUS_ENDPOINT` = `http://192.168.2.161:9090`
- `BAILIAN_API_KEY` = 你的百炼 API Key
- `BAILIAN_APP_ID` = `e579a36217c64c219ca64d1099c98299`
- `RETENTION_MONTHS` = `12`

### 2. 菜单与权限配置

在系统管理 → 菜单管理中添加：

- 一级菜单：**巡检管理**（路由 `/inspection`，图标 `monitor`）
- 二级菜单：**巡检报告**（路由 `/inspection/report`，组件 `inspection/report/index`）
- 按钮权限：
  - `inspection:report:query` — 查询
  - `inspection:report:create` — 创建/重试
  - `inspection:report:download` — 下载
  - `inspection:report:delete` — 删除

### 3. 后端启动

```bash
cd RuoYi-Vue-Plus
mvn spring-boot:run -pl ruoyi-admin -P dev
```

验证：访问 `http://localhost:8080/inspection/report/list` 返回空列表（需登录获取 Token）。

### 4. 前端启动

```bash
cd plus-ui
npm run dev
```

验证：登录后侧边栏出现"巡检管理"菜单，点击进入巡检报告页面。

### 5. 功能验证

1. 点击"立即巡检"按钮 → 列表出现 `pending` 状态记录
2. 等待报告生成完成 → 状态变为 `success`，通过 SSE 实时更新
3. 点击"查看详情" → 查看 Markdown 报告和实例指标
4. 点击"下载" → 下载 `.md` 文件，文件名支持中文
5. 对失败报告点击"重试" → 状态回到 `pending`，重新生成

### 6. 定时任务配置（SnailJob）

在 SnailJob 管理后台配置：

- 任务名称：`inspectionScheduledJob`
- 任务处理器：`inspectionScheduledJob`
- Cron 表达式：`0 0 9 1 * ?`（每月 1 号 9:00）
- 执行器类型：集群

## 关键文件清单

### 后端（新增模块 `ruoyi-modules/ruoyi-inspection`）

```
ruoyi-inspection/
├── pom.xml
└── src/main/java/org/dromara/inspection/
    ├── controller/
    │   └── InspectionReportController.java
    ├── domain/
    │   ├── InspectionReport.java
    │   ├── InspectionInstanceMetric.java
    │   ├── bo/
    │   │   ├── InspectionReportBo.java
    │   │   └── InspectionInstanceMetricBo.java
    │   └── vo/
    │       ├── InspectionReportVo.java
    │       └── InspectionInstanceMetricVo.java
    ├── mapper/
    │   ├── InspectionReportMapper.java
    │   └── InspectionInstanceMetricMapper.java
    ├── service/
    │   ├── IInspectionReportService.java
    │   └── impl/
    │       ├── InspectionReportServiceImpl.java
    │       ├── PrometheusCollectorService.java
    │       └── BailianReportService.java
    └── job/
        └── InspectionScheduledJob.java
```

### 前端（`plus-ui/src/`）

```
api/inspection/report/
├── index.ts
└── types.ts

views/inspection/report/
├── index.vue
├── detail.vue
└── components/
    └── ReportDetail.vue
```
