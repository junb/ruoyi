# Data Model: 项目知识助手

**Branch**: `005-knowledge-assistant` | **Date**: 2026-04-18

## 实体关系

```
KnDocument (1) ──→ (N) Chroma Vector Chunks
KnChatSession (1) ──→ (N) KnChatMessage
KnUser (1) ──→ (N) KnChatSession  [通过 create_by 关联]
KnUser (1) ──→ (N) KnDocument     [通过 create_by 关联]
```

## 实体定义

### KnDocument（知识文档）

所有实体继承 `TenantEntity`（含 tenant_id、create_dept、create_by、create_time、update_by、update_time、del_flag），以下仅列出业务字段。

| 字段 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|----------|------|
| id | bigint | PK | 雪花ID | 主键 |
| title | varchar(255) | Y | @NotBlank @Size(max=255) | 文档标题 |
| file_name | varchar(500) | Y | @NotBlank @Size(max=500) | 原始文件名 |
| file_path | varchar(1000) | Y | @NotBlank | OSS 存储路径 |
| file_type | varchar(20) | Y | @NotBlank | 文件类型后缀（pdf/docx/doc/txt/md） |
| file_size | bigint | Y | @NotNull | 文件大小（字节），上限 20MB |
| chunk_count | int | N | | 分块数量，解析完成后填充 |
| status | char(1) | Y | 默认 "0" | 状态：0=解析中 1=成功 2=失败 |
| error_msg | varchar(1000) | N | | 解析失败时的错误信息（status=2 时填充） |
| remark | varchar(500) | N | @Size(max=500) | 用户备注 |

**状态转换**: 0（解析中）→ 1（成功）/ 2（失败）。不可逆。

**向量数据（Chroma 侧）**:
- Collection: `knowledge_base`
- 每个 chunk 的 ID 格式: `{document_id}_{chunk_index}`
- Metadata: `{ document_id, tenant_id, chunk_index, file_name }`
- Embedding: float[1536]（百炼 text-embedding-v3）

### KnChatSession（对话会话）

| 字段 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|----------|------|
| id | bigint | PK | 雪花ID | 主键 |
| title | varchar(255) | Y | 默认 "新对话" | 会话标题，首次提问后自动取问题前20字 |

**业务约束**: 每个用户可创建无限会话。会话按 create_time 降序排列。

### KnChatMessage（对话消息）

| 字段 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|----------|------|
| id | bigint | PK | 雪花ID | 主键 |
| session_id | bigint | Y | @NotNull, FK → kn_chat_session.id | 关联会话 |
| role | varchar(20) | Y | @NotBlank, 枚举: user/assistant | 消息角色 |
| content | text | Y | @NotBlank | 消息内容 |
| source_docs | json | N | | 引用的文档片段信息，JSON 数组格式 |

**source_docs JSON 格式**:
```json
[
  {
    "document_id": "1234567890",
    "file_name": "项目手册.pdf",
    "chunk_index": 5,
    "content_snippet": "相关文本片段前100字..."
  }
]
```

**业务约束**: 消息按 create_time 升序排列。删除会话时级联删除所有消息。

## 数据库索引建议

```sql
-- kn_document
CREATE INDEX idx_kn_document_tenant ON kn_document(tenant_id);
CREATE INDEX idx_kn_document_status ON kn_document(status);

-- kn_chat_session
CREATE INDEX idx_kn_chat_session_tenant_user ON kn_chat_session(tenant_id, create_by);

-- kn_chat_message
CREATE INDEX idx_kn_chat_message_session ON kn_chat_message(session_id);
```
