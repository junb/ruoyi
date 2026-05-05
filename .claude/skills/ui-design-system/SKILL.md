---
name: ruoyi-element-plus
description: |
  Element Plus 设计规范 Skill — 当需要设计 RuoYi-Vue-Plus 管理后台页面、编写 HTML 设计稿、检查 Element Plus 组件规范、输出 UI 设计规范文档时触发。
---

# Element Plus 设计规范 Skill

> 适用项目：RuoYi-Vue-Plus 5.6.0
> UI 组件库：Element Plus 2.13.5
> CSS 方案：UnoCSS（原子化 CSS）
> 挂载角色：UI 设计师（ui-designer）

---

## 概述

本 Skill 是 UI 设计师 Agent 的核心设计参考，定义了基于 Element Plus 的完整设计体系。包含设计 Token、组件使用规范、HTML 设计稿模板、页面检查清单，以及 RuoYi 管理后台特有的设计模式。

**使用原则：**
1. **组件优先**：所有 UI 元素优先使用 Element Plus 原生组件，不自造轮子
2. **一致至上**：与 RuoYi-Vue-Plus 已有页面风格保持完全一致
3. **可渲染验证**：设计稿必须是可直接浏览器打开的 HTML 文件

---

## 一、设计 Token

### 1.1 颜色体系

#### 主色（Primary）

| Token | 色值 | 用途 |
|-------|------|------|
| `--el-color-primary` | `#409eff` | 主按钮、链接、激活状态 |
| `--el-color-primary-light-3` | `#79bbff` | hover 状态 |
| `--el-color-primary-light-5` | `#a0cfff` | active/pressed 状态 |
| `--el-color-primary-light-7` | `#c6e2ff` | 浅背景 |
| `--el-color-primary-light-8` | `#d9ecff` | 禁用背景 |
| `--el-color-primary-light-9` | `#ecf5ff` | 极浅背景 |
| `--el-color-primary-dark-2` | `#337ecc` | 深色变体 |

#### 功能色

| 类型 | 色值 | CSS 变量 | 用途 |
|------|------|----------|------|
| 成功 Success | `#67c23a` | `--el-color-success` | 成功状态、启用标签 |
| 警告 Warning | `#e6a23c` | `--el-color-warning` | 警告状态、待处理标签 |
| 危险 Danger | `#f56c6c` | `--el-color-danger` | 错误状态、删除操作、停用标签 |
| 信息 Info | `#909399` | `--el-color-info` | 中性信息、禁用状态 |

#### 中性色

| 用途 | 色值 | 说明 |
|------|------|------|
| 正文主色 | `#303133` | `--el-text-color-primary` |
| 正文次色 | `#606266` | `--el-text-color-regular` |
| 辅助文字 | `#909399` | `--el-text-color-secondary` |
| 占位文字 | `#a8abb2` | `--el-text-color-placeholder` |
| 页面背景 | `#f5f7fa` | `--el-bg-color-page` |
| 卡片背景 | `#ffffff` | `--el-bg-color` |
| 边框颜色 | `#ebeef5` | `--el-border-color` |
| 分割线 | `#dcdfe6` | `--el-border-color-light` |

### 1.2 字体体系

```css
--el-font-size-extra-large: 20px;    /* 大标题 */
--el-font-size-large: 18px;          /* 标题 */
--el-font-size-medium: 16px;         /* 正文（默认） */
--el-font-size-base: 14px;           /* 常规文字 */
--el-font-size-small: 13px;          /* 辅助文字 */
--el-font-size-extra-small: 12px;    /* 标签、注释 */

--el-font-weight-primary: 500;       /* 标题加粗 */
--el-font-weight-regular: 400;       /* 正文常规 */
--el-font-line-height-base: 1.5;     /* 行高 */
```

**字体层级规范：**

| 层级 | 大小 | 字重 | 用途 |
|------|------|------|------|
| H1 | 20px | 600 | 页面标题 |
| H2 | 18px | 600 | 区块标题 |
| H3 | 16px | 500 | 卡片标题 |
| Body | 14px | 400 | 正文、表格内容 |
| Caption | 12px | 400 | 标签、注释、时间戳 |

### 1.3 间距体系

```css
/* 基准单位 4px */
--el-spacing-xs: 4px;
--el-spacing-sm: 8px;
--el-spacing-md: 12px;
--el-spacing-lg: 16px;
--el-spacing-xl: 20px;
--el-spacing-xxl: 24px;
--el-spacing-xxxl: 32px;
```

**常用间距场景：**

| 场景 | UnoCSS 类 | 像素值 |
|------|-----------|--------|
| 页面内边距 | `p-2` | 8px |
| 搜索区与表格区间距 | `mb-[10px]` | 10px |
| 卡片内边距 | Element Plus 默认 | 20px |
| 表单项间距 | Element Plus 默认 | 18px |
| 按钮间距 | `mr-[10px]` | 10px |
| 卡片标题与内容 | Element Plus 默认 | 12px |

### 1.4 圆角

| 用途 | 圆角值 | CSS 变量 |
|------|--------|----------|
| 默认圆角 | `4px` | `--el-border-radius-base` |
| 小圆角 | `2px` | `--el-border-radius-small` |
| 大圆角 | `8px` | `--el-border-radius-round` |
| 圆形 | `100%` | `--el-border-radius-circle` |
| 卡片圆角 | `4px` | el-card 默认 |
| 按钮/输入框 | `4px` | Element Plus 默认 |

