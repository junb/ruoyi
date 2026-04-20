# Tasks: 项目知识助手

**Input**: Design documents from `/specs/005-knowledge-assistant/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Tests**: 未在规格中明确要求，不包含测试任务。

**Organization**: 任务按用户故事分组，每个故事可独立实现和测试。

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 可并行执行（不同文件，无依赖）
- **[Story]**: 所属用户故事（US1, US2, US3）
- 包含具体文件路径

## Phase 1: Setup（项目初始化）

**Purpose**: 创建后端模块骨架和前端目录结构

- [x] T001 创建 ruoyi-knowledge 模块目录结构及 pom.xml，参考 ruoyi-inspection 模块结构，路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/pom.xml`
- [x] T002 [P] 在 ruoyi-modules 的父 pom.xml 中添加 ruoyi-knowledge 模块声明，路径：`RuoYi-Vue-Plus/ruoyi-modules/pom.xml`
- [x] T003 [P] 在 ruoyi-admin 的 pom.xml 中添加 ruoyi-knowledge 依赖，路径：`RuoYi-Vue-Plus/ruoyi-admin/pom.xml`
- [x] T004 [P] 添加 application.yml 配置项（chroma url、embedding 模型、chat 模型、分块参数等），路径：`RuoYi-Vue-Plus/ruoyi-admin/src/main/resources/application-dev.yml`
- [x] T005 [P] 创建前端目录结构和 TypeScript 类型定义，路径：`plus-ui/src/api/knowledge/types/document.ts` 和 `plus-ui/src/api/knowledge/types/chat.ts`

---

## Phase 2: Foundational（基础设施，阻塞性前置）

**Purpose**: 所有用户故事共享的核心实体、配置类和基础设施

**⚠️ CRITICAL**: 所有用户故事任务必须在此阶段完成后才能开始

