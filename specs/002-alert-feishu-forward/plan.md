# Implementation Plan: 告警飞书转发

**Branch**: `002-alert-feishu-forward` | **Date**: 2026-04-06 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/002-alert-feishu-forward/spec.md`

## Summary

将 Alertmanager/Loki 告警通过 Webhook 接收后异步转发到飞书群机器人，同时将告警记录持久化到数据库。支持告警记录查询、飞书发送失败重试、定时清理过期记录。基于现有 `ruoyi-inspection` 模块扩展，作为"巡检管理"菜单下的子功能"告警记录"。

## Technical Context

**Language/Version**: Java 17（Spring Boot 3.5.12）
**Primary Dependencies**: MyBatis-Plus 3.5.16, Sa-Token 1.44.0, Hutool HTTP, Jackson, Spring Async
**Storage**: MySQL（`ry-vue` 库），Redis（Spring Cache/Redisson）
**Testing**: Maven Surefire（项目默认 skipTests=true）
**Target Platform**: Linux 服务器（Docker 部署）
**Project Type**: Web 服务（Spring Boot 多模块企业应用）
**Performance Goals**: 告警 5 秒内到达飞书，查询响应 <2 秒（10 万条数据）
**Constraints**: 飞书发送与持久化互不依赖，异步处理，飞书失败不丢失告警
**Scale/Scope**: 中小规模运维团队，单次 Webhook 最多 100 条告警

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 原则 | 状态 | 说明 |
|------|------|------|
| I. 分层架构 | ✅ 通过 | Controller（Webhook + CRUD）→ Service（IAlertRecordService）→ Mapper（AlertRecordMapper），层间职责清晰 |
| II. 统一响应与异常处理 | ✅ 通过 | CRUD 接口返回 `R<T>` / `TableDataInfo<T>`；Webhook 接口返回 200/400（Alertmanager 不识别 `R<T>` 格式） |
| III. 多租户数据隔离 | ⚠️ 需注意 | 告警记录表包含 `tenant_id`，但 Webhook 接口无登录态需特殊处理（见下方说明） |
| IV. 注解驱动开发 | ✅ 通过 | @AutoMapper 转换、@Validated 校验、@TableName 实体映射、雪花ID |
| V. 安全与权限 | ✅ 通过 | CRUD 接口 @SaCheckPermission，Webhook 接口 URL token 鉴权 |
| VI. 缓存策略 | ✅ 通过 | 字典配置走系统缓存，告警列表不缓存（实时性要求高） |
| VII. 前后端协作规范 | ✅ 通过 | 前端 API/Types 与后端 Controller/VO 一一对应 |

**多租户处理说明**：Webhook 接口由 Alertmanager 调用，无登录态，需要：
1. 在安全配置或 `application.yml` 中排除 `/webhook/alert/**` 路径的 Sa-Token 认证拦截和租户插件拦截
2. 在 Service 层手动设置 `tenant_id`（从字典配置 `alert_config.DEFAULT_TENANT_ID` 获取）
3. CRUD 接口走正常 Sa-Token + 租户插件隔离

## Project Structure

### Documentation (this feature)

```text
specs/002-alert-feishu-forward/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── api.md           # API contracts
└── tasks.md             # Phase 2 output (/speckit.tasks)
```

### Source Code (repository root)

在现有 `ruoyi-inspection` 模块中扩展：

```text
RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/java/org/dromara/inspection/
├── domain/
│   ├── AlertRecord.java                           # 告警记录实体
│   ├── bo/
│   │   └── AlertRecordBo.java                     # 查询 BO
│   └── vo/
│       ├── AlertRecordVo.java                     # 列表 VO
│       └── AlertRecordDetailVo.java               # 详情 VO
├── mapper/
│   └── AlertRecordMapper.java                     # Mapper 接口
├── service/
│   ├── IAlertRecordService.java                   # Service 接口
│   └── impl/
│       ├── AlertRecordServiceImpl.java            # Service 实现
│       └── FeishuNotifyService.java               # 飞书消息发送服务
├── controller/
│   ├── AlertRecordController.java                 # CRUD 接口
│   └── AlertWebhookController.java                # Webhook 接口（无登录态）
└── job/
    └── AlertCleanupJob.java                       # 定时清理任务

RuoYi-Vue-Plus/ruoyi-modules/ruoyi-inspection/src/main/resources/mapper/inspection/
└── AlertRecordMapper.xml                          # Mapper XML

RuoYi-Vue-Plus/script/sql/
└── alert.sql                                      # 建表 + 字典 + 菜单 SQL

plus-ui/src/
├── api/inspection/alarm/
│   ├── index.ts                                   # API 函数
│   └── types.ts                                   # TypeScript 类型
└── views/inspection/alarm/
    ├── index.vue                                  # 告警记录列表页
    └── detail.vue                                 # 告警详情页
```

**Structure Decision**: 在现有 `ruoyi-inspection` 模块中扩展，与巡检报告共享基础设施。新增类遵循现有的 domain/bo/vo/mapper/service/controller 分层结构。

## Complexity Tracking

> 无违规项，所有设计符合 Constitution 要求。