### 1.5 阴影

| 级别 | 值 | 用途 |
|------|-----|------|
| 基础阴影 | `0 2px 12px 0 rgba(0,0,0,0.1)` | el-card 默认 |
| 悬浮阴影 | `0 2px 12px 0 rgba(0,0,0,0.1)` | `shadow="hover"` |
| 弹窗阴影 | Element Plus 默认 | el-dialog |
| 抽屉阴影 | Element Plus 默认 | el-drawer |

**RuoYi 统一使用 `shadow="hover"`**。

### 1.6 暗色模式适配

Element Plus 原生支持暗色模式，通过 HTML 根节点添加 `class="dark"` 开启：

```html
<html class="dark">
```

**暗色模式下的关键调整：**

| 元素 | 亮色 | 暗色 |
|------|------|------|
| 页面背景 | `#f5f7fa` | `#0a0a0a` |
| 卡片背景 | `#ffffff` | `#141414` |
| 正文颜色 | `#303133` | `#e5eaf3` |
| 边框颜色 | `#ebeef5` | `#4c4d4f` |
| 表格边框 | `#ebeef5` | `#4c4d4f` |

**设计稿中的暗色模式处理：**
- 不需要单独出暗色设计稿，Element Plus 自动适配
- 自定义颜色需使用 CSS 变量而非硬编码
- UnoCSS 支持 `dark:` 前缀：`dark:bg-gray-900`

---

## 二、组件规范

### 2.1 布局组件

#### ElContainer / ElAside / ElHeader / ElMain

RuoYi 管理后台的整体布局结构：

```
┌─────────────────────────────────────────┐
│              ElHeader（顶部导航）          │
├──────────┬──────────────────────────────┤
│          │                              │
│  ElAside │       ElMain（内容区）         │
│ （侧边栏）│                              │
│  220px   │     动态路由页面               │
│          │                              │
│          │                              │
└──────────┴──────────────────────────────┘
```

**设计要点：**
- 侧边栏宽度：220px（可折叠至 64px）
- 内容区使用路由占位，不硬编码布局
- 页面内容区外层统一使用 `<div class="p-2">`

#### ElCard

```html
<el-card shadow="hover">
  <template #header>
    <span>卡片标题</span>
  </template>
  <!-- 卡片内容 -->
</el-card>
```

**使用规范：**
- 所有内容区块使用 `<el-card shadow="hover">`
- 需要标题时使用 `#header` 插槽
- 列表页的搜索区和表格区分别用独立的 `el-card`

#### ElMenu

- 侧边栏菜单由 RuoYi 框架动态渲染，不需要手动编写
- 菜单项图标使用 Element Plus 图标或自定义 SVG
- 支持菜单折叠（`collapse` 属性）

#### ElBreadcrumb

```html
<el-breadcrumb separator="/">
  <el-breadcrumb-item :to="{ path: '/' }">首页</el-breadcrumb-item>
  <el-breadcrumb-item>模块名称</el-breadcrumb-item>
  <el-breadcrumb-item>页面名称</el-breadcrumb-item>
</el-breadcrumb>
```

**使用规范：**
- 面包屑放在页面顶部（内容区上方）
- RuoYi 框架通常在标签页中显示，面包屑按需使用

### 2.2 数据展示组件

#### ElTable

```html
<el-table
  v-loading="loading"
  :data="list"
  border
  @selection-change="handleSelectionChange"
>
  <el-table-column type="selection" width="55" align="center" />
  <el-table-column label="ID" prop="id" width="80" align="center" />
  <el-table-column label="名称" prop="name" min-width="150" align="left" :show-overflow-tooltip="true" />
  <el-table-column label="状态" prop="status" width="100" align="center">
    <template #default="scope">
      <el-tag :type="scope.row.status === '0' ? 'success' : 'info'">
        {{ scope.row.status === '0' ? '启用' : '停用' }}
      </el-tag>
    </template>
  </el-table-column>
  <el-table-column label="创建时间" prop="createTime" width="180" align="center">
    <template #default="scope">
      <span>{{ parseTime(scope.row.createTime) }}</span>
    </template>
  </el-table-column>
  <el-table-column label="操作" width="180" align="center" class-name="small-padding fixed-width">
    <template #default="scope">
      <el-tooltip content="编辑" placement="top">
        <el-button v-hasPermi="['module:feature:edit']" link type="primary" icon="Edit" @click="handleEdit(scope.row)" />
      </el-tooltip>
      <el-tooltip content="删除" placement="top">
        <el-button v-hasPermi="['module:feature:remove']" link type="primary" icon="Delete" @click="handleDelete(scope.row)" />
      </el-tooltip>
    </template>
  </el-table-column>
</el-table>
```

**列定义规范：**

| 列类型 | 对齐方式 | 宽度 | 说明 |
|--------|---------|------|------|
| 选择框 | center | 55 | type="selection" |
| ID / 序号 | center | 80-100 | 固定宽度 |
| 状态 / 标签 | center | 100 | 使用 el-tag |
| 时间 | center | 180 | 格式化为 YYYY-MM-DD HH:mm:ss |
| 操作 | center | 180 | 使用 link 按钮组 |
| 名称 / 描述 | left | min-width | show-overflow-tooltip |

**必须遵守：**
- 所有表格使用 `border` 属性
- 长文本列必须加 `:show-overflow-tooltip="true"`
- 状态列使用 `<el-tag>` 渲染
- 操作列按钮使用 `link` 模式 + `el-tooltip`