- [x] T006 创建 KnDocument 实体类（继承 TenantEntity），路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/domain/KnDocument.java`
- [x] T007 [P] 创建 KnChatSession 实体类（继承 TenantEntity），路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/domain/KnChatSession.java`
- [x] T008 [P] 创建 KnChatMessage 实体类（继承 TenantEntity），路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/domain/KnChatMessage.java`
- [x] T009 [P] 创建 KnDocumentVo、KnDocumentBo（@AutoMapper 转换），路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/domain/vo/KnDocumentVo.java` 和 `domain/bo/KnDocumentBo.java`
- [x] T010 [P] 创建 KnChatSessionVo、KnChatSessionBo、KnChatMessageVo、ChatRequestVo，路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/domain/vo/` 和 `domain/bo/`
- [x] T011 [P] 创建 Mapper 接口（KnDocumentMapper、KnChatSessionMapper、KnChatMessageMapper），继承 BaseMapperPlus<Entity, Vo>，路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/mapper/`
- [x] T012 创建建表 SQL 脚本（kn_document、kn_chat_session、kn_chat_message 含索引），路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/resources/sql/knowledge.sql`
- [x] T013 [P] 创建 DashScopeEmbeddingService 封装类（百炼 text-embedding-v3 批量向量化，每批最多25条），路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/service/embedding/DashScopeEmbeddingService.java`
- [x] T014 [P] 创建 Chroma 配置属性类和 Client Bean（读取 application.yml 中的 knowledge.chroma 配置），路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/config/ChromaConfig.java`

**Checkpoint**: 基础设施就绪，用户故事可并行开发

---

## Phase 3: User Story 1 - 上传项目文档建立知识库 (Priority: P1) 🎯 MVP

**Goal**: 用户可上传 PDF/Word/文本文件，系统异步解析、分块、向量化存入 Chroma，展示处理状态

**Independent Test**: 上传一个 PDF 和一个 Markdown 文件，验证文档状态从"解析中"变为"成功"，分块数量正确显示

### Implementation for User Story 1

- [x] T015 [US1] 创建 IKnDocumentService 接口和 KnDocumentServiceImpl 实现类（上传、列表、详情、删除），路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/service/IKnDocumentService.java` 和 `service/impl/KnDocumentServiceImpl.java`
- [x] T016 [US1] 实现文档上传核心逻辑：保存文件到 OSS + 写入 kn_document（状态=解析中）+ @Async 异步处理（Tika 解析 → 500字分块/50字重叠 → 批量 Embedding → 写入 Chroma → 更新状态），在 KnDocumentServiceImpl 中实现
- [x] T017 [US1] 实现文档删除逻辑：删除 kn_document + 删除 Chroma 中对应 document_id 的所有向量（chunk ID 格式 {document_id}_{index}），在 KnDocumentServiceImpl 中实现
- [x] T018 [US1] 创建 KnowledgeDocumentController（upload、list、getById、remove），标注 @SaCheckPermission 权限注解和 @Log 操作日志注解，路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/controller/KnowledgeDocumentController.java`
- [x] T019 [P] [US1] 创建前端文档管理 API 模块（upload、list、getInfo、remove），路径：`plus-ui/src/api/knowledge/document.ts`
- [x] T020 [US1] 实现前端知识管理页面（el-upload 批量上传、el-table 列表展示含状态列和分块数、搜索、删除），路径：`plus-ui/src/views/knowledge/document/index.vue`
- [x] T021 [US1] 创建菜单 SQL（知识助手一级菜单 + 知识管理子菜单 + 权限按钮），在建表 SQL 中追加

**Checkpoint**: 用户可上传文档、查看解析状态和分块数、删除文档。MVP 就绪。

---

## Phase 4: User Story 2 - 通过多轮对话进行知识问答 (Priority: P2)

**Goal**: 用户在问答页面提问，系统通过 Chroma 检索 + 百炼 qwen-plus 流式生成回答，支持多轮上下文和引用来源

**Independent Test**: 确保至少一个文档已入库，提问验证流式输出、引用来源、连续追问上下文保持

### Implementation for User Story 2

- [x] T022 [US2] 创建 IKnChatService 接口和 KnChatServiceImpl 实现类，路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/service/IKnChatService.java` 和 `service/impl/KnChatServiceImpl.java`
- [x] T023 [US2] 实现问答核心逻辑（RAG 流程）：问题 Embedding → Chroma 查询 Top-5（where tenant_id 过滤）→ 加载会话最近 10 轮历史 → 构建 Prompt（System + 参考片段 + 历史 + 问题）→ 调用 qwen-plus 流式接口 → 逐 chunk 通过 SseEmitter 发送 → 完成后保存消息，在 KnChatServiceImpl 中实现
- [x] T024 [US2] 实现会话管理逻辑：创建会话（标题默认"新对话"）、首次提问后自动取问题前20字更新标题、删除会话级联删除消息，在 KnChatServiceImpl 中实现
- [x] T025 [US2] 创建 KnowledgeChatController（send 返回 SseEmitter、sessions、addSession、removeSessions、messages），send 方法标注 @RateLimiter(count=10, time=60)，所有方法标注 @SaCheckPermission 和 @Log 注解，路径：`RuoYi-Vue-Plus/ruoyi-modules/ruoyi-knowledge/src/main/java/org/dromara/knowledge/controller/KnowledgeChatController.java`
- [x] T026 [P] [US2] 创建前端对话 API 模块（sendChat 使用 EventSource/SSE、sessions、addSession、removeSession、messages），路径：`plus-ui/src/api/knowledge/chat.ts`
- [x] T027 [US2] 实现前端知识问答页面主框架（左侧会话列表 + 右侧对话区），路径：`plus-ui/src/views/knowledge/chat/index.vue`
- [x] T028 [P] [US2] 实现前端 SessionList 组件（新建对话、切换会话、删除会话、时间倒序排列），路径：`plus-ui/src/views/knowledge/chat/SessionList.vue`
- [x] T029 [P] [US2] 实现前端 ChatDialog 组件（对话气泡、SSE 流式渲染 Markdown、引用来源折叠显示、Enter 发送/Shift+Enter 换行），路径：`plus-ui/src/views/knowledge/chat/ChatDialog.vue`
- [x] T030 [US2] 追加菜单 SQL（知识问答子菜单 + 权限按钮），在建表 SQL 中追加

**Checkpoint**: 用户可通过多轮对话进行知识问答，流式输出、引用来源、上下文保持均正常

---

## Phase 5: User Story 3 - 管理对话会话 (Priority: P3)

**Goal**: 用户可创建、切换、删除会话，每个会话独立维护对话历史

