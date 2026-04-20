# 004 - 项目知识助手设计文档

## 概述

新增项目知识助手功能，包含知识管理和知识问答两个核心能力。用户可上传项目文档（PDF/Word/文本），系统自动解析、分块、向量化存储到 Chroma；用户可通过多轮对话问答，系统检索相关文档片段后调用百炼大模型生成回答，支持 SSE 流式输出。

## 技术决策

| 决策项 | 选择 | 理由 |
|--------|------|------|
| 架构方案 | 后端 RAG + SSE 流式输出 | 复用项目已有 ruoyi-common-sse；多轮对话体验好 |
| 向量数据库 | Chroma Server（独立部署） | 生产环境稳定，职责分离 |
| Embedding | 百炼 text-embedding-v3 | 无需本地模型，与项目 DashScope SDK 一致 |
| LLM | 百炼 qwen-plus | 性价比高，中文能力强 |
| 文档解析 | Apache Tika | PDF/Word/文本一站解决 |
| 后端模块 | 新建 ruoyi-knowledge | 职责清晰，独立演进 |
| 用户范围 | 所有登录用户 | 知识共享，全员可用 |

## 整体架构

```
┌─────────────────────────────────────────────────┐
│                   前端 (plus-ui)                  │
│  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  知识管理页面  │  │  知识搜索/对话页面 (SSE)   │  │
│  └──────┬───────┘  └──────────┬───────────────┘  │
└─────────┼─────────────────────┼──────────────────┘
          │ REST                │ SSE + REST
┌─────────┴─────────────────────┴──────────────────┐
│             ruoyi-knowledge (新建模块)              │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │ DocumentSvc │  │ KnowledgeSvc │  │ ChatSvc  │ │
│  │ 上传/解析/分块│  │ 检索/Embedding│  │ 多轮对话  │ │
│  └──────┬──────┘  └──────┬───────┘  └────┬─────┘ │
│         │                │                │       │
│  ┌──────┴──────┐  ┌──────┴───────┐  ┌────┴─────┐ │
│  │ Apache Tika │  │ Chroma Client│  │DashScope │ │
│  │ 文档解析     │  │ 向量存储/检索  │  │SDK       │ │
│  └─────────────┘  └──────────────┘  └──────────┘ │
└───────────────────────────────────────────────────┘
         │                  │                │
    ┌────┴────┐    ┌───────┴───────┐  ┌─────┴─────┐
    │ MySQL   │    │ Chroma Server │  │ 百炼 API   │
    │ 文档元数据│    │ 向量存储       │  │ Embed+LLM │
    └─────────┘    └───────────────┘  └───────────┘
```

### 核心组件职责

- **DocumentService** — 文档上传、Tika 解析、分块、调用百炼 Embedding 向量化后存入 Chroma
- **ChatService** — 管理多轮对话上下文，检索相关片段 + 对话历史 → 调用百炼 LLM → SSE 流式输出
- **DashScopeEmbeddingService** — 封装百炼 Embedding API 调用

### 数据存储分离策略

- MySQL：文档元数据、会话、消息（结构化查询需求）
- Chroma：向量和文档片段（相似度检索需求）
- OSS：原始文件存储

## 数据模型

所有实体继承 TenantEntity（含 tenant_id、create_dept、create_by、create_time、update_by、update_time、del_flag 审计字段），以下仅列出业务字段。

### kn_document（知识文档表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 雪花ID主键 |
| title | varchar(255) | 文档标题 |
| file_name | varchar(500) | 原始文件名 |
| file_path | varchar(1000) | OSS存储路径 |
| file_type | varchar(20) | 文件类型（pdf/docx/txt/md等） |
| file_size | bigint | 文件大小（字节） |
| chunk_count | int | 分块数量 |
| status | char(1) | 状态：0解析中 1成功 2失败 |
| remark | varchar(500) | 备注 |

### kn_chat_session（对话会话表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 雪花ID主键 |
| title | varchar(255) | 会话标题 |

### kn_chat_message（对话消息表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 雪花ID主键 |
| session_id | bigint | 关联会话ID |
| role | varchar(20) | 角色：user/assistant |
| content | text | 消息内容 |
| source_docs | json | 引用的文档片段（JSON数组） |

### Chroma 存储结构

- **Collection**: `knowledge_base`（全局一个 Collection）
- **Document**: 每个分块为一个 Document
- **Metadata**: `{ document_id, tenant_id, chunk_index, file_name }`
- **Embedding**: 百炼 text-embedding-v3，维度 1536

## 核心流程

### 文档上传与向量化

```
用户上传文件 → DocumentController
    ├─ 1. 保存文件到 OSS，写入 kn_document（状态=解析中）
    ├─ 2. @Async 异步处理：
    │     ├─ Apache Tika 解析文档提取纯文本
    │     ├─ 按固定长度（500字）+ 重叠（50字）分块
    │     ├─ 批量调用百炼 Embedding API 向量化
    │     ├─ 写入 Chroma（metadata 含 document_id, tenant_id）
    │     └─ 更新 kn_document 状态=成功，chunk_count
    └─ 3. 解析失败 → 更新状态=失败，记录错误信息
```

### 知识搜索对话（SSE）