#### ElPagination（RuoYi 封装）

```html
<pagination
  v-show="total > 0"
  v-model:page="queryParams.pageNum"
  v-model:limit="queryParams.pageSize"
  :total="total"
  @pagination="getList"
/>
```

**使用规范：**
- 使用 RuoYi 封装的 `<pagination>` 组件，非 Element Plus 原生
- 放在 el-table 下方、el-card 内部
- `v-show="total > 0"` 控制显示

#### ElDescriptions

```html
<el-descriptions :column="2" border>
  <el-descriptions-item label="名称">{{ detail.name }}</el-descriptions-item>
  <el-descriptions-item label="状态">
    <el-tag :type="detail.status === '0' ? 'success' : 'info'">
      {{ detail.status === '0' ? '启用' : '停用' }}
    </el-tag>
  </el-descriptions-item>
  <el-descriptions-item label="创建时间">{{ detail.createTime }}</el-descriptions-item>
  <el-descriptions-item label="备注">{{ detail.remark }}</el-descriptions-item>
</el-descriptions>
```

**使用规范：**
- 详情页信息展示使用 `el-descriptions` + `border`
- `:column` 根据内容调整（通常 2 或 3）
- 状态字段使用 `el-tag` 渲染

#### ElTag

**状态标签颜色映射：**

| 状态值 | tag type | 显示文本 |
|--------|----------|---------|
| 启用 / 正常 / 成功 | `success` | 对应文本 |
| 停用 / 禁用 / 失败 | `info` 或 `danger` | 对应文本 |
| 待处理 / 审核中 | `warning` | 对应文本 |
| 异常 / 错误 | `danger` | 对应文本 |

#### ElBadge

```html
<el-badge :value="12" class="ml-3">
  <el-button>消息</el-button>
</el-badge>
```

**使用场景：**
- 通知数量标记
- 待办事项数量

### 2.3 表单组件

#### ElForm

```html
<el-form ref="formRef" :model="form" :rules="rules" label-width="100px">
  <el-form-item label="名称" prop="name">
    <el-input v-model="form.name" placeholder="请输入名称" maxlength="50" show-word-limit />
  </el-form-item>
  <el-form-item label="状态" prop="status">
    <el-radio-group v-model="form.status">
      <el-radio value="0">启用</el-radio>
      <el-radio value="1">停用</el-radio>
    </el-radio-group>
  </el-form-item>
  <el-form-item label="备注" prop="remark">
    <el-input v-model="form.remark" type="textarea" :rows="3" placeholder="请输入备注" />
  </el-form-item>
</el-form>
```

**表单规范：**
- 弹窗表单 `label-width="100px"` 或 `80px`
- 抽屉表单 `label-width="100px"`
- 必填字段 `prop` 绑定 + `rules` 校验
- 输入框加 `placeholder` + `maxlength` + `show-word-limit`（按需）

#### ElInput

| 场景 | 属性配置 |
|------|---------|
| 普通输入 | `placeholder`, `maxlength`, `clearable` |
| 文本域 | `type="textarea"`, `:rows="3"` |
| 密码 | `type="password"`, `show-password` |
| 数字 | `type="number"` 或使用 el-input-number |
| 搜索框（列表页） | `placeholder`, `clearable` |

#### ElSelect

```html
<el-select v-model="form.type" placeholder="请选择类型" clearable>
  <el-option
    v-for="dict in dict_type"
    :key="dict.value"
    :label="dict.label"
    :value="dict.value"
  />
</el-select>
```

**使用规范：**
- 下拉选项优先使用字典数据（`dict`）
- 加 `clearable` 允许清空
- 搜索表单中的下拉加 `placeholder="全部"`

#### ElDatePicker

```html
<!-- 单日期 -->
<el-date-picker v-model="form.date" type="date" placeholder="选择日期" value-format="YYYY-MM-DD" />

<!-- 日期范围（搜索表单） -->
<el-date-picker
  v-model="dateRange"
  type="daterange"
  range-separator="-"
  start-placeholder="开始日期"
  end-placeholder="结束日期"
  value-format="YYYY-MM-DD"
/>
```

**使用规范：**
- 搜索表单中的日期范围使用 `daterange` 类型
- `value-format` 指定提交格式
- 范围选择绑定数组，提交时拆分为 `params.beginTime` / `params.endTime`

#### ElSwitch

```html
<el-switch
  v-model="form.status"
  active-value="0"
  inactive-value="1"
  inline-prompt
  active-text="启用"
  inactive-text="停用"
/>
```

### 2.4 反馈组件

#### ElDialog

```html
<el-dialog :title="dialogTitle" v-model="dialogVisible" width="680px" append-to-body>
  <el-form ref="formRef" :model="form" :rules="rules" label-width="100px">
    <!-- 表单内容 -->
  </el-form>
  <template #footer>
    <div class="dialog-footer">
      <el-button type="primary" @click="submitForm">确 定</el-button>
      <el-button @click="cancel">取 消</el-button>
    </div>
  </template>
</el-dialog>
```

**弹窗规范：**
- 宽度：单列表单 `500px`，双列表单 `680px`，宽表单 `780px`
- 必须 `append-to-body`
- 按钮顺序：确定（左）→ 取消（右）
- 标题：新增时"添加XXX"，编辑时"修改XXX"

#### ElDrawer

