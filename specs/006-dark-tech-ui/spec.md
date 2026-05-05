# Feature Specification: 全站深色科技风 UI 改造

**Feature Branch**: `006-dark-tech-ui`
**Created**: 2026-04-21
**Status**: Draft
**Input**: 参考 Nexus 控制中心设计语言，将 RuoYi-Vue-Plus 管理系统从浅色/深色双主题改为纯深色科技风格，保留 Element Plus 组件，通过全局 CSS 变量覆盖实现。

## User Scenarios & Testing *(mandatory)*

### User Story 1 - 深色框架与导航 (Priority: P1)

用户打开系统后，立即感受到统一的深色科技风格——侧边栏使用深色背景和发光活跃指示条，顶部导航栏采用毛玻璃效果，页面背景带有微妙的网格纹理。整个操作流程（导航、搜索、消息通知）在深色环境下保持清晰可读。

**Why this priority**: 框架是所有页面的容器，框架风格的改变对用户感知影响最大，且完成后所有后续页面改动都能在一致的上下文中进行。

**Independent Test**: 启动系统后，检查侧边栏、顶部栏、主内容区背景是否呈现统一的深色科技风格，导航和交互是否正常工作。

**Acceptance Scenarios**:

1. **Given** 系统已启动, **When** 用户打开任意页面, **Then** 整个界面呈现统一的深色背景（#111118），侧边栏和顶部栏使用深色主题
2. **Given** 侧边栏菜单可见, **When** 用户点击某个菜单项, **Then** 该菜单项显示 primary 色高亮背景和左侧发光指示条，其他菜单项保持静默
3. **Given** 顶部导航栏可见, **When** 用户滚动页面, **Then** 顶部栏保持毛玻璃半透明效果，内容区域在其下方可见
4. **Given** 主内容区域可见, **When** 页面加载完成, **Then** 背景呈现微妙的网格纹理

---

### User Story 2 - 深色登录页 (Priority: P2)

用户在登录页面看到深色渐变背景、网格纹理和玻璃拟态效果的登录表单卡片，输入框和按钮均适配深色风格，整体氛围与主系统一致。

**Why this priority**: 登录页是用户的第一印象，需要与改造后的主系统风格一致，但可以独立于框架改造完成。

**Independent Test**: 打开登录页面，验证深色背景、玻璃拟态卡片、输入框和按钮样式，完成登录流程。

**Acceptance Scenarios**:

1. **Given** 用户访问登录页面, **When** 页面加载完成, **Then** 显示深色渐变背景和网格纹理
2. **Given** 登录表单可见, **When** 用户查看表单卡片, **Then** 卡片呈现玻璃拟态效果（半透明背景 + 模糊 + 半透明边框）
3. **Given** 用户输入凭据, **When** 点击登录按钮, **Then** 按钮呈现 primary 色发光效果，登录流程正常完成

---

### User Story 3 - Element Plus 组件深色适配 (Priority: P2)

用户在业务页面操作时（查看表格、填写表单、打开对话框），所有 Element Plus 组件（表格、按钮、标签、输入框、对话框、下拉菜单等）均呈现深色科技风格，hover/focus 状态带有微妙的发光反馈。

**Why this priority**: 业务页面大量使用 Element Plus 组件，统一适配后用户才能在日常操作中感受到一致的视觉体验。

**Independent Test**: 打开任意包含表格、表单、对话框的页面，验证组件的深色样式和交互反馈。

**Acceptance Scenarios**:

1. **Given** 用户打开包含数据表格的页面, **When** 表格加载完成, **Then** 表头使用深色背景，行 hover 时显示半透明高亮
2. **Given** 用户点击操作按钮, **When** 按钮处于 hover 状态, **Then** 按钮显示微弱上移 + 阴影反馈
3. **Given** 用户打开对话框, **When** 对话框可见, **Then** 对话框呈现毛玻璃背景效果
4. **Given** 用户聚焦输入框, **When** 输入框获得焦点, **Then** 输入框边框显示 primary 色发光

---

### User Story 4 - 业务页面深色统一 (Priority: P3)

巡检告警、巡检报告、知识助手聊天等业务页面全部适配深色风格，卡片容器、搜索面板、状态标签等元素统一使用深色科技风设计语言。

**Why this priority**: 业务页面是用户的日常工作界面，深色适配能提升整体沉浸感，但依赖框架和组件层先完成。

