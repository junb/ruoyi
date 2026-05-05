# Tasks: 全站深色科技风 UI 改造

**Input**: Design documents from `/specs/006-dark-tech-ui/`
**Prerequisites**: plan.md (required), spec.md (required), research.md (available)

**Tests**: 无自动化测试任务，通过 quickstart.md 手动视觉验证。

**Organization**: 按 User Story 组织，每个 Story 可独立实施和验证。

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 可并行执行（不同文件，无依赖）
- **[Story]**: 所属 User Story（US1, US2, US3, US4）
- 包含精确文件路径

## Path Conventions

- 前端代码根目录: `plus-ui/src/`
- 样式文件: `plus-ui/src/assets/styles/`
- 布局组件: `plus-ui/src/layout/`
- 页面组件: `plus-ui/src/views/`

---

## Phase 1: Setup（强制深色模式）

**Purpose**: 确保系统强制进入深色模式，为后续样式改造奠定基础

- [x] T001 修改 `plus-ui/src/settings.ts` 将 `dark: false` 改为 `dark: true`，强制系统默认深色模式
- [x] T002 修改 `plus-ui/src/layout/components/Settings/index.vue` 隐藏暗黑模式开关控件和主题颜色选择器，防止用户切换回浅色

**Checkpoint**: 系统强制使用 `html.dark` 类，所有页面以深色模式渲染（虽然此时颜色可能不理想）

---

## Phase 2: Foundational（Design Token + 效果类库）

**Purpose**: 建立深色科技风的 Design Token 体系和可复用效果类，所有后续任务均依赖此阶段

**⚠️ CRITICAL**: 此阶段完成前，不可开始任何 User Story 实施

- [x] T003 重写 `plus-ui/src/assets/styles/variables.module.scss` 中的 `:root` CSS 变量：背景色 #111118、卡片 #161622、主色 #4fd1a5、边框 #2a2a3e、前景色 #f0f0f2、静默文字 #8a8a9e；覆盖 Element Plus 全套 `--el-bg-color-*`、`--el-text-color-*`、`--el-border-color-*`、`--el-fill-color-*` token；覆盖 `html.dark` 中的按钮/标签/开关语义色（success/warning/danger/info）适配深色科技风；新增 `--glass-bg`、`--glass-border`、`--glow-primary`、`--glow-secondary` 自定义属性；完成后验证主色 #4fd1a5 在背景 #111118 上的对比度 >= 4.5:1（WCAG AA），前景色 #f0f0f2 在背景 #111118 上的对比度 >= 4.5:1
- [x] T004 在 `plus-ui/src/assets/styles/index.scss` 中新增效果类和全局样式：`.glass` 玻璃拟态、`.glass-subtle` 轻量玻璃拟态、`.glow-sm` 和 `.glow` 发光效果、`.grid-pattern` 网格纹理、`.gradient-border` 渐变边框、`.animate-pulse-glow` 呼吸发光动画、深色自定义滚动条样式、`@supports` backdrop-filter 降级方案；更新 `aside` 元素的深色适配（背景色、文字色）；更新 `.app-container`、`.panel`、`.search` 的深色适配值

**Checkpoint**: Design Token 体系就绪，所有 Element Plus 组件自动继承深色主题，效果类可用于布局和页面改造

---

## Phase 3: User Story 1 - 深色框架与导航 (Priority: P1) 🎯 MVP

**Goal**: 侧边栏、顶部栏、主布局全部呈现深色科技风格，导航和交互正常工作

**Independent Test**: 启动系统，检查侧边栏深色背景 + 发光指示条、顶部栏毛玻璃效果、页面网格纹理，验证导航功能正常

### Implementation for User Story 1

- [x] T005 [P] [US1] 修改 `plus-ui/src/assets/styles/sidebar.scss`：菜单背景改为深色渐变，活跃菜单项添加 primary 色半透明背景 + 左侧 2px 发光指示条（`box-shadow: 0 0 8px rgba(79,209,165,0.4)`），hover 状态添加玻璃拟态效果，收起状态样式同步适配深色
- [x] T006 [P] [US1] 修改 `plus-ui/src/layout/index.vue`：根容器 div 添加 `grid-pattern` class，在 template 末尾添加两个绝对定位装饰光球（左上 `rgba(79,209,165,0.05)` 右下 `rgba(91,192,222,0.05)`，均 `blur(3xl)`）
- [x] T007 [P] [US1] 修改 `plus-ui/src/layout/components/Navbar.vue`：`.navbar` 的 `background` 改为 `rgba(17,17,24,0.8)`，添加 `backdrop-filter: blur(12px)`，`border-bottom` 改为 `1px solid rgba(42,42,62,0.3)`
- [x] T008 [US1] 修改 `plus-ui/src/layout/components/Sidebar/Logo.vue`：Logo 区域背景适配深色主题，确保文字和图标在深色背景上清晰可见