**Independent Test**: 创建多个会话分别问答，切换后历史消息正确加载，删除后从列表移除

### Implementation for User Story 3

- [x] T031 [US3] 实现会话列表查询接口（当前用户的会话按创建时间降序），在 KnChatServiceImpl 中补充完善
- [x] T032 [US3] 实现历史消息加载接口（按会话 ID 查询消息，按创建时间升序），在 KnChatServiceImpl 中补充完善
- [x] T033 [US3] 完善前端会话切换逻辑：点击历史会话加载消息、新建会话清空对话区、删除后自动切换到最近会话，在 `plus-ui/src/views/knowledge/chat/index.vue` 中实现

**Checkpoint**: 会话管理功能完整，用户可自由管理多个问答会话

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: 跨故事的优化和完善

- [x] T034 [P] 添加文档解析中状态的轮询刷新（前端定时器每 5 秒检查状态），在 `plus-ui/src/views/knowledge/document/index.vue` 中实现
- [x] T035 [P] 添加 SSE 错误处理（连接超时、AI 服务不可用时显示友好提示），在 `plus-ui/src/views/knowledge/chat/ChatDialog.vue` 中实现
- [ ] T036 [P] 验证多租户隔离（不同租户上传文档后交叉提问，确认检索结果隔离）
- [ ] T037 验证完整流程：上传文档 → 等待解析 → 新建对话 → 提问 → 流式回答 → 引用来源 → 追问 → 切换会话 → 删除文档 → 确认历史引用保留
- [ ] T038 [P] 性能验证：上传 5MB PDF 确认 30 秒内向量化完成（SC-001），提问确认 2 秒内开始流式输出（SC-002）

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 无依赖，立即开始
- **Foundational (Phase 2)**: 依赖 Phase 1 完成 — 阻塞所有用户故事
- **US1 (Phase 3)**: 依赖 Phase 2 完成 — MVP
- **US2 (Phase 4)**: 依赖 Phase 2 完成，建议 US1 完成后开始（需要文档数据支撑问答）
- **US3 (Phase 5)**: 依赖 Phase 4 完成（会话管理是对问答功能的增强）
- **Polish (Phase 6)**: 依赖所有用户故事完成

### User Story Dependencies

- **US1 (P1)**: Phase 2 完成后可独立开始
- **US2 (P2)**: 建议 US1 完成后开始（需要至少一个文档入库才能测试问答），但代码层面可并行开发
- **US3 (P3)**: 依赖 US2 的会话 API

### Parallel Opportunities

**Phase 1**: T002、T003、T004、T005 可并行
**Phase 2**: T007、T008、T009、T010、T011、T013、T014 可并行
**Phase 3**: T019 可与其他 US1 任务并行
**Phase 4**: T026、T028、T029 可并行
**Phase 6**: T034、T035、T036 可并行

---

## Parallel Example: Phase 2

```bash
# 并行创建所有实体类：
Task T006: "创建 KnDocument 实体"
Task T007: "创建 KnChatSession 实体"
Task T008: "创建 KnChatMessage 实体"

# 并行创建所有 VO/BO：
Task T009: "创建 KnDocumentVo、KnDocumentBo"
Task T010: "创建会话和消息相关 VO/BO"

# 并行创建基础设施：
Task T013: "创建 DashScopeEmbeddingService"
Task T014: "创建 Chroma 配置"
```

## Parallel Example: Phase 4

```bash
# 并行创建前端组件：
Task T028: "SessionList 组件"
Task T029: "ChatDialog 组件"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. 完成 Phase 1: Setup
2. 完成 Phase 2: Foundational
3. 完成 Phase 3: User Story 1（文档上传 + 解析 + 向量化）
4. **STOP and VALIDATE**: 上传文档，验证解析状态和分块数
5. 可部署/演示 MVP

### Incremental Delivery

1. Setup + Foundational → 基础就绪
2. US1 → 文档管理可用（MVP!）
3. US2 → 多轮问答可用
4. US3 → 会话管理增强
5. Polish → 生产级完善

---

## Notes

- [P] 任务 = 不同文件，无依赖，可并行
- [Story] 标签将任务映射到具体用户故事
- 每个用户故事独立可完成和测试
- 每个任务或逻辑组完成后提交
- 任何 Checkpoint 处可停下来独立验证