```html
<el-drawer :title="drawerTitle" v-model="drawerVisible" size="600px">
  <el-descriptions :column="1" border>
    <!-- 详情内容 -->
  </el-descriptions>
</el-drawer>
```

**抽屉规范：**
- 宽度：详情查看 `500px-600px`，表单编辑 `600px-700px`
- 适合内容较多、需要较长滚动区域的场景

**弹窗 vs 抽屉选择标准：**

| 维度 | ElDialog | ElDrawer |
|------|----------|----------|
| 表单字段 | ≤6 个字段 | >6 个字段 |
| 内容量 | 少量信息 | 较多信息，需滚动 |
| 使用场景 | 新增、编辑表单 | 详情查看、复杂表单 |
| 宽度 | 500-780px | 500-700px |

#### ElMessage / ElMessageBox

**RuoYi 封装方式（优先使用）：**

```typescript
// 成功提示
proxy?.$modal.msgSuccess("操作成功");

// 错误提示
proxy?.$modal.msgError("操作失败");

// 确认弹窗
proxy?.$modal.confirm("确认删除该记录？").then(() => {
  // 确认操作
}).catch(() => {});
```

#### ElNotification

```html
<el-button @click="notify">通知</el-button>
```

```typescript
ElNotification({
  title: '提示',
  message: '这是一条通知消息',
  type: 'success',  // success / warning / info / error
  duration: 3000,
});
```

**使用规范：**
- 简单操作反馈 → `ElMessage`（RuoYi 封装的 `$modal.msgSuccess/msgError`）
- 需要用户确认 → `ElMessageBox`（RuoYi 封装的 `$modal.confirm`）
- 重要通知需持续展示 → `ElNotification`

### 2.5 导航组件

#### ElTabs

```html
<el-tabs v-model="activeTab" @tab-click="handleTabClick">
  <el-tab-pane label="基本信息" name="basic">
    <!-- 基本信息内容 -->
  </el-tab-pane>
  <el-tab-pane label="详细信息" name="detail">
    <!-- 详细信息内容 -->
  </el-tab-pane>
</el-tabs>
```

**使用场景：**
- 详情页内多区块切换
- 设置页面分类展示

#### ElDropdown

```html
<el-dropdown @command="handleCommand">
  <el-button>
    更多操作<el-icon class="el-icon--right"><arrow-down /></el-icon>
  </el-button>
  <template #dropdown>
    <el-dropdown-menu>
      <el-dropdown-item command="export">导出</el-dropdown-item>
      <el-dropdown-item command="import">导入</el-dropdown-item>
      <el-dropdown-item command="batchDelete" divided>批量删除</el-dropdown-item>
    </el-dropdown-menu>
  </template>
</el-dropdown>
```

---

## 三、HTML 设计稿模板

### 3.1 完整 HTML 基础模板

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>[页面名称] - 设计稿</title>
  <!-- Element Plus CDN -->
  <link rel="stylesheet" href="https://unpkg.com/element-plus/dist/index.css">
  <!-- Element Plus Icons -->
  <script src="https://unpkg.com/@element-plus/icons-vue"></script>
  <!-- Vue 3 -->
  <script src="https://unpkg.com/vue@3/dist/vue.global.prod.js"></script>
  <!-- Element Plus -->
  <script src="https://unpkg.com/element-plus"></script>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: #f5f7fa;
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
      padding: 20px;
    }
    .page-container {
      max-width: 1400px;
      margin: 0 auto;
    }
    .page-title {
      font-size: 20px;
      font-weight: 600;
      color: #303133;
      margin-bottom: 16px;
    }
    /* 设计稿水印 */
    .page-container::before {
      content: '设计稿 - 仅供参考';
      position: fixed;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%) rotate(-30deg);
      font-size: 48px;
      color: rgba(0,0,0,0.03);
      pointer-events: none;
      white-space: nowrap;
    }
  </style>
</head>
<body>
  <div id="app">
    <div class="page-container">
      <!-- 在此编写设计稿内容 -->
    </div>
  </div>
  <script>
    const { createApp, ref, reactive } = Vue;

    const app = createApp({
      setup() {
        // 在此编写数据和逻辑
        return {};
      }
    });

    // 注册 Element Plus
    app.use(ElementPlus);

    // 注册所有图标（CDN 模式）
    for (const [key, component] of Object.entries(ElementPlusIconsVue)) {
      app.component(key, component);
    }

    app.mount('#app');
  </script>