**Checkpoint**: 打开系统后框架（侧边栏 + 顶部栏 + 主背景）呈现统一的深色科技风格，导航功能正常

---

## Phase 4: User Story 2 - 深色登录页 (Priority: P2)

**Goal**: 登录页使用深色渐变背景、网格纹理和玻璃拟态表单卡片

**Independent Test**: 访问登录页面，验证深色背景、玻璃拟态卡片、输入框和按钮样式，完成登录流程

### Implementation for User Story 2

- [x] T009 [US2] 修改 `plus-ui/src/views/login.vue` 的 `<style>` 部分：`.login` 背景改为深色渐变（`linear-gradient(135deg, #0d0d1a, #111118, #1a1a2e)`）+ `.grid-pattern` 效果；`.login-form` 改为 `.glass` 玻璃拟态样式（`background: rgba(22,22,34,0.6)`、`backdrop-filter: blur(20px)`、`border: 1px solid rgba(42,42,62,0.3)`）；输入框背景改为 `rgba(22,22,34,0.8)`；登录按钮添加发光 hover 效果；底部 footer 文字颜色适配深色

**Checkpoint**: 登录页呈现深色科技风格，登录流程正常完成

---

## Phase 5: User Story 3 - Element Plus 组件深色适配 (Priority: P2)

**Goal**: 所有 Element Plus 组件在深色模式下清晰可读，交互状态有发光反馈

**Independent Test**: 打开包含表格、表单、对话框的页面，验证组件深色样式和 hover/focus 反馈

### Implementation for User Story 3

- [x] T010 [P] [US3] 修改 `plus-ui/src/assets/styles/element-ui.scss`：对话框 `.el-overlay-dialog .el-dialog` 添加毛玻璃背景（`background: rgba(22,22,34,0.9)` + `backdrop-filter: blur(12px)`）；下拉菜单 `.el-dropdown-menu` 添加毛玻璃效果；`.el-table` 深色背景确认统一；输入框 focus 状态 `.el-input__wrapper.is-focus` 添加 primary 色发光 `box-shadow: 0 0 0 2px rgba(79,209,165,0.2)`
- [x] T011 [P] [US3] 修改 `plus-ui/src/views/index.vue`：`.home` 背景适配深色，`.diagram-wrapper` 添加半透明深色背景容器（`background: rgba(22,22,34,0.5)` + `border-radius: 16px` + `padding`）确保 SVG 架构图在深色背景上清晰可见
- [x] T012 [P] [US3] 修改 `plus-ui/src/layout/components/TagsView/index.vue`：标签页样式深色适配，活跃标签使用 primary 色高亮

**Checkpoint**: 打开用户管理、菜单管理等页面，表格/表单/对话框均呈现深色风格，hover/focus 有发光反馈

---

## Phase 6: User Story 4 - 业务页面深色统一 (Priority: P3)

**Goal**: 巡检告警、巡检报告、知识助手等业务页面全部适配深色风格

**Independent Test**: 逐个打开巡检告警、巡检报告、知识助手页面，验证深色适配

### Implementation for User Story 4

- [x] T013 [P] [US4] 验证并微调 `plus-ui/src/views/inspection/alarm/index.vue` 巡检告警页面：检查搜索面板、告警列表卡片、状态标签（严重/警告/提示）在深色模式下的显示，必要时添加深色覆盖样式
- [x] T014 [P] [US4] 验证并微调 `plus-ui/src/views/inspection/report/index.vue` 和 `detail.vue` 巡检报告页面：检查报告列表和详情页卡片在深色模式下的显示
- [x] T015 [P] [US4] 验证并微调 `plus-ui/src/views/knowledge/chat/index.vue`、`ChatDialog.vue`、`SessionList.vue` 知识助手页面：检查聊天界面气泡、会话列表、文档管理在深色模式下的显示，必要时调整气泡颜色和输入框样式
- [x] T016 [P] [US4] 验证并微调 `plus-ui/src/views/knowledge/document/index.vue` 知识文档管理页面的深色适配
- [x] T017 [US4] 全站回归验证：打开系统管理（用户/角色/菜单）、监控中心（操作日志/在线用户）、工作流等模块页面，确认无浅色残留元素，表格/表单/卡片均正确显示深色主题；显式验证完整交互链路：登录→侧边栏导航→搜索→表单提交→数据查询→对话框操作→退出

