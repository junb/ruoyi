# Implementation Plan: 全站深色科技风 UI 改造

**Branch**: `006-dark-tech-ui` | **Date**: 2026-04-21 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/006-dark-tech-ui/spec.md`

## Summary

将 RuoYi-Vue-Plus 前端系统从浅色/深色双主题改为纯深色科技风格。通过覆盖 CSS 变量和新增效果类实现，保留 Element Plus 组件库，不引入新依赖，不修改组件逻辑。

## Technical Context

**Language/Version**: TypeScript ~5.9.3
**Primary Dependencies**: Vue 3.5.30, Element Plus 2.13.5, UnoCSS, SCSS
**Storage**: N/A（纯样式改造）
**Testing**: 手动视觉验证 + E2E 冒烟测试
**Target Platform**: 现代浏览器（Chrome 90+, Firefox 90+, Safari 15+, Edge 90+）
**Project Type**: Web application（前端样式改造）
**Performance Goals**: 不引入额外渲染延迟，CSS-only 效果
**Constraints**: 不修改组件逻辑、不引入新依赖、保留 Element Plus
**Scale/Scope**: 全站 60+ Vue 页面，12 个核心样式/布局文件需修改

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. 分层架构 | N/A | 纯前端样式改造，不涉及后端 |
| II. 统一响应与异常处理 | N/A | 不涉及 API 变更 |
| III. 多租户数据隔离 | N/A | 不涉及数据层 |
| IV. 注解驱动开发 | N/A | 不涉及后端注解 |
| V. 安全与权限 | PASS | 样式变更不影响权限机制 |
| VI. 缓存策略 | N/A | 不涉及缓存 |
| VII. 前后端协作规范 | PASS | 前端使用 `<script setup lang="ts">` + Composition API，本次仅修改样式和少量模板 class |
| 技术栈约束 - 前端 | PASS | 保持 TypeScript + Vue 3 + Element Plus + UnoCSS，不引入新依赖 |
| 禁止事项 | PASS | 不手动注册组件、不硬编码密钥 |

**Gate Result**: PASS — 所有适用的原则均满足。

## Project Structure

### Documentation (this feature)

```text
specs/006-dark-tech-ui/
├── plan.md              # 本文件
├── research.md          # Phase 0 研究输出
├── quickstart.md        # Phase 1 快速验证指南
└── tasks.md             # Phase 2 任务列表（/speckit.tasks 生成）
```

### Source Code (repository root)

```text
plus-ui/src/
├── assets/styles/
│   ├── variables.module.scss    # [修改] 重写全部 CSS 变量为深色科技风
│   ├── index.scss               # [修改] 新增 glass/glow/grid 效果类和滚动条
│   ├── sidebar.scss             # [修改] 深色侧边栏 + 发光指示条
│   ├── element-ui.scss          # [修改] 毛玻璃对话框/下拉、focus 发光
│   ├── btn.scss                 # [审查] 按钮深色适配
│   └── ruoyi.scss               # [审查] 通用工具类深色适配
├── layout/
│   ├── index.vue                # [修改] 添加网格背景 + 装饰光晕
│   └── components/
│       ├── Navbar.vue           # [修改] 毛玻璃顶部栏
│       ├── Settings/index.vue   # [修改] 移除主题切换选项
│       ├── Sidebar/Logo.vue     # [修改] 深色 Logo 区域
│       └── TagsView/index.vue   # [审查] 深色标签页适配
├── views/
│   ├── login.vue                # [修改] 深色登录页 + 玻璃拟态
│   ├── index.vue                # [修改] 深色首页卡片
│   ├── inspection/              # [审查] 巡检页面深色验证
│   └── knowledge/               # [审查] 知识助手深色验证
└── settings.ts                  # [修改] 强制 dark: true
```

**Structure Decision**: 不创建新文件/目录，仅修改现有文件。所有样式效果类添加到已有的 `index.scss` 中。

## Implementation Phases

### Phase 1: P0 核心框架（6 个文件）

**目标**: 侧边栏、顶部栏、主布局全部呈现深色科技风格

1. **`settings.ts`** — 强制深色模式
   - 将 `dark: false` 改为 `dark: true`
   - 在 Settings 面板中移除暗黑模式切换控件

2. **`variables.module.scss`** — 全量 CSS 变量重写
   - `:root` 改为深色默认值：background #111118, card #161622, primary #4fd1a5, border #2a2a3e
   - 覆盖 Element Plus 全套 token：`--el-bg-color`, `--el-bg-color-page`, `--el-bg-color-overlay`, `--el-text-color-*`, `--el-border-color-*`, `--el-fill-color-*`
   - 新增自定义属性：`--glass-bg`, `--glass-border`, `--glow-primary`, `--glow-secondary`
   - 语义色适配：success → #4fd1a5 系, warning → #e6b45e 系, danger → #e87461 系, info → #8a8a9e 系

3. **`index.scss`** — 新增效果类
   - `.glass`: 玻璃拟态（rgba 背景 + backdrop-blur + rgba 边框）
   - `.glass-subtle`: 轻量版玻璃拟态
   - `.glow-sm`, `.glow`: 发光效果（box-shadow）
   - `.grid-pattern`: 网格纹理背景
   - `.gradient-border`: 渐变边框（mask 实现）
   - `.animate-pulse-glow`: 呼吸发光动画
   - 自定义深色滚动条

4. **`sidebar.scss`** — 侧边栏深色改造
   - 菜单背景：深色渐变 + 半透明边框
   - 活跃菜单项：primary 色半透明背景 + 左侧 2px 发光指示条
   - hover 状态：玻璃拟态效果
   - 收起状态样式同步适配

5. **`layout/index.vue`** — 主布局装饰
   - 根容器添加 `.grid-pattern` class
   - 添加两个绝对定位的模糊光球装饰（左上 primary, 右下 chart-2）

6. **`Navbar.vue`** — 毛玻璃顶部栏
   - 背景改为 `rgba(17,17,24,0.8) + backdrop-filter: blur(12px)`
   - 底部边框改为 `rgba(42,42,62,0.3)`

### Phase 2: P1 关键页面（5 个文件）

**目标**: 登录页、首页、Element Plus 组件全面深色适配

7. **`login.vue`** — 深色登录页
   - 背景改为深色渐变 + `.grid-pattern`
   - 登录表单卡片改为 `.glass` 玻璃拟态
   - 输入框背景适配深色
   - 登录按钮添加 `.glow-sm` 发光效果

8. **`index.vue`** — 深色首页
   - `.app-container` 自动继承深色样式
   - 检查 SVG 架构图在深色背景上的可见性，必要时调整

9. **`element-ui.scss`** — Element Plus 深色微调
   - 对话框 (`.el-overlay-dialog .el-dialog`): 毛玻璃背景
   - 下拉菜单 (`.el-dropdown-menu`): 毛玻璃背景
   - 表格 (`.el-table`): 深色背景统一
   - 输入框 focus: primary 色发光 box-shadow

10. **`Settings/index.vue`** — 移除主题切换
    - 隐藏暗黑模式开关控件
    - 隐藏主题颜色选择器（改为固定 primary 色）

11. **`Sidebar/Logo.vue`** — 深色 Logo 区域
    - Logo 区域背景适配深色主题

### Phase 3: P2 业务页面验证

**目标**: 确保所有业务页面在深色主题下正确显示

12. **巡检模块** (`inspection/*`) — 验证并微调
    - 告警状态标签颜色适配
    - 报告详情页卡片适配
    - 搜索面板适配

13. **知识助手** (`knowledge/*`) — 验证并微调
    - 聊天气泡样式适配
    - 会话列表适配
    - 文档管理页面适配

14. **全局回归** — 全站页面视觉验证
    - 系统管理、监控、工作流等模块的表格/表单/卡片验证
    - 确认无浅色残留元素

## Risk Mitigation

| 风险 | 缓解措施 |
|------|----------|
| Element Plus 部分 CSS 变量无法覆盖 | 优先使用 `--el-*` 变量，必要时用 `:deep()` 穿透 |
| backdrop-filter 不支持 | graceful degradation — `@supports` 检测，降级为纯色背景 |
| SVG 架构图在深色背景不可见 | 预留调整方案：invert filter 或添加半透明背景层 |
| 移动端样式异常 | 响应式断点测试，确保 sidebar 折叠后一致 |
| 用户习惯改变 | 无渐进方案（用户明确选择纯深色），但保留所有功能不变 |

## Complexity Tracking

无 Constitution 违反，无需记录。