</body>
</html>
```

### 3.2 列表页面设计稿模板

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>[模块名称]管理 - 设计稿</title>
  <link rel="stylesheet" href="https://unpkg.com/element-plus/dist/index.css">
  <script src="https://unpkg.com/@element-plus/icons-vue"></script>
  <script src="https://unpkg.com/vue@3/dist/vue.global.prod.js"></script>
  <script src="https://unpkg.com/element-plus"></script>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { background: #f5f7fa; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; padding: 20px; }
    .page-container { max-width: 1400px; margin: 0 auto; }
    .search-area { margin-bottom: 10px; }
    .toolbar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; }
    .toolbar-left { display: flex; gap: 8px; }
  </style>
</head>
<body>
  <div id="app">
    <div class="page-container">
      <!-- 搜索区域 -->
      <div class="search-area">
        <el-card shadow="hover">
          <el-form :inline="true" :model="queryParams">
            <el-form-item label="名称">
              <el-input v-model="queryParams.name" placeholder="请输入名称" clearable style="width: 200px;" />
            </el-form-item>
            <el-form-item label="状态">
              <el-select v-model="queryParams.status" placeholder="全部" clearable style="width: 120px;">
                <el-option label="启用" value="0" />
                <el-option label="停用" value="1" />
              </el-select>
            </el-form-item>
            <el-form-item label="创建时间">
              <el-date-picker
                v-model="queryParams.dateRange"
                type="daterange"
                range-separator="-"
                start-placeholder="开始日期"
                end-placeholder="结束日期"
                style="width: 240px;"
              />
            </el-form-item>
            <el-form-item>
              <el-button type="primary" :icon="Search" @click="handleQuery">搜索</el-button>
              <el-button :icon="Refresh" @click="resetQuery">重置</el-button>
            </el-form-item>
          </el-form>
        </el-card>
      </div>

      <!-- 表格区域 -->
      <el-card shadow="hover">
        <!-- 工具栏 -->
        <div class="toolbar">
          <div class="toolbar-left">
            <el-button type="primary" plain :icon="Plus">新增</el-button>
            <el-button type="danger" plain :icon="Delete">删除</el-button>
            <el-button type="info" plain :icon="Download">导出</el-button>
          </div>
          <div>
            <el-button :icon="Search" circle />
            <el-button :icon="Refresh" circle />
          </div>
        </div>

        <!-- 数据表格 -->
        <el-table :data="tableData" border style="width: 100%;">
          <el-table-column type="selection" width="55" align="center" />
          <el-table-column label="ID" prop="id" width="80" align="center" />
          <el-table-column label="名称" prop="name" min-width="150" align="left" show-overflow-tooltip />
          <el-table-column label="类型" prop="type" width="120" align="center" />
          <el-table-column label="状态" prop="status" width="100" align="center">
            <template #default="scope">
              <el-tag :type="scope.row.status === '0' ? 'success' : 'info'">
                {{ scope.row.status === '0' ? '启用' : '停用' }}
              </el-tag>
            </template>
          </el-table-column>
          <el-table-column label="创建时间" prop="createTime" width="180" align="center" />
          <el-table-column label="操作" width="180" align="center" class-name="small-padding fixed-width">
            <template #default="scope">
              <el-tooltip content="编辑" placement="top">
                <el-button link type="primary" :icon="Edit" />
              </el-tooltip>
              <el-tooltip content="删除" placement="top">
                <el-button link type="primary" :icon="Delete" />
              </el-tooltip>
              <el-tooltip content="详情" placement="top">
                <el-button link type="primary" :icon="View" />
              </el-tooltip>
            </template>
          </el-table-column>
        </el-table>

        <!-- 分页 -->
        <div style="display: flex; justify-content: flex-end; margin-top: 16px;">
          <el-pagination
            v-model:current-page="queryParams.pageNum"
            v-model:page-size="queryParams.pageSize"
            :page-sizes="[10, 20, 50, 100]"
            :total="total"
            layout="total, sizes, prev, pager, next, jumper"
            background
          />
        </div>
      </el-card>
    </div>
  </div>
  <script>
    const { createApp, ref, reactive } = Vue;
    const { Search, Refresh, Plus, Delete, Download, Edit, View } = ElementPlusIconsVue;

    const app = createApp({
      setup() {
        const queryParams = reactive({
          name: '',
          status: '',
          dateRange: null,
          pageNum: 1,
          pageSize: 10,
        });
        const total = ref(100);

        const tableData = ref([
          { id: 1, name: '示例数据A', type: '类型1', status: '0', createTime: '2026-04-30 10:00:00' },
          { id: 2, name: '示例数据B', type: '类型2', status: '1', createTime: '2026-04-30 11:00:00' },
          { id: 3, name: '示例数据C', type: '类型1', status: '0', createTime: '2026-04-30 12:00:00' },
        ]);

        const handleQuery = () => { /* 搜索逻辑 */ };
        const resetQuery = () => {
          queryParams.name = '';
          queryParams.status = '';
          queryParams.dateRange = null;
        };

        return { queryParams, total, tableData, handleQuery, resetQuery, Search, Refresh, Plus, Delete, Download, Edit, View };
      }
    });

    app.use(ElementPlus);
    for (const [key, component] of Object.entries(ElementPlusIconsVue)) {
      app.component(key, component);
    }
    app.mount('#app');
  </script>
</body>
</html>
```

### 3.3 表单弹窗设计稿模板

```html
<!-- 弹窗内容（嵌入在列表页模板中） -->
<el-dialog title="添加[模块名称]" v-model="dialogVisible" width="680px" append-to-body>
  <el-form :model="form" :rules="rules" label-width="100px">
    <el-row :gutter="20">
      <el-col :span="12">
        <el-form-item label="名称" prop="name">
          <el-input v-model="form.name" placeholder="请输入名称" maxlength="50" show-word-limit />
        </el-form-item>
      </el-col>
      <el-col :span="12">
        <el-form-item label="类型" prop="type">
          <el-select v-model="form.type" placeholder="请选择类型" style="width: 100%;">
            <el-option label="类型A" value="a" />
            <el-option label="类型B" value="b" />
          </el-select>
        </el-form-item>
      </el-col>
    </el-row>
    <el-row :gutter="20">
      <el-col :span="12">
        <el-form-item label="排序" prop="orderNum">
          <el-input-number v-model="form.orderNum" :min="0" controls-position="right" style="width: 100%;" />
        </el-form-item>
      </el-col>
      <el-col :span="12">
        <el-form-item label="状态" prop="status">
          <el-radio-group v-model="form.status">
            <el-radio value="0">启用</el-radio>
            <el-radio value="1">停用</el-radio>
          </el-radio-group>
        </el-form-item>
      </el-col>
    </el-row>
    <el-form-item label="备注" prop="remark">
      <el-input v-model="form.remark" type="textarea" :rows="3" placeholder="请输入备注" maxlength="500" show-word-limit />
    </el-form-item>
  </el-form>
  <template #footer>
    <div style="text-align: right;">
      <el-button type="primary">确 定</el-button>
      <el-button>取 消</el-button>
    </div>
  </template>
</el-dialog>
```