**Checkpoint**: 所有业务页面在深色主题下正确显示，无浅色残留

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: 最终打磨和跨模块优化

- [x] T018 运行 `plus-ui/` 下 `npm run lint:eslint:fix` 修复可能的 lint 问题
- [x] T019 执行 quickstart.md 中的 15 步验证清单，确认所有检查项通过（需手动浏览器验证）
- [x] T020 移动端响应式验证：在 992px 以下宽度测试侧边栏折叠、页面布局、组件显示是否正常（需手动浏览器验证）

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: 无依赖，立即开始
- **Phase 2 (Foundational)**: 依赖 Phase 1 完成 — 阻塞所有 User Story
- **Phase 3 (US1 - 框架)**: 依赖 Phase 2 完成
- **Phase 4 (US2 - 登录页)**: 依赖 Phase 2 完成，可与 Phase 3 并行
- **Phase 5 (US3 - 组件适配)**: 依赖 Phase 2 完成，可与 Phase 3/4 并行
- **Phase 6 (US4 - 业务页面)**: 依赖 Phase 3 + Phase 5 完成（需要框架和组件层先完成）
- **Phase 7 (Polish)**: 依赖所有 User Story 完成

### User Story Dependencies

- **US1 (P1)**: Phase 2 完成后可开始，无其他 Story 依赖
- **US2 (P2)**: Phase 2 完成后可开始，与 US1 独立
- **US3 (P2)**: Phase 2 完成后可开始，与 US1/US2 独立
- **US4 (P3)**: 依赖 US1（框架）+ US3（组件适配）完成

### Parallel Opportunities

- Phase 1 中 T001 和 T002 可并行（不同文件）
- Phase 2 中 T003 和 T004 可并行（不同文件）
- Phase 3 中 T005、T006、T007 可并行（不同文件）
- Phase 5 中 T010、T011、T012 可并行（不同文件）
- Phase 6 中 T013、T014、T015、T016 可并行（不同文件）

---

## Parallel Example: Phase 2 (Foundational)

```bash
# 两个核心样式文件可同时修改：
Task T003: "重写 variables.module.scss 全部 CSS 变量"
Task T004: "在 index.scss 新增效果类"
```

## Parallel Example: Phase 3 (US1)

```bash
# 三个布局文件可同时修改：
Task T005: "修改 sidebar.scss 侧边栏深色样式"
Task T006: "修改 layout/index.vue 添加网格背景"
Task T007: "修改 Navbar.vue 毛玻璃顶部栏"
```

## Parallel Example: Phase 6 (US4)

```bash
# 四个业务模块可同时验证和微调：
Task T013: "验证巡检告警页面深色适配"
Task T014: "验证巡检报告页面深色适配"
Task T015: "验证知识助手聊天页面深色适配"
Task T016: "验证知识文档管理页面深色适配"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup → 强制深色模式
2. Complete Phase 2: Foundational → Design Token + 效果类库
3. Complete Phase 3: User Story 1 → 框架深色改造
4. **STOP and VALIDATE**: 打开系统验证侧边栏、顶部栏、主背景
5. 此时系统已呈现基本的深色科技风格

### Incremental Delivery

1. Setup + Foundational → 深色 Token 就绪
2. Add US1 → 框架深色 → **MVP**
3. Add US2 → 登录页深色
4. Add US3 → Element Plus 组件适配
5. Add US4 → 业务页面统一
6. Polish → 全站验证

---

## Notes

- 所有任务仅修改 CSS/SCSS 和少量模板 class，不修改组件逻辑
- 每个 Checkpoint 后建议提交一次 git commit
- US4（业务页面）以验证为主，只有发现问题时才需要添加覆盖样式
- 如果 Element Plus 某些组件的 CSS 变量无法覆盖，使用 `:deep()` 穿透
