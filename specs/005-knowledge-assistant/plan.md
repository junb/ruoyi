# Implementation Plan: 项目知识助手

**Branch**: `005-knowledge-assistant` | **Date**: 2026-04-18 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/005-knowledge-assistant/spec.md`

## Summary

新增项目知识助手功能：后端新建 `ruoyi-knowledge` 模块，支持用户上传文档（PDF/Word/文本）经 Apache Tika 解析、分块后通过百炼 Embedding API 向量化存入 Chroma Server；前端知识问答页面通过 SSE 流式输出实现多轮 RAG 对话（百炼 qwen-plus），支持引用来源标注、会话管理、速率限制。

## Technical Context

**Language/Version**: Java 17（Spring Boot 3.5.12）+ TypeScript ~5.9.3（Vue 3.5.30）
**Primary Dependencies**: MyBatis-Plus 3.5.16, Sa-Token 1.44.0, DashScope SDK 2.22.13, Apache Tika 2.9.2, Element Plus 2.13.5
**Storage**: MySQL（文档元数据/会话/消息）, Chroma Server（向量存储）, OSS（原始文件）
**Testing**: Spring Boot Test + JUnit 5
**Target Platform**: Linux server（后端）, 现代浏览器（前端）
**Project Type**: Web application（前后端分离）
**Performance Goals**: 5MB 文档 30 秒内完成向量化，AI 回答 2 秒内开始流式输出
**Constraints**: 单文件 20MB 上限，每用户每分钟 10 次提问，对话历史 10 轮
**Scale/Scope**: 所有登录用户可用，支持多租户隔离

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 原则 | 状态 | 说明 |
|------|------|------|
| I. 分层架构 | PASS | Controller → Service → Mapper 三层分离，SSE 流式输出在 Service 层处理 |
| II. 统一响应与异常处理 | PASS | 普通接口返回 R<T>/TableDataInfo<T>，SSE 接口使用 SseEmitter（特殊场景合理例外） |
| III. 多租户数据隔离 | PASS | 所有实体继承 TenantEntity，Chroma 按 tenant_id metadata 过滤 |
| IV. 注解驱动开发 | PASS | @AutoMapper 做对象转换，@RateLimiter 做限流，@Validated 校验参数 |
| V. 安全与权限 | PASS | 所有 Controller 方法标注 @SaCheckPermission。SSE 接口因异步线程 Sa-Token ThreadLocal 不可用，改为手动调用 StpUtil.checkPermission() 在同步阶段完成鉴权 |
| VI. 缓存策略 | PASS | 文档列表等查询可考虑 @Cacheable，但对话场景不适合缓存 |
| VII. 前后端协作规范 | PASS | API 模块一一对应，TypeScript 类型同步 |

**无违规项**。

## Project Structure

### Documentation (this feature)

```text
specs/005-knowledge-assistant/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   └── api.md           # Phase 1 output
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 output (/speckit.tasks)
```

### Source Code (repository root)

```text
# 后端 - 新建模块
RuoYi-Vue-Plus/
├── ruoyi-modules/
│   └── ruoyi-knowledge/                    # 新建模块
│       ├── pom.xml
│       └── src/main/java/org/dromara/knowledge/
│           ├── controller/
│           │   ├── KnowledgeDocumentController.java
│           │   └── KnowledgeChatController.java
│           ├── domain/
│           │   ├── KnDocument.java
│           │   ├── KnChatSession.java
│           │   └── KnChatMessage.java
│           ├── domain/vo/
│           │   ├── KnDocumentVo.java
│           │   ├── KnChatSessionVo.java
│           │   ├── KnChatMessageVo.java
│           │   └── ChatRequestVo.java
│           ├── domain/bo/
│           │   ├── KnDocumentBo.java
│           │   └── KnChatSessionBo.java
│           ├── mapper/
│           │   ├── KnDocumentMapper.java
│           │   ├── KnChatSessionMapper.java
│           │   └── KnChatMessageMapper.java
│           └── service/
│               ├── IKnDocumentService.java
│               ├── IKnChatService.java
│               ├── impl/
│               │   ├── KnDocumentServiceImpl.java
│               │   └── KnChatServiceImpl.java
│               ├── embedding/
│               │   └── DashScopeEmbeddingService.java
│               └── chroma/
│                   └── ChromaApiService.java
│           ├── config/
│           │   ├── ChromaConfig.java
│           │   └── KnowledgeApiKeyProvider.java
│
├── ruoyi-admin/
│   └── pom.xml                             # 新增 ruoyi-knowledge 依赖
│
└── ruoyi-modules/pom.xml                    # 新增 ruoyi-knowledge 模块

# 前端
plus-ui/src/
├── api/knowledge/
│   ├── document.ts
│   ├── chat.ts
│   └── types/
│       ├── document.ts
│       └── chat.ts
├── views/knowledge/
│   ├── document/
│   │   └── index.vue
│   └── chat/
│       ├── index.vue
│       ├── ChatDialog.vue
│       └── SessionList.vue
```

**Structure Decision**: 遵循项目已有的模块化结构，后端新建 `ruoyi-knowledge` 模块（与 `ruoyi-inspection` 同级），前端在 `views/knowledge/` 下组织页面。

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| III. 多租户 — Chroma 查询手动传 tenant_id | Chroma 是独立向量数据库，不支持 MyBatis-Plus 租户插件自动注入，必须通过 metadata where 过滤实现租户隔离 | Chroma 无 Collection 级别的多租户支持（按租户建 Collection 管理复杂度高且不可扩展） |