### 3.4 详情页设计稿模板

```html
<!-- 详情页（抽屉模式） -->
<el-drawer title="[模块名称]详情" v-model="drawerVisible" size="600px">
  <el-descriptions :column="2" border>
    <el-descriptions-item label="ID">1</el-descriptions-item>
    <el-descriptions-item label="名称">示例数据A</el-descriptions-item>
    <el-descriptions-item label="类型">类型A</el-descriptions-item>
    <el-descriptions-item label="状态">
      <el-tag type="success">启用</el-tag>
    </el-descriptions-item>
    <el-descriptions-item label="排序">1</el-descriptions-item>
    <el-descriptions-item label="创建人">admin</el-descriptions-item>
    <el-descriptions-item label="创建时间">2026-04-30 10:00:00</el-descriptions-item>
    <el-descriptions-item label="更新时间">2026-04-30 12:00:00</el-descriptions-item>
    <el-descriptions-item label="备注" :span="2">
      这是一段备注信息，用于展示多行文本的描述。
    </el-descriptions-item>
  </el-descriptions>

  <!-- 详情页底部操作区 -->
  <template #footer>
    <div style="display: flex; justify-content: space-between;">
      <el-button type="primary" plain :icon="Edit">编辑</el-button>
      <el-button @click="drawerVisible = false">关 闭</el-button>
    </div>
  </template>
</el-drawer>
```