**Independent Test**: 逐个打开巡检告警、巡检报告、知识助手等页面，验证卡片、表格、聊天界面的深色适配。

**Acceptance Scenarios**:

1. **Given** 用户打开巡检告警页面, **When** 页面加载完成, **Then** 搜索面板和告警列表卡片均使用深色背景和半透明边框
2. **Given** 用户打开知识助手页面, **When** 进入聊天界面, **Then** 聊天气泡和会话列表均适配深色风格
3. **Given** 用户打开任意使用 .app-container 的页面, **When** 内容区可见, **Then** 容器使用深色背景 + 深色阴影 + 半透明边框

---

### Edge Cases

- 浏览器不支持 backdrop-filter（如某些旧版浏览器）时，玻璃拟态效果降级为纯色深色背景
- 移动端响应式布局下，侧边栏折叠后的视觉效果保持一致
- Element Plus 组件的极端状态（超长文本、大量数据表格）在深色模式下保持可读性
- SVG 架构图在深色背景下的可见性（可能需要调整 SVG 颜色）

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: 系统必须始终以深色主题呈现，不提供浅色模式切换选项
- **FR-002**: 系统的配色方案必须基于统一的 Design Token（background: #111118, primary: #4fd1a5, card: #161622 等），所有页面保持色彩一致
- **FR-003**: 侧边栏必须使用深色背景，活跃菜单项显示 primary 色高亮背景和左侧发光指示条
- **FR-004**: 顶部导航栏必须呈现毛玻璃效果（半透明背景 + 模糊 + 半透明底部边框）
- **FR-005**: 页面主背景必须带有微妙的网格纹理
- **FR-006**: 所有卡片容器必须使用深色背景 + 深色阴影 + 半透明边框的统一风格
- **FR-007**: 登录页面必须使用深色渐变背景、网格纹理和玻璃拟态登录表单
- **FR-008**: 所有 Element Plus 组件（按钮、输入框、表格、标签、对话框、下拉菜单）必须在深色模式下保持清晰可读和交互反馈
- **FR-009**: hover/focus 状态必须提供发光反馈效果（primary 色辉光或阴影变化）
- **FR-010**: 玻璃拟态效果必须在支持的浏览器中正确呈现，不支持的浏览器降级为纯色深色背景
- **FR-011**: 首页架构图必须在深色背景上保持清晰可见
- **FR-012**: 原有系统的所有交互功能（导航、搜索、消息通知、用户下拉、租户切换等）必须保持正常工作
- **FR-013**: 系统的布局设置面板中必须移除主题/暗黑模式切换选项

### Design Token 体系

系统必须建立以下 Design Token 层级：

| Token 类别 | 变量示例 | 说明 |
|------------|----------|------|
| 背景色 | background, card, muted | 页面和容器背景 |
| 前景色 | foreground, muted-foreground | 文字颜色 |
| 主色 | primary | 交互高亮和关键操作 |
| 语义色 | destructive, success, warning, info | 状态反馈 |
| 图表色 | chart-1 ~ chart-5 | 数据可视化 |
| 边框 | border, input | 分隔线 |
| 圆角 | radius-sm, radius-md, radius-lg | 组件圆角 |
| 阴影 | shadow-sm, shadow-md, shadow-lg | 层级深度 |
| 效果 | glass-bg, glass-border, glow-primary | 玻璃拟态和发光 |

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 用户打开系统后 100% 的页面均呈现深色主题，无任何浅色残留元素
- **SC-002**: 所有 Element Plus 组件在深色模式下文字对比度达到 WCAG AA 标准（4.5:1）
- **SC-003**: 系统所有原有交互功能（登录、导航、搜索、表单提交、数据查询）100% 正常工作
- **SC-004**: 用户从打开登录页到完成一个完整操作流程（如查看一条巡检告警），视觉风格全程统一无跳变
- **SC-005**: 页面渲染性能不受影响，不引入额外的渲染延迟（效果均通过 CSS 实现）

## Assumptions

- 用户使用现代浏览器（Chrome 90+, Firefox 90+, Safari 15+, Edge 90+），支持 backdrop-filter
- 现有 Element Plus 组件的 CSS 变量体系足够覆盖深色主题需求，无需修改组件源码
- SVG 架构图在深色背景上的可见性可以通过调整 SVG 内部颜色或添加背景层解决
- 移动端用户对深色主题的接受度与桌面端一致
- 系统管理员的布局设置面板中的其他选项（tagsView、fixedHeader 等）保持不变，仅移除主题切换
