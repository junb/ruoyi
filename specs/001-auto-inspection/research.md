# Research: 服务器自动巡检

**Feature**: `001-auto-inspection`
**Date**: 2026-04-05

## 研究决策记录

### R1: SSE 实时推送方案

**Decision**: 使用项目已有的 `ruoyi-common-sse` 模块，通过 `SseMessageUtils.publishMessage()` 推送巡检状态变更。

**Rationale**:
- 项目已有完整的 SSE 基础设施（`SseEmitterManager`、Redis 跨节点分发、心跳检测）
- 前端已集成 SSE 接收（`plus-ui/src/utils/sse.ts`），自动重连、通知展示均已实现
- 发送示例：工作流通知（`FlwCommonServiceImpl`）、登录欢迎消息（`AuthController`）

**Alternatives**:
- WebSocket：项目已实现但默认关闭（`websocket.enabled: false`），SSE 更轻量
- 前端轮询：实现简单但用户体验差，不推荐

**Implementation Pattern**:
```java
// 推送给指定用户
SseMessageDto dto = new SseMessageDto();
dto.setUserIds(List.of(userId));
dto.setMessage("巡检报告生成完成");
SseMessageUtils.publishMessage(dto);
```

### R2: 异步执行方案

**Decision**: 使用 Spring `@Async` + Spring Boot 3.5 内置线程池执行巡检报告生成。

**Rationale**:
- 项目已启用 `@EnableAsync`（`ApplicationConfig.java`）
- Spring Boot 3.5 自带线程池配置（`spring.task.execution.mode: force`，前缀 `async-`）
- 无需额外配置 ThreadPoolConfig，直接使用 `@Async` 注解即可
- 现有项目已有异步使用先例：操作日志（`SysOperLogServiceImpl`）、登录信息（`SysLogininforServiceImpl`）

**Alternatives**:
- CompletableFuture：灵活但缺少 Spring 的事务管理和异常处理支持
- 线程池直接提交：过于底层，不符合项目风格

### R3: 定时任务方案

**Decision**: 使用 SnailJob 分布式任务调度框架。

**Rationale**:
- 项目已集成 SnailJob（`ruoyi-common-job` 模块），配置 `snail-job.enabled: true`
- 使用 `@JobExecutor(name = "xxx")` + `@Component` 注解定义任务
- 任务返回 `ExecuteResult.success()` 或 `ExecuteResult.failure()`
- 现有示例：`TestAnnoJobExecutor`、`AlipayBillTask`

**Implementation Pattern**:
```java
@Component
@JobExecutor(name = "inspectionScheduledJob")
public class InspectionScheduledJob {
    public ExecuteResult jobExecute(JobArgs jobArgs) {
        // 执行巡检逻辑
        return ExecuteResult.success("巡检报告生成成功");
    }
}
```

### R4: 外部 HTTP 调用方案

**Decision**: 使用 Hutool 的 `HttpRequest` 调用 Prometheus API 和百炼 API。

**Rationale**:
- 项目已在 `ruoyi-common-core` 中引入 `hutool-http` 依赖
- 社交登录模块（`AuthTopIamRequest`）已有 Hutool HTTP 使用先例
- Hutool HTTP API 简洁，支持链式调用，适合简单的 GET/POST 请求
- 无需额外引入 RestTemplate 或 OkHttp

**Implementation Pattern**:
```java
// Prometheus 查询
String result = HttpRequest.get(prometheusUrl + "/api/v1/query?query=" + encodedQuery)
    .timeout(10000)
    .execute().body();

// 百炼 API 调用
String response = HttpRequest.post(bailianUrl)
    .header("Authorization", "Bearer " + apiKey)
    .body(JsonUtils.toJsonString(requestBody))
    .execute().body();
```

### R5: 字典配置读取方案

**Decision**: 使用 `DictService` 读取巡检相关配置项。

**Rationale**:
- 项目提供 `org.dromara.common.core.service.DictService` 通用字典服务
- 支持按 `dictType` 获取所有字典值的 Map
- 字典类型：`prometheus_inspection`，包含 `PROMETHEUS_ENDPOINT`、`BAILIAN_API_KEY`、`BAILIAN_APP_ID`、`RETENTION_MONTHS` 等键

**Implementation Pattern**:
```java
@Autowired
private DictService dictService;

Map<String, String> config = dictService.getAllDictByDictType("prometheus_inspection");
String endpoint = config.get("PROMETHEUS_ENDPOINT");
```

### R6: 文件下载方案

**Decision**: 复用项目现有的 `FileUtils.setAttachmentResponseHeader()` 处理 RFC 5987 中文文件名。

**Rationale**:
- `FileUtils` 已实现 `percentEncode` + `filename*=utf-8''` 标准格式
- 现有下载场景：OSS 文件下载（`SysOssServiceImpl`）、代码生成下载（`GenController`）
- 直接写入 response 流，无需临时文件

### R7: 数据库表设计

**Decision**: 两张业务表（`prometheus_inspection_report`、`prometheus_inspection_instance_metric`），需加入多租户支持字段。

**Rationale**:
- Constitution 规定新增业务表必须包含 `tenant_id` 字段
- 实体类继承 `TenantEntity`（含 `tenant_id`、`create_by`、`create_time`、`update_by`、`update_time`、`del_flag`）
- 原始需求文档已提供完整字段定义，需补充审计字段
- 磁盘分区使用 JSON 类型存储（MySQL 5.7+ 原生支持 JSON 类型）