```
用户发送问题 → ChatController (SSE)
    ├─ 1. 调用百炼 Embedding API 将问题向量化
    ├─ 2. Chroma 相似度检索 Top-K（K=5），按 tenant_id 过滤
    ├─ 3. 加载该会话最近 N 轮历史消息（N=10）
    ├─ 4. 构建 Prompt：
    │     System: "你是知识助手，根据以下参考内容回答问题..."
    │     + 参考文档片段（检索结果）
    │     + 历史对话
    │     + 用户当前问题
    ├─ 5. 调用百炼 LLM（qwen-plus），SSE 流式返回
    └─ 6. 流式结束后，保存用户消息和助手回复到 kn_chat_message
          source_docs 记录引用的文档片段信息
```

### 关键参数

| 参数 | 值 | 说明 |
|------|------|------|
| 分块大小 | 500 字符 | 平衡精度和上下文 |
| 分块重叠 | 50 字符 | 避免语义断裂 |
| 检索 Top-K | 5 | 返回最相关的 5 个片段 |
| 对话历史轮数 | 10 轮 | 控制上下文窗口 |
| Embedding 模型 | text-embedding-v3 | 百炼最新文本向量模型 |
| LLM 模型 | qwen-plus | 百炼通义千问，性价比较高 |

## 后端模块结构

```
ruoyi-modules/ruoyi-knowledge/
├── pom.xml
└── src/main/java/org/dromara/knowledge/
    ├── controller/
    │   ├── KnowledgeDocumentController.java
    │   └── KnowledgeChatController.java
    ├── domain/
    │   ├── KnDocument.java
    │   ├── KnChatSession.java
    │   └── KnChatMessage.java
    ├── domain/vo/
    │   ├── KnDocumentVo.java
    │   ├── KnChatSessionVo.java
    │   ├── KnChatMessageVo.java
    │   └── ChatRequestVo.java
    ├── domain/bo/
    │   ├── KnDocumentBo.java
    │   └── KnChatSessionBo.java
    ├── mapper/
    │   ├── KnDocumentMapper.java
    │   ├── KnChatSessionMapper.java
    │   └── KnChatMessageMapper.java
    └── service/
        ├── IKnDocumentService.java
        ├── IKnChatService.java
        ├── impl/
        │   ├── KnDocumentServiceImpl.java
        │   └── KnChatServiceImpl.java
        └── embedding/
            └── DashScopeEmbeddingService.java
```

## API 设计

### 文档管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /knowledge/document/upload | 上传文档（MultipartFile） |
| GET | /knowledge/document/list | 文档列表（分页） |
| GET | /knowledge/document/{id} | 文档详情 |
| DELETE | /knowledge/document/{ids} | 删除文档（同步删除 Chroma 向量） |

### 对话

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /knowledge/chat/send | 发送消息（返回 SSE 流） |
| GET | /knowledge/chat/sessions | 会话列表 |
| POST | /knowledge/chat/session | 新建会话 |
| DELETE | /knowledge/chat/session/{ids} | 删除会话 |
| GET | /knowledge/chat/messages/{sessionId} | 获取会话历史消息 |

## 新增依赖

```xml
<!-- Apache Tika 文档解析 -->
<dependency>
    <groupId>org.apache.tika</groupId>
    <artifactId>tika-core</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.tika</groupId>
    <artifactId>tika-parsers-standard-package</artifactId>
</dependency>

<!-- Chroma DB Client -->
<dependency>
    <groupId>io.github.amikos-tech</groupId>
    <artifactId>chromadb-java-client</artifactId>
</dependency>

<!-- 百炼 DashScope SDK（版本与 ruoyi-inspection 保持一致） -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>dashscope-sdk-java</artifactId>
</dependency>
```

## 前端设计

### 菜单结构

```
知识助手（新增一级菜单）
├── 知识管理（/knowledge/document）
└── 知识问答（/knowledge/chat）
```

### 知识管理页面

- 上传：多文件批量上传，使用 Element Plus el-upload 组件
- 列表：文档名称、文件类型、大小、分块数、状态、操作（删除）
- 状态：解析中（轮询刷新）、成功、失败
- 支持按文档名称搜索

### 知识问答页面

- 左侧：会话列表，支持新建、删除、切换
- 右侧：对话气泡样式，用户消息靠右，AI 消息靠左
- AI 回复：SSE 流式渲染，Markdown 格式，引用来源折叠显示
- 输入框：Enter 发送，Shift+Enter 换行

### 前端文件结构

```
plus-ui/src/
├── api/knowledge/
│   ├── document.ts
│   ├── chat.ts
│   └── types/
├── views/knowledge/
│   ├── document/
│   │   └── index.vue
│   └── chat/
│       ├── index.vue
│       ├── ChatDialog.vue
│       └── SessionList.vue
```

## 错误处理

- 文档解析失败：记录错误日志，更新状态为失败，不阻断其他文档
- Chroma 不可用：返回服务不可用提示，不降级
- 百炼 API 超时：SSE 流返回超时提示，对话消息仍保存
- 文件格式不支持：上传时校验文件后缀，拒绝不支持的格式
- 文件大小限制：单文件上限 20MB