### 3.5 仪表盘设计稿模板

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>仪表盘 - 设计稿</title>
  <link rel="stylesheet" href="https://unpkg.com/element-plus/dist/index.css">
  <script src="https://unpkg.com/@element-plus/icons-vue"></script>
  <script src="https://unpkg.com/vue@3/dist/vue.global.prod.js"></script>
  <script src="https://unpkg.com/element-plus"></script>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { background: #f5f7fa; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; padding: 20px; }
    .page-container { max-width: 1400px; margin: 0 auto; }
    .stat-row { display: grid; grid-template-columns: repeat(4, 1fr); gap: 16px; margin-bottom: 16px; }
    .stat-card { text-align: center; padding: 20px; }
    .stat-value { font-size: 32px; font-weight: 600; color: #303133; margin-bottom: 8px; }
    .stat-label { font-size: 14px; color: #909399; }
    .chart-row { display: grid; grid-template-columns: 2fr 1fr; gap: 16px; margin-bottom: 16px; }
  </style>
</head>
<body>
  <div id="app">
    <div class="page-container">
      <!-- 统计卡片行 -->
      <div class="stat-row">
        <el-card shadow="hover" class="stat-card">
          <div class="stat-value" style="color: #409eff;">1,256</div>
          <div class="stat-label">总用户数</div>
        </el-card>
        <el-card shadow="hover" class="stat-card">
          <div class="stat-value" style="color: #67c23a;">89</div>
          <div class="stat-label">今日活跃</div>
        </el-card>
        <el-card shadow="hover" class="stat-card">
          <div class="stat-value" style="color: #e6a23c;">23</div>
          <div class="stat-label">待处理工单</div>
        </el-card>
        <el-card shadow="hover" class="stat-card">
          <div class="stat-value" style="color: #f56c6c;">5</div>
          <div class="stat-label">异常告警</div>
        </el-card>
      </div>

      <!-- 图表行 -->
      <div class="chart-row">
        <el-card shadow="hover">
          <template #header><span>趋势图</span></template>
          <div style="height: 300px; display: flex; align-items: center; justify-content: center; color: #909399;">
            [图表区域 - 使用 ECharts 等图表库渲染]
          </div>
        </el-card>
        <el-card shadow="hover">
          <template #header><span>分布图</span></template>
          <div style="height: 300px; display: flex; align-items: center; justify-content: center; color: #909399;">
            [图表区域 - 使用 ECharts 等图表库渲染]
          </div>
        </el-card>
      </div>

      <!-- 快捷操作 + 最近动态 -->
      <div class="chart-row">
        <el-card shadow="hover">
          <template #header><span>最近动态</span></template>
          <el-timeline>
            <el-timeline-item timestamp="2026-04-30 10:00" placement="top">
              <el-card shadow="never">用户张三提交了工单 #1234</el-card>
            </el-timeline-item>
            <el-timeline-item timestamp="2026-04-30 09:30" placement="top">
              <el-card shadow="never">系统完成了数据备份</el-card>
            </el-timeline-item>
            <el-timeline-item timestamp="2026-04-30 09:00" placement="top">
              <el-card shadow="never">管理员更新了配置参数</el-card>
            </el-timeline-item>
          </el-timeline>
        </el-card>
        <el-card shadow="hover">
          <template #header><span>快捷操作</span></template>
          <div style="display: grid; grid-template-columns: repeat(2, 1fr); gap: 12px;">
            <el-button type="primary" plain :icon="Plus" style="width: 100%;">新增记录</el-button>
            <el-button type="success" plain :icon="Upload" style="width: 100%;">导入数据</el-button>
            <el-button type="warning" plain :icon="Download" style="width: 100%;">导出报表</el-button>
            <el-button type="info" plain :icon="Setting" style="width: 100%;">系统设置</el-button>
          </div>
        </el-card>
      </div>
    </div>
  </div>
  <script>
    const { createApp, ref } = Vue;
    const { Plus, Upload, Download, Setting } = ElementPlusIconsVue;

    const app = createApp({ setup() { return { Plus, Upload, Download, Setting }; } });
    app.use(ElementPlus);
    for (const [key, component] of Object.entries(ElementPlusIconsVue)) {
      app.component(key, component);
    }
    app.mount('#app');
  </script>
</body>
</html>
```

---

## 四、页面设计检查清单

### 4.1 布局一致性

- [ ] 页面外层使用 `<div class="p-2">` 包裹
- [ ] 内容区域使用 `<el-card shadow="hover">` 容器
- [ ] 搜索区与表格区之间间距 `mb-[10px]`（或搜索区 margin-bottom: 10px）
- [ ] 页面最大宽度不超过 1400px（设计稿中设置）
- [ ] 表单弹窗宽度统一（500px / 680px / 780px）

### 4.2 间距规范

- [ ] 工具栏按钮间距一致（`gap: 8px` 或 `mr-[10px]`）
- [ ] 表单项之间间距使用 Element Plus 默认值
- [ ] 分页与表格之间间距 `mt-4`（16px）
- [ ] 卡片之间间距 `mb-[10px]` 或 `gap-4`

### 4.3 颜色使用

- [ ] 主操作按钮使用 `type="primary"`（蓝色）
- [ ] 危险操作（删除）使用 `type="danger"` 或 `plain`
- [ ] 状态标签颜色符合规范（success / warning / danger / info）
- [ ] 不使用 Element Plus 色板外的自定义颜色
- [ ] 文字颜色使用 CSS 变量，不硬编码

### 4.4 字体层级

- [ ] 页面标题 20px / font-weight 600
- [ ] 区块标题 18px / font-weight 600
- [ ] 正文内容 14px / font-weight 400
- [ ] 辅助文字 12px / font-weight 400
- [ ] 表格内容 14px

### 4.5 交互状态

- [ ] 所有输入框有 placeholder 提示
- [ ] 所有输入框支持 clearable（按需）
- [ ] 按钮有 loading 状态（提交操作）
- [ ] 表格有 v-loading 加载状态
- [ ] 删除操作有二次确认弹窗
- [ ] 操作按钮有 el-tooltip 文字提示
- [ ] 长文本有 show-overflow-tooltip 或省略号
- [ ] 空数据使用 el-empty 提示

### 4.6 表格规范

- [ ] 表格使用 `border` 属性
- [ ] ID/序号列居中、固定宽度
- [ ] 状态列使用 el-tag 渲染
- [ ] 时间列格式统一（YYYY-MM-DD HH:mm:ss）
- [ ] 操作列宽度 180px，使用 link 按钮
- [ ] 多选列宽度 55px，居中

### 4.7 表单规范

- [ ] 必填字段有红色星号标识
- [ ] 表单校验规则完整
- [ ] 输入框有 maxlength 限制
- [ ] 下拉选择支持清空
- [ ] 日期选择器指定 value-format
- [ ] 表单布局合理（单列 / 双列）

### 4.8 权限标记

- [ ] 新增按钮：`v-hasPermi="['module:feature:add']"`
- [ ] 编辑按钮：`v-hasPermi="['module:feature:edit']"`
- [ ] 删除按钮：`v-hasPermi="['module:feature:remove']"`
- [ ] 导出按钮：`v-hasPermi="['module:feature:export']"`
- [ ] 权限标识格式：`模块:功能:操作`（全小写）

---

## 五、RuoYi 管理后台特有设计模式

### 5.1 整体布局模式

RuoYi-Vue-Plus 采用经典的后台管理布局：

```
┌──────────────────────────────────────────────────┐
│  Logo   │  首页  系统管理  监控中心  ...    头像▼  │  ← Header（顶部导航）
├────────┼─────────────────────────────────────────┤
│        │  标签页：首页 │ 系统管理 │ ...           │
│  菜单  │─────────────────────────────────────────│
│  管理  │                                         │
│        │  页面内容区                               │
│  系统  │  <div class="p-2">                       │
│  监控  │    <el-card>...</el-card>               │
│  ...  │  </div>                                   │
│        │                                         │
├────────┤                                         │
│  折叠  │                                         │
└────────┴─────────────────────────────────────────┘
```

**设计稿中的处理：**
- 不需要模拟完整的后台布局框架
- 只设计页面内容区（`<div class="p-2">` 内的部分）
- 页面顶部可加面包屑或页面标题

### 5.2 列表页标准布局

列表页是 RuoYi 中最常见的页面类型，标准结构：

```
┌─────────────────────────────────────┐
│ 🔍 搜索区域（可折叠）                  │  ← el-card shadow="hover"
│   [名称输入] [状态下拉] [日期范围]     │
│   [搜索] [重置]                      │
├─────────────────────────────────────┤
│ [+新增] [删除] [导出]     🔍 ↻      │  ← el-card header 工具栏
├─────────────────────────────────────┤
│ ☐ │ ID │ 名称  │ 状态 │ 时间 │ 操作 │  ← el-table border
│ ☐ │ 1  │ xxx   │ 启用 │ ...  │ ✏🗑 │
│ ☐ │ 2  │ xxx   │ 停用 │ ...  │ ✏🗑 │
├─────────────────────────────────────┤
│            共 X 条  < 1 2 3 >       │  ← pagination
└─────────────────────────────────────┘
```

**关键要素（缺一不可）：**
1. 搜索区（可折叠，带动画）
2. 工具栏（左侧操作按钮 + 右侧搜索/刷新图标按钮）
3. 数据表格（border、loading、多选）
4. 分页组件（total、sizes、prev/pager/next、jumper）

### 5.3 弹窗式 vs 抽屉式表单选择标准

| 判断维度 | 弹窗 ElDialog | 抽屉 ElDrawer |
|---------|---------------|---------------|
| **表单字段数** | ≤ 6 个 | > 6 个 |
| **内容高度** | 一屏内可展示 | 需要滚动 |
| **交互复杂度** | 简单 CRUD | 复杂配置、多步骤 |
| **默认选择** | ✅ 新增 / 编辑表单 | 详情查看 |
| **参考案例** | 用户管理的新增编辑 | 系统配置的详情查看 |

### 5.4 按钮权限标记规范

RuoYi 使用 `v-hasPermi` 自定义指令控制按钮权限：

```html
<!-- 新增按钮 -->
<el-button v-hasPermi="['system:user:add']" type="primary" plain icon="Plus">新增</el-button>

<!-- 编辑按钮 -->
<el-button v-hasPermi="['system:user:edit']" link type="primary" icon="Edit" />

<!-- 删除按钮 -->
<el-button v-hasPermi="['system:user:remove']" link type="primary" icon="Delete" />

<!-- 导出按钮 -->
<el-button v-hasPermi="['system:user:export']" type="warning" plain icon="Download">导出</el-button>
```

**权限标识命名规则：**
- 格式：`{模块}:{功能}:{操作}`
- 全部小写
- 操作类型：`add` / `edit` / `remove` / `query` / `export` / `import`

### 5.5 搜索区折叠动画

RuoYi 列表页的搜索区支持展开/折叠：

```html
<transition
  :enter-active-class="animate__animated animate__fadeInDown"
  :leave-active-class="animate__animated animate__fadeOutUp"
>
  <div v-show="showSearch" class="mb-[10px]">
    <!-- 搜索表单 -->
  </div>
</transition>
```

- 默认展开（`showSearch = true`）
- 工具栏右侧有搜索图标按钮控制折叠
- 使用 animate.css 动画

### 5.6 RuoYi 特有组件

| 组件 | 说明 | 使用场景 |
|------|------|---------|
| `<right-toolbar>` | 工具栏右侧按钮组（搜索切换 + 刷新） | 列表页工具栏 |
| `<pagination>` | 封装的分页组件 | 列表页底部 |
| `<dict-tag>` | 字典标签渲染 | 表格中的字典值展示 |
| `<image-preview>` | 图片预览 | 表格中的图片展示 |
| `<file-upload>` | 文件上传 | 表单中的文件字段 |
| `<image-upload>` | 图片上传 | 表单中的图片字段 |
| `proxy?.$modal.confirm()` | 确认弹窗 | 删除等危险操作 |
| `proxy?.$modal.msgSuccess()` | 成功提示 | 操作成功反馈 |
| `proxy?.$modal.msgError()` | 错误提示 | 操作失败反馈 |
| `v-hasPermi` | 权限控制指令 | 按钮权限 |
| `parseTime()` | 时间格式化 | 时间列渲染 |

---

## 六、经验库写入规范

当设计过程中发现值得记录的经验时，写入 `doc/lessons-learned.md`，格式如下：

```markdown
### [经验标题]

- **类型：** 设计模式 / 组件用法 / 交互规范 / 布局技巧
- **场景：** [什么情况下适用]
- **经验：** [具体做法，原则性描述，不含具体页面名]
- **反例：** [常见错误做法]
```

**经验判断标准：** 去掉具体数值和页面名，这句话还能指导未来的设计决策吗？能 → 写入；不能 → 不写。

**经验示例：**

```markdown
### 状态标签颜色统一规则

- **类型：** 设计模式
- **场景：** 所有表格中的状态列展示
- **经验：** 启用/正常类状态使用 success 绿色，停用/禁用类使用 info 灰色，异常/错误类使用 danger 红色。避免不同页面对同一状态使用不同颜色。
- **反例：** 有的页面启用用蓝色，有的页面启用用绿色，导致用户困惑。
```

---

## 七、UI 设计规范文档产出检查

设计稿完成后，逐项检查：

- [ ] 所有页面都在页面清单中列出
- [ ] 每个页面有对应的 HTML 设计稿
- [ ] HTML 设计稿可直接浏览器打开渲染
- [ ] 设计稿使用了 Element Plus CDN（非本地文件）
- [ ] 表格列定义完整（对齐、宽度、特殊处理）
- [ ] 操作按钮有权限标识
- [ ] 状态标签有颜色规范
- [ ] 搜索条件组件类型正确
- [ ] 弹窗/抽屉选择合理
- [ ] 整体风格与 RuoYi 已有页面一致
