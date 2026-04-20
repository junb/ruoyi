# API Contracts: 项目知识助手

**Branch**: `005-knowledge-assistant` | **Date**: 2026-04-18

## 文档管理 API

### POST /knowledge/document/upload

上传文档，异步解析向量化。

- **Content-Type**: `multipart/form-data`
- **Permission**: `knowledge:document:upload`
- **Request**:
  - `file`: MultipartFile（必填，支持 pdf/docx/doc/txt/md，单文件最大 20MB）
  - `title`: String（可选，默认取文件名）
  - `remark`: String（可选）
- **Response**: `R<KnDocumentVo>`

### GET /knowledge/document/list

文档列表（分页）。

- **Permission**: `knowledge:document:list`
- **Query Params**:
  - `title`: String（可选，模糊搜索）
  - `fileType`: String（可选，精确过滤）
  - `status`: String（可选，精确过滤）
  - `pageNum`: int（默认 1）
  - `pageSize`: int（默认 10）
- **Response**: `TableDataInfo<KnDocumentVo>`

### GET /knowledge/document/{id}

文档详情。

- **Permission**: `knowledge:document:query`
- **Response**: `R<KnDocumentVo>`

### DELETE /knowledge/document/{ids}

删除文档（逗号分隔的 ID 列表），同步删除 Chroma 向量。

- **Permission**: `knowledge:document:remove`
- **Response**: `R<Void>`

## 对话 API

### POST /knowledge/chat/send

发送消息，SSE 流式返回 AI 回答。

- **Content-Type**: `application/json`
- **Produces**: `text/event-stream`
- **Permission**: `knowledge:chat:send`
- **Rate Limit**: 每用户每分钟 10 次（`@RateLimiter`）
- **Request Body** (`ChatRequestVo`):
  ```json
  {
    "sessionId": 1234567890,
    "content": "项目的技术架构是怎样的？"
  }
  ```
  - `sessionId`: bigint（必填，若传 0 或 null 则自动创建新会话）
  - `content`: String（必填，@NotBlank，最大 2000 字符）
- **Response**: `SseEmitter`（流式事件）
  - 事件格式: `data: {"content": "部分内容"}\n\n`
  - 结束事件: `data: {"done": true, "sources": [...]}\n\n`
  - 错误事件: `data: {"error": "错误信息"}\n\n`

### GET /knowledge/chat/sessions

当前用户的会话列表。

- **Permission**: `knowledge:chat:list`
- **Response**: `R<List<KnChatSessionVo>>`（按创建时间降序）

### POST /knowledge/chat/session

新建会话。

- **Permission**: `knowledge:chat:add`
- **Request Body** (可选):
  ```json
  {
    "title": "自定义标题"
  }
  ```
- **Response**: `R<KnChatSessionVo>`（默认标题 "新对话"）

### DELETE /knowledge/chat/session/{ids}

删除会话（逗号分隔），级联删除消息。

- **Permission**: `knowledge:chat:remove`
- **Response**: `R<Void>`

### GET /knowledge/chat/messages/{sessionId}

获取会话历史消息。

- **Permission**: `knowledge:chat:list`
- **Response**: `R<List<KnChatMessageVo>>`（按创建时间升序）

## VO 定义

### KnDocumentVo

```json
{
  "id": 1234567890,
  "title": "项目技术手册",
  "fileName": "手册.pdf",
  "fileType": "pdf",
  "fileSize": 2097152,
  "chunkCount": 45,
  "status": "1",
  "remark": "",
  "createBy": 1,
  "createTime": "2026-04-18 10:00:00"
}
```

### KnChatSessionVo

```json
{
  "id": 1234567890,
  "title": "项目的技术架构是怎样的...",
  "createTime": "2026-04-18 10:30:00"
}
```

### KnChatMessageVo

```json
{
  "id": 1234567890,
  "sessionId": 1234567890,
  "role": "assistant",
  "content": "根据项目文档...",
  "sourceDocs": [
    {
      "contentSnippet": "该项目采用..."
    }
  ],
  "createTime": "2026-04-18 10:30:05"
}
```

### ChatRequestVo

```json
{
  "sessionId": 1234567890,
  "content": "请介绍项目的技术架构"
}
```
