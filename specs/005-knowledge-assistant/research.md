# Research: 项目知识助手

**Branch**: `005-knowledge-assistant` | **Date**: 2026-04-18

## 1. Chroma 向量存储

**Decision**: 使用 JDK 17 HttpClient 直接调用 Chroma REST API v2（`/api/v2/tenants/default_tenant/databases/default_database/collections`）

**Rationale**: `chromadb-java-client` 在国内 Maven 镜像不可用，且 Chroma v1 API 在 1.0+ 版本已废弃。使用 JDK 内置 HttpClient 直接调用 REST API，零额外依赖，完全可控。

**Alternatives considered**:
- `chromadb-java-client` — 国内镜像不可用，且 API 版本与 Chroma 1.0+ 不兼容
- Python 微服务封装 — 增加部署复杂度

**关键实现**: 自定义 `ChromaApiService` 封装 CRUD 操作，Collection ID 带缓存，支持 metadata where 过滤实现多租户隔离。

## 2. Apache Tika 文档解析

**Decision**: 使用 `org.apache.tika:tika-core:2.9.2` + `tika-parsers-standard-package:2.9.2`

**Rationale**: 成熟的文档解析库，支持 PDF/DOCX/TXT/MD 等主流格式，API 简洁（`Tika.parseToString()`）。

**Alternatives considered**:
- POI + PDFBox 分别处理 — 需要引入多个依赖且各自处理不同格式，维护成本高
- 仅支持纯文本格式 — 功能不足，无法满足 PDF/Word 需求

**潜在风险**: Tika 2.x 依赖中的 `commons-compress` 可能与 Spring Boot 3.x 的依赖树冲突，需通过 exclusion 处理。

## 3. 百炼 DashScope Embedding API

**Decision**: 使用项目已有的 `dashscope-sdk-java:2.22.13` 调用 `text-embedding-v3` 模型

**Rationale**: 项目已有 DashScope SDK 依赖和 API Key 配置（通过字典管理），保持技术栈一致。

**关键参数**:
- 模型: `text-embedding-v3`
- 向量维度: 1024
- 批量限制: 单次最多 10 条文本
- API Key 来源: 独立字典 `knowledge_assistant` 中的 `BAILIAN_API_KEY`，通过 `DictService.getAllDictByDictType()` 读取，yml 配置作为 fallback

## 4. 百炼 DashScope 流式对话 API

**Decision**: 使用 DashScope SDK 的 `Chat` 类实现流式对话

**Rationale**: 项目已有 DashScope SDK。现有 `BailianReportService` 使用的是 `Application`（智能体应用）模式，知识问答需要使用更灵活的 `Chat`（直接模型调用）模式，支持自定义 System Prompt 和多轮消息。

**关键模式**:
- 使用 `GenerationParam.builder().model("qwen-plus").messages(msgList).incrementalOutput(true).apiKey(...).build()`
- 流式响应通过 `Flowable<GenerationResult>` 的 `blockingForEach` 获取增量文本
- SSE 流式输出使用自定义 `SseEmitter` 返回
- **重要**: 异步线程中 Sa-Token ThreadLocal 不可用，必须在同步阶段提前获取 userId/tenantId

## 5. SSE 流式输出方案

**Decision**: 使用 Spring 的 `SseEmitter` 直接返回流式响应

**Rationale**: 项目 `ruoyi-common-sse` 模块的 `SseEmitterManager` 设计用于服务端推送场景（如通知、消息），采用 userId + token 管理长连接。知识问答的 SSE 是请求-响应模式（一次请求对应一次流式输出），更适合直接在 Controller 方法中创建 `SseEmitter` 并返回。

**实现方式**:
```java
@PostMapping(value = "/chat/send", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter chat(@RequestBody ChatRequestVo request) {
    SseEmitter emitter = new SseEmitter(300_000L); // 5分钟超时
    // 异步调用 LLM 流式接口，逐个 chunk 通过 emitter.send() 发送
    // 完成后 emitter.complete()
    return emitter;
}
```

## 6. 速率限制方案

**Decision**: 使用项目已有的 `ruoyi-common-ratelimiter` 模块，通过 `@RateLimiter` 注解实现

**Rationale**: 项目已有基于 Redis 的限流实现，通过注解即可声明式控制接口速率，无需额外开发。

## 7. 文件上传方案

**Decision**: 使用项目已有的 `ruoyi-common-oss` 模块存储上传文件

**Rationale**: 项目已有完整的 OSS 存储封装，支持 S3 协议。上传的文档文件通过 OSS 存储，仅保存路径到数据库。
