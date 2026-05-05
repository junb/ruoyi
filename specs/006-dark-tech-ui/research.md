# Research: 全站深色科技风 UI 改造

**Branch**: `006-dark-tech-ui` | **Date**: 2026-04-21

## 研究项 1: Element Plus CSS 变量覆盖完整性

**Decision**: 通过 `--el-*` CSS 变量 + `:deep()` 穿透组合方案

**Rationale**: Element Plus 2.x 提供了完整的 CSS 变量体系（`--el-bg-color-*`, `--el-text-color-*`, `--el-border-color-*`, `--el-fill-color-*`），足以覆盖深色主题需求。少数无法通过变量覆盖的样式（如 `el-table__row hover` 的硬编码颜色）使用 `:deep()` 穿透处理。

**Alternatives considered**:
- 直接修改 Element Plus 源码：违反"不引入新依赖"约束，升级困难
- 使用 Element Plus 官方 dark mode CSS：仅提供基础深色，无法自定义科技风效果
- 完全自定义主题 SCSS：工作量过大，且需维护版本兼容性

## 研究项 2: 玻璃拟态效果的浏览器兼容性

**Decision**: 使用 `@supports (backdrop-filter: blur(1px))` 检测，不支持的浏览器降级为 `rgba(22,22,34,0.95)` 纯色背景

**Rationale**: 目标浏览器（Chrome 90+, Firefox 90+, Safari 15+, Edge 90+）均支持 `backdrop-filter`，降级方案仅作为防御性措施。Firefox 在 103 版本之前需要 `layout.css.backdrop-filter.enabled` 标志，但从 103 起默认启用。

**Alternatives considered**:
- 不提供降级：可能导致旧浏览器显示异常
- 使用 SVG filter 模拟：性能差，代码复杂
- 使用伪元素叠加：效果不精确

## 研究项 3: SVG 架构图深色背景适配方案

**Decision**: 优先在 SVG 外层添加半透明背景容器（`background: rgba(22,22,34,0.5) + border-radius + padding`），保持 SVG 原始颜色不变

**Rationale**: 修改 SVG 内部颜色需要重新编辑文件且可能破坏图形细节。外层添加容器是最小侵入方案，且与玻璃拟态风格一致。

**Alternatives considered**:
- `filter: invert(1)` CSS 反色：会改变所有颜色，效果不可控
- 直接修改 SVG 文件中的颜色：侵入性强，后续维护困难
- 使用 CSS `mix-blend-mode`：兼容性风险

## 研究项 4: 强制深色模式的技术实现

**Decision**: 在 `settings.ts` 中设置 `dark: true`，同时在 `App.vue` 或 `main.ts` 中添加 `document.documentElement.classList.add('dark')` 作为双重保险

**Rationale**: 当前系统通过 Pinia store 的 `dark` 状态控制 `html.dark` 类。设置默认值为 `true` 后，新用户自动进入深色模式。对于已有 localStorage 缓存的老用户，需要在应用初始化时强制设置。

**Alternatives considered**:
- 仅修改 `settings.ts`：老用户 localStorage 中可能缓存了 `dark: false`
- 删除 Settings 面板中的切换控件：治标不治本
- 在 CSS 中将 `:root` 直接改为深色值而不依赖 `html.dark`：改动更大且失去 Element Plus dark mode 的原生支持

## 研究项 5: oklch 到 hex 转换的精确性

**Decision**: 使用在线转换工具和 Chrome DevTools 验证，取最近似的 hex 值

**Rationale**: 参考系统使用 oklch 色彩空间定义颜色，但 Element Plus CSS 变量使用 hex/rgb 格式。转换后需要视觉对比验证，确保主色 (#4fd1a5) 和语义色在 Element Plus 组件上的显示效果与参考系统一致。

**已确认的转换值**:
| oklch | 用途 | Hex |
|-------|------|-----|
| oklch(0.11 0.01 250) | background | #111118 |
| oklch(0.14 0.015 250) | card | #161622 |
| oklch(0.7 0.18 165) | primary | #4fd1a5 |
| oklch(0.65 0.15 200) | chart-2 | #5bc0de |
| oklch(0.75 0.12 80) | chart-3 | #e6b45e |
| oklch(0.6 0.2 300) | chart-4 | #a78bfa |
| oklch(0.7 0.15 30) | chart-5 | #e87461 |
| oklch(0.25 0.02 250) | border | #2a2a3e |
| oklch(0.6 0.02 250) | muted-foreground | #8a8a9e |
| oklch(0.09 0.01 250) | sidebar | #0d0d1a |
