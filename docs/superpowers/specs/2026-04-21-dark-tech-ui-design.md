# 全站深色科技风 UI 改造设计文档

## 概述

参考 Nexus 控制中心（Next.js + shadcn/ui + Tailwind）的设计语言，将 RuoYi-Vue-Plus 管理系统从当前浅色/深色双主题改为纯深色科技风格。保留 Element Plus 组件库，通过全局 CSS 变量覆盖实现视觉效果统一。

## 设计语言

### 配色系统

基于参考系统的 oklch 色彩空间，转换为 hex 值适配 Element Plus CSS 变量体系：

| Token | Hex | 用途 |
|-------|-----|------|
| background | `#111118` | 页面背景 |
| foreground | `#f0f0f2` | 主文字色 |
| card | `#161622` | 卡片背景 |
| primary | `#4fd1a5` | 主色（青绿色） |
| secondary | `#1e1e30` | 次要背景 |
| muted | `#1a1a2e` | 静默区域 |
| muted-foreground | `#8a8a9e` | 次要文字 |
| border | `#2a2a3e` | 边框 |
| chart-1 | `#4fd1a5` | 图表色 1（青绿） |
| chart-2 | `#5bc0de` | 图表色 2（天蓝） |
| chart-3 | `#e6b45e` | 图表色 3（琥珀） |
| chart-4 | `#a78bfa` | 图表色 4（紫色） |
| chart-5 | `#e87461` | 图表色 5（珊瑚） |
| destructive | `#e87461` | 危险色 |

### 视觉效果

1. **玻璃拟态 (Glassmorphism)**
   - `.glass`: `background: rgba(22,22,34,0.6) + backdrop-filter: blur(20px) + border: 1px solid rgba(42,42,62,0.3)`
   - `.glass-subtle`: 更轻的透明度版本

2. **霓虹发光 (Glow)**
   - `.glow-sm`: `box-shadow: 0 0 10px rgba(79,209,165,0.4)`
   - `.glow`: `box-shadow: 0 0 20px rgba(79,209,165,0.4), 0 0 40px rgba(91,192,222,0.3)`

3. **网格纹理 (Grid Pattern)**
   - `.grid-pattern`: 40px 间距的微妙网格线 `rgba(42,42,62,0.1)`

4. **渐变边框**
   - `.gradient-border`: 使用 mask 实现 135deg 从 primary 到 chart-2 的渐变边框

### 动画效果

- `.animate-pulse-glow`: 2s 呼吸发光动画
- hover 状态统一使用 `transition: all 0.3s`

### 圆角和阴影

| Token | 值 |
|-------|-----|
| radius-sm | 8px |
| radius-md | 12px |
| radius-lg | 16px |
| shadow-sm | `0 1px 2px rgba(0,0,0,0.3), 0 6px 16px rgba(0,0,0,0.25)` |
| shadow-md | `0 8px 24px rgba(0,0,0,0.35)` |
| shadow-lg | `0 12px 32px rgba(0,0,0,0.4)` |

## 改造范围

### P0 — 核心框架

1. **`settings.ts`** — 强制 `dark: true`，移除主题切换能力
2. **`variables.module.scss`** — 重写 `:root`（改为深色默认）和 `html.dark` 的全部 CSS 变量：
   - 菜单背景色改为深色渐变
   - 表头背景、文字色适配
   - Element Plus 全套 token 覆盖（bg-color、text-color、border-color、fill-color 等）
   - 新增 glassmorphism 和 glow 相关自定义属性
3. **`index.scss`** — 新增效果类：
   - `.glass`, `.glass-subtle`, `.glow`, `.glow-sm`, `.grid-pattern`, `.gradient-border`
   - `.animate-pulse-glow` 动画
   - 自定义深色滚动条样式
4. **`sidebar.scss`** — 侧边栏改造：
   - 深色背景 + 半透明边框
   - 活跃菜单项：primary 色背景 + 左侧发光指示条
   - hover 状态：玻璃拟态效果
5. **`layout/index.vue`** — 主布局：
   - 添加 `.grid-pattern` 背景
   - 背景装饰光晕（左上/右下两个模糊光球）
6. **`Navbar.vue`** — 顶部导航栏：
   - 毛玻璃效果（backdrop-filter: blur）
   - 半透明背景 + 底部半透明边框

### P1 — 关键页面

7. **`login.vue`** — 深色登录页：
   - 深色渐变背景 + 网格纹理
   - 玻璃拟态登录表单卡片
   - 输入框深色适配
   - 按钮发光效果
8. **`index.vue`** — 深色首页（保留架构图）：
   - 深色卡片容器
   - SVG 架构图适配深色背景
9. **`element-ui.scss`** — Element Plus 深色微调：
   - 对话框毛玻璃效果
   - 下拉菜单毛玻璃效果
   - 表格深色背景统一
   - 输入框 focus 状态发光

### P2 — 业务页面

10. **巡检模块** (`inspection/*`) — 卡片、表格、状态标签深色适配
11. **知识助手** (`knowledge/*`) — 聊天界面深色适配，消息气泡样式调整
12. **全局** — 所有使用 `.app-container`、`.panel`、`.search` 的页面自动适配

## 技术约束

- **不修改组件逻辑**：所有改动仅限 CSS/SCSS 和少量模板 class 绑定
- **保留 Element Plus**：通过 CSS 变量控制主题，不替换组件
- **强制深色模式**：移除 Settings 面板中的主题切换选项
- **兼容现有功能**：所有交互功能不受影响
- **不引入新依赖**：纯 CSS 实现，不新增 npm 包

## 文件修改清单

| 文件 | 改动类型 | 优先级 |
|------|----------|--------|
| `plus-ui/src/settings.ts` | 修改 `dark: false` → `dark: true` | P0 |
| `plus-ui/src/assets/styles/variables.module.scss` | 重写全部 CSS 变量 | P0 |
| `plus-ui/src/assets/styles/index.scss` | 新增效果类和滚动条样式 | P0 |
| `plus-ui/src/assets/styles/sidebar.scss` | 深色侧边栏样式 | P0 |
| `plus-ui/src/layout/index.vue` | 添加网格背景和装饰 | P0 |
| `plus-ui/src/layout/components/Navbar.vue` | 毛玻璃顶部栏 | P0 |
| `plus-ui/src/views/login.vue` | 深色登录页 | P1 |
| `plus-ui/src/views/index.vue` | 深色首页卡片 | P1 |
| `plus-ui/src/assets/styles/element-ui.scss` | Element Plus 深色微调 | P1 |
| `plus-ui/src/layout/components/Settings/index.vue` | 移除主题切换选项 | P1 |
| `plus-ui/src/layout/components/Sidebar/Logo.vue` | 深色 Logo 区域适配 | P1 |
| 巡检/知识页面 | 视觉微调 | P2 |
