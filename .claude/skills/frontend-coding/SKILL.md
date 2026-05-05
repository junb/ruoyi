---
name: ruoyi-frontend
description: |
  RuoYi-Vue-Plus 5.6.0 前端开发规范 Skill。当需要在 RuoYi-Vue-Plus 框架下开发前端页面时触发，包括新建业务模块页面、编写 Vue 组件、API 对接、Element Plus 使用、权限控制等。
---

# RuoYi-Vue-Plus 前端开发规范 Skill

> 版本：v1.0 | 适配 RuoYi-Vue-Plus 5.6.0
> 技术栈：Vue 3 + Vite + Element Plus 2.13.5 + Pinia 3.0.4 + UnoCSS + TypeScript

---

## 概述

本 Skill 为前端工程师 Agent 提供 RuoYi-Vue-Plus 框架下的完整前端开发规范，包括：
- 新建业务模块页面的标准步骤
- API 封装、页面组件、TypeScript、Pinia、UnoCSS 等编码规范（铁律）
- 可直接使用的完整代码模板（非伪代码）
- 常见开发模式（字典回显、文件上传、权限控制等）
- 代码自验清单
- 经验库写入规范

**项目路径：** `{FRONTEND_DIR}/`

---

## 使用步骤

### 阅读顺序

1. 先阅读「编码规范（铁律）」，理解所有强制规则
2. 根据任务类型，参考「新建业务模块页面步骤」或「代码模板」
3. 编写代码
4. 使用「自验清单」逐项检查
5. 如有值得记录的经验，写入经验库

### 触发条件

- 在 RuoYi-Vue-Plus 框架下开发任何前端页面
- 新建业务模块页面
- 对接后端 API
- 使用 Element Plus 组件、字典、权限控制等功能

---

## 编码规范（铁律）

### 1. 目录结构规范

```
plus-ui/src/
├── api/{module}/{feature}/          # API 接口层
│   ├── index.ts                     # API 函数（使用 request 工具）
│   └── types.ts                     # TypeScript 类型定义
├── views/{module}/{feature}/        # 页面层
│   ├── index.vue                    # 列表页（标准 CRUD 页面）
│   ├── detail.vue                   # 详情页（如需要）
│   └── components/                  # 页面私有组件
├── components/                      # 全局公共组件（不要随意修改）
│   ├── Pagination/index.vue         # RuoYi 分页组件
│   ├── RightToolbar/index.vue       # 右侧工具栏
│   ├── DictTag/index.vue            # 字典标签
│   ├── ImageUpload/                 # 图片上传组件
│   ├── FileUpload/                  # 文件上传组件
│   ├── Editor/                      # 富文本编辑器
│   └── ...
├── store/modules/                   # Pinia Store
├── hooks/                           # 组合式函数（如 useDialog.ts）
├── directive/                       # 自定义指令（权限等）
├── utils/
│   ├── request.ts                   # Axios 封装
│   ├── ruoyi.ts                     # RuoYi 工具函数（parseTime 等）
│   └── auth.ts                      # Token 管理
├── plugins/                         # 插件（cache、modal 等）
├── types/                           # 全局 TypeScript 类型
└── router/                          # 路由配置
```

### 2. API 封装规范

| 规则 | 说明 |
|------|------|
| 请求工具 | 统一使用 `import request from '@/utils/request'` |
| 类型定义 | 在 `types.ts` 中定义接口类型 |
| 返回类型 | 列表查询 → `AxiosPromise<TableDataInfo<Vo>>`，操作 → `AxiosPromise<R<T>>` |
| URL 路径 | 与后端 `@RequestMapping` 保持一致 |
| 文件命名 | `{feature}/index.ts`（API 函数）+ `{feature}/types.ts`（类型） |
| 函数命名 | `list{Feature}`、`get{Feature}`、`add{Feature}`、`update{Feature}`、`del{Feature}` |

### 3. 页面组件规范

| 规则 | 说明 |
|------|------|
| 语法 | `<script setup name="XxxPage" lang="ts">` |
| 实例 | `const { proxy } = getCurrentInstance() as ComponentInternalInstance` |
| 工具访问 | `proxy?.$modal`（弹窗）、`proxy?.parseTime()`（时间格式化）、`proxy?.animate`（动画） |
| 分页 | 使用 RuoYi 封装的 `<pagination>` 组件 |
| 工具栏 | 使用 `<right-toolbar>` 组件 |
| 搜索折叠 | 搜索区域用 `<transition>` 包裹，`v-show="showSearch"` |
| 权限 | 使用 `v-hasPermi` / `v-hasRole` 指令 |
| 确认弹窗 | `proxy?.$modal.confirm('确认信息')` |
| 消息提示 | `proxy?.$modal.msgSuccess()` / `msgError()` |
| 日期范围 | `proxy?.addDateRange(queryParams, dateRange)` |

### 4. Element Plus 规范

| 规则 | 说明 |
|------|------|
| 引入 | 框架已全局注册，无需手动按需引入 Element Plus 组件 |
| 表格 | `<el-table>` 必须加 `border` 属性 |
| 表单校验 | `<el-form>` 加 `:rules="rules"`，`<el-form-item>` 加 `prop` |
| 弹窗 | `<el-dialog>` 加 `append-to-body`，宽度统一用 `px` 值 |
| 抽屉 | 详情展示优先使用 `<el-drawer>` 而非弹窗 |
| 图标 | 使用 `icon="Search"` 属性或 `<i-ep-xxx />` 组件 |

### 5. TypeScript 规范

| 规则 | 说明 |
|------|------|
| 类型文件 | 每个 API 模块配一个 `types.ts` |
| 命名 | 接口用 PascalCase（`AlertRecordVo`），查询用 `XxxQuery` |
| ref 类型 | `ref<Type>(initialValue)` 显式声明 |
| 全局类型 | `PageQuery`、`R<T>`、`TableDataInfo<T>`、`DictDataOption` 等在全局已声明 |
| 事件处理 | `async/await` 优先，错误用 `try/catch` |

### 6. Pinia Store 规范

| 规则 | 说明 |
|------|------|
| 定义 | 使用 `defineStore` + Setup Store 语法（函数式） |
| 文件位置 | `store/modules/{module}.ts` |
| 使用 | `const store = useXxxStore()` + `const { state } = storeToRefs(store)` |
| 全局 Store | 不要随意修改已有的全局 Store（user、dict、permission 等） |

### 7. UnoCSS 规范

| 场景 | 类名 | 说明 |
|------|------|------|
| 页面外层 | `class="p-2"` | 内边距 8px |
| 间距 | `class="mb-[10px]"` / `class="mb8"` | 底部间距 |
| 布局 | `flex` / `items-center` / `justify-between` | Flex 布局 |
| 文字 | `text-sm` / `font-600` / `text-gray-500` | 文字样式 |
| 圆角 | `rounded` / `rounded-md` | 圆角 |

### 8. 权限控制规范

| 规则 | 说明 |
|------|------|
| 按钮权限 | `v-hasPermi="['{module}:{feature}:{action}']"` |
| 角色权限 | `v-hasRole="['admin']"` |
| 权限字符串 | `{module}:{feature}:{action}`，action 为 `query`/`add`/`edit`/`remove`/`export` |
| 菜单权限 | 通过后端系统管理配置，前端自动生成路由 |

---

## 新建业务模块页面完整步骤

### 步骤 1：创建 API 类型定义

创建 `src/api/{module}/{feature}/types.ts`：

```typescript
// 参考下方「API types.ts 模板」
```

### 步骤 2：创建 API 函数

创建 `src/api/{module}/{feature}/index.ts`：

```typescript
// 参考下方「API index.ts 模板」
```

### 步骤 3：创建页面组件

创建 `src/views/{module}/{feature}/index.vue`：

```vue
<!-- 参考下方「列表页面模板」 -->
```

### 步骤 4：配置菜单（后端操作）

在后台管理系统 → 系统管理 → 菜单管理中添加菜单：
- **菜单名称**：功能名称
- **路由地址**：`{feature}`（与 views 下的目录名对应）
- **组件路径**：`{module}/{feature}/index`
- **权限标识**：`{module}:{feature}:query`
- 按钮权限：`{module}:{feature}:add`、`{module}:{feature}:edit`、`{module}:{feature}:remove`

### 步骤 5：验证

1. 启动前端 `npm run dev`
2. 登录后台，确认菜单出现
3. 测试 CRUD 功能

---

## 代码模板

### API types.ts 模板

```typescript
export interface {Feature}Query {
  pageNum?: number;
  pageSize?: number;
  name?: string;
  status?: string;
  beginTime?: string;
  endTime?: string;
}

export interface {Feature}Vo {
  id: string;
  name: string;
  status: string;
  createTime: string;
}

export interface {Feature}DetailVo {
  id: string;
  name: string;
  status: string;
  remark: string;
  createTime: string;
  updateTime: string;
}
```

### API index.ts 模板

```typescript
import request from '@/utils/request';
import { AxiosPromise } from 'axios';
import { {Feature}Query, {Feature}Vo, {Feature}DetailVo } from './types';

// 查询列表
export function list{Feature}(query: {Feature}Query): AxiosPromise<TableDataInfo<{Feature}Vo>> {
  return request({
    url: '/{module}/{feature}/list',
    method: 'get',
    params: query
  });
}

// 查询详情
export function get{Feature}(id: string): AxiosPromise<R<{Feature}DetailVo>> {
  return request({
    url: `/{module}/{feature}/${id}`,
    method: 'get'
  });
}

// 新增
export function add{Feature}(data: any): AxiosPromise<R<void>> {
  return request({
    url: '/{module}/{feature}',
    method: 'post',
    data: data
  });
}

// 修改
export function update{Feature}(data: any): AxiosPromise<R<void>> {
  return request({
    url: '/{module}/{feature}',
    method: 'put',
    data: data
  });
}

// 删除
export function del{Feature}(ids: Array<string | number> | string): AxiosPromise<R<void>> {
  return request({
    url: `/{module}/{feature}/${ids}`,
    method: 'delete'
  });
}
```

### 列表页面模板（完整 CRUD）

```vue
<template>
  <div class="p-2">
    <!-- 搜索区域（可折叠） -->
    <transition :enter-active-class="proxy?.animate.searchAnimate.enter" :leave-active-class="proxy?.animate.searchAnimate.leave">
      <div v-show="showSearch" class="mb-[10px]">
        <el-card shadow="hover">
          <el-form ref="queryFormRef" :model="queryParams" :inline="true">
            <el-form-item label="名称" prop="name">
              <el-input v-model="queryParams.name" placeholder="请输入名称" clearable @keyup.enter="handleQuery" />
            </el-form-item>
            <el-form-item label="状态" prop="status">
              <el-select v-model="queryParams.status" placeholder="全部" clearable>
                <el-option label="正常" value="0" />
                <el-option label="停用" value="1" />
              </el-select>
            </el-form-item>
            <el-form-item>
              <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
              <el-button icon="Refresh" @click="resetQuery">重置</el-button>
            </el-form-item>
          </el-form>
        </el-card>
      </div>
    </transition>

    <!-- 表格区域 -->
    <el-card shadow="hover">
      <template #header>
        <el-row :gutter="10" class="mb8">
          <el-col :span="1.5">
            <el-button v-hasPermi="['{module}:{feature}:add']" type="primary" plain icon="Plus" @click="handleAdd">新增</el-button>
          </el-col>
          <el-col :span="1.5">
            <el-button v-hasPermi="['{module}:{feature}:remove']" type="danger" plain icon="Delete" :disabled="multiple" @click="handleDelete()">删除</el-button>
          </el-col>
          <right-toolbar v-model:show-search="showSearch" @query-table="getList"></right-toolbar>
        </el-row>
      </template>

      <el-table v-loading="loading" border :data="list" @selection-change="handleSelectionChange">
        <el-table-column type="selection" width="55" align="center" />
        <el-table-column label="名称" align="left" prop="name" :show-overflow-tooltip="true" min-width="150" />
        <el-table-column label="状态" align="center" prop="status" width="100">
          <template #default="scope">
            <el-tag :type="scope.row.status === '0' ? 'success' : 'info'">{{ scope.row.status === '0' ? '正常' : '停用' }}</el-tag>
          </template>
        </el-table-column>
        <el-table-column label="创建时间" align="center" prop="createTime" width="180">
          <template #default="scope">
            <span>{{ proxy?.parseTime(scope.row.createTime) }}</span>
          </template>
        </el-table-column>
        <el-table-column label="操作" align="center" width="180" class-name="small-padding fixed-width">
          <template #default="scope">
            <el-tooltip content="编辑" placement="top">
              <el-button v-hasPermi="['{module}:{feature}:edit']" link type="primary" icon="Edit" @click="handleUpdate(scope.row)" />
            </el-tooltip>
            <el-tooltip content="删除" placement="top">
              <el-button v-hasPermi="['{module}:{feature}:remove']" link type="primary" icon="Delete" @click="handleDelete(scope.row)" />
            </el-tooltip>
          </template>
        </el-table-column>
      </el-table>
      <pagination v-show="total > 0" v-model:page="queryParams.pageNum" v-model:limit="queryParams.pageSize" :total="total" @pagination="getList" />
    </el-card>

    <!-- 新增/编辑对话框 -->
    <el-dialog :title="dialogTitle" v-model="dialogVisible" width="600px" append-to-body>
      <el-form ref="formRef" :model="form" :rules="rules" label-width="100px">
        <el-form-item label="名称" prop="name">
          <el-input v-model="form.name" placeholder="请输入名称" maxlength="100" />
        </el-form-item>
        <el-form-item label="状态" prop="status">
          <el-radio-group v-model="form.status">
            <el-radio value="0">正常</el-radio>
            <el-radio value="1">停用</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="备注" prop="remark">
          <el-input v-model="form.remark" type="textarea" placeholder="请输入备注" :rows="3" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button type="primary" @click="submitForm">确 定</el-button>
        <el-button @click="cancel">取 消</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup name="{Feature}Page" lang="ts">
import { list{Feature}, get{Feature}, add{Feature}, update{Feature}, del{Feature} } from '@/api/{module}/{feature}';
import type { {Feature}Query, {Feature}Vo } from '@/api/{module}/{feature}/types';

const { proxy } = getCurrentInstance() as ComponentInternalInstance;

const list = ref<{Feature}Vo[]>([]);
const loading = ref(true);
const showSearch = ref(true);
const ids = ref<Array<string | number>>([]);
const multiple = ref(true);
const total = ref(0);
const dialogVisible = ref(false);
const dialogTitle = ref('');

const queryFormRef = ref<ElFormInstance>();
const formRef = ref<ElFormInstance>();

const queryParams = ref<{Feature}Query>({
  pageNum: 1,
  pageSize: 10,
  name: undefined,
  status: undefined
});

const form = ref<any>({});
const rules = ref<ElFormRules>({
  name: [{ required: true, message: '名称不能为空', trigger: 'blur' }]
});

/** 查询列表 */
function getList() {
  loading.value = true;
  list{Feature}(queryParams.value).then((res: any) => {
    list.value = res.rows;
    total.value = res.total;
  }).finally(() => {
    loading.value = false;
  });
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1;
  getList();
}

/** 重置按钮操作 */
function resetQuery() {
  queryFormRef.value?.resetFields();
  handleQuery();
}

/** 多选框选中数据 */
function handleSelectionChange(selection: {Feature}Vo[]) {
  ids.value = selection.map((item) => item.id);
  multiple.value = !selection.length;
}

/** 新增按钮操作 */
function handleAdd() {
  reset();
  dialogVisible.value = true;
  dialogTitle.value = '新增';
}

/** 修改按钮操作 */
function handleUpdate(row: {Feature}Vo) {
  reset();
  const id = row.id;
  get{Feature}(id).then((res: any) => {
    form.value = res.data;
    dialogVisible.value = true;
    dialogTitle.value = '修改';
  });
}

/** 提交按钮 */
function submitForm() {
  formRef.value?.validate((valid: boolean) => {
    if (valid) {
      if (form.value.id) {
        update{Feature}(form.value).then(() => {
          proxy?.$modal.msgSuccess('修改成功');
          dialogVisible.value = false;
          getList();
        });
      } else {
        add{Feature}(form.value).then(() => {
          proxy?.$modal.msgSuccess('新增成功');
          dialogVisible.value = false;
          getList();
        });
      }
    }
  });
}

/** 删除按钮操作 */
function handleDelete(row?: {Feature}Vo) {
  const deleteIds = row?.id ? [row.id] : ids.value;
  proxy?.$modal.confirm('是否确认删除？').then(() => {
    return del{Feature}(deleteIds);
  }).then(() => {
    getList();
    proxy?.$modal.msgSuccess('删除成功');
  }).catch(() => {});
}

/** 表单重置 */
function reset() {
  form.value = {};
  formRef.value?.resetFields();
}

/** 取消按钮 */
function cancel() {
  dialogVisible.value = false;
  reset();
}

onMounted(() => {
  getList();
});
</script>
```

### 详情抽屉模板

```vue
<template>
  <!-- 在列表页 index.vue 中添加详情抽屉 -->
  <el-drawer v-model="detailDrawer" title="详情" size="60%" :destroy-on-close="true">
    <div v-loading="detailLoading">
      <el-descriptions :column="2" border>
        <el-descriptions-item label="名称">{{ detail.name }}</el-descriptions-item>
        <el-descriptions-item label="状态">
          <el-tag :type="detail.status === '0' ? 'success' : 'info'">
            {{ detail.status === '0' ? '正常' : '停用' }}
          </el-tag>
        </el-descriptions-item>
        <el-descriptions-item label="备注" :span="2">{{ detail.remark || '-' }}</el-descriptions-item>
      </el-descriptions>

      <el-descriptions :column="2" border class="mt-4">
        <el-descriptions-item label="创建时间">{{ proxy?.parseTime(detail.createTime) }}</el-descriptions-item>
        <el-descriptions-item label="更新时间">{{ proxy?.parseTime(detail.updateTime) }}</el-descriptions-item>
      </el-descriptions>
    </div>
  </el-drawer>
</template>

<script setup lang="ts">
// 在列表页 script 中添加以下代码

const detailDrawer = ref(false);
const detailLoading = ref(false);
const detail = ref<{Feature}DetailVo>({} as {Feature}DetailVo);

/** 查看详情 */
const handleDetail = async (row: {Feature}Vo) => {
  detailDrawer.value = true;
  detailLoading.value = true;
  try {
    const res = await get{Feature}(row.id);
    detail.value = res.data;
  } finally {
    detailLoading.value = false;
  }
};
</script>
```

### 带日期范围的列表页面模板（搜索增强版）

```vue
<!-- 在搜索区域中添加日期范围选择器 -->
<el-form-item label="创建时间" style="width: 308px">
  <el-date-picker
    v-model="dateRange"
    value-format="YYYY-MM-DD HH:mm:ss"
    type="daterange"
    range-separator="-"
    start-placeholder="开始日期"
    end-placeholder="结束日期"
    :default-time="[new Date(2000, 1, 1, 0, 0, 0), new Date(2000, 1, 1, 23, 59, 59)]"
  />
</el-form-item>
```

```typescript
// script 中添加
const dateRange = ref<[DateModelType, DateModelType]>(['', '']);

// 修改 getList 函数，使用 proxy?.addDateRange
const getList = async () => {
  loading.value = true;
  const res = await list{Feature}(proxy?.addDateRange(queryParams.value, dateRange.value));
  list.value = res.rows;
  total.value = res.total;
  loading.value = false;
};

// 修改 resetQuery，重置日期范围
const resetQuery = () => {
  dateRange.value = ['', ''];
  queryFormRef.value?.resetFields();
  handleQuery();
};
```

### Pinia Store 模板

```typescript
// store/modules/{module}.ts
import { defineStore } from 'pinia';

export const use{Module}Store = defineStore('{module}', () => {
  // 状态
  const currentId = ref<string>('');
  const detailInfo = ref<any>(null);

  // 操作
  function setCurrentId(id: string) {
    currentId.value = id;
  }

  function setDetailInfo(info: any) {
    detailInfo.value = info;
  }

  function reset() {
    currentId.value = '';
    detailInfo.value = null;
  }

  return {
    currentId,
    detailInfo,
    setCurrentId,
    setDetailInfo,
    reset
  };
});
```

**使用方式：**

```typescript
import { use{Module}Store } from '@/store/modules/{module}';
import { storeToRefs } from 'pinia';

const { currentId, detailInfo } = storeToRefs(use{Module}Store());
const {Module}Store = use{Module}Store();

// 修改状态
{Module}Store.setCurrentId('123');
```

---

## 常见模式

### 1. 字典回显

**方式1：useDict（推荐）**

```vue
<script setup lang="ts">
const { sys_normal_disable } = toRefs<any>(proxy?.useDict('sys_normal_disable'));
</script>

<template>
  <!-- 使用 DictTag 组件回显 -->
  <dict-tag :options="sys_normal_disable" :value="row.status" />
</template>
```

**方式2：el-select 下拉（字典选项）**

```vue
<el-select v-model="form.status" placeholder="请选择状态">
  <el-option
    v-for="dict in sys_normal_disable"
    :key="dict.value"
    :label="dict.label"
    :value="dict.value"
  />
</el-select>
```

### 2. 文件上传

**图片上传：**

```vue
<image-upload v-model="form.avatar" :limit="1" />
```

**文件上传：**

```vue
<file-upload v-model="form.fileList" :limit="5" />
```

> 组件路径：`src/components/ImageUpload/`、`src/components/FileUpload/`

### 3. 富文本编辑器

```vue
<editor v-model="form.content" :min-height="300" />
```

> 组件路径：`src/components/Editor/`

### 4. 图表（ECharts）

```vue
<script setup lang="ts">
import * as echarts from 'echarts';

const chartRef = ref<HTMLDivElement>();
let chartInstance: echarts.ECharts | null = null;

function initChart() {
  if (chartRef.value) {
    chartInstance = echarts.init(chartRef.value);
    chartInstance.setOption({
      // ECharts 配置
    });
  }
}

onMounted(() => {
  initChart();
});

onBeforeUnmount(() => {
  chartInstance?.dispose();
});
</script>

<template>
  <div ref="chartRef" style="width: 100%; height: 400px" />
</template>
```

### 5. 权限按钮控制

```vue
<!-- 按钮级权限 -->
<el-button v-hasPermi="['system:user:add']">新增用户</el-button>
<el-button v-hasPermi="['system:user:edit']">编辑用户</el-button>
<el-button v-hasPermi="['system:user:remove']">删除用户</el-button>
<el-button v-hasPermi="['system:user:export']">导出用户</el-button>

<!-- 角色级权限 -->
<el-button v-hasRole="['admin']">管理员操作</el-button>

<!-- 操作列中 -->
<el-tooltip content="编辑" placement="top">
  <el-button v-hasPermi="['{module}:{feature}:edit']" link type="primary" icon="Edit" @click="handleUpdate(scope.row)" />
</el-tooltip>
```

### 6. 表格列自定义渲染

```vue
<!-- 状态标签 -->
<el-table-column label="状态" align="center" prop="status" width="100">
  <template #default="scope">
    <el-tag :type="scope.row.status === '0' ? 'success' : 'info'">
      {{ scope.row.status === '0' ? '正常' : '停用' }}
    </el-tag>
  </template>
</el-table-column>

<!-- 时间格式化 -->
<el-table-column label="创建时间" align="center" prop="createTime" width="180">
  <template #default="scope">
    <span>{{ proxy?.parseTime(scope.row.createTime) }}</span>
  </template>
</el-table-column>

<!-- 长文本省略 -->
<el-table-column label="描述" align="left" prop="description" :show-overflow-tooltip="true" min-width="200" />
```

### 7. 导入导出

```vue
<script setup lang="ts">
// 导出
function handleExport() {
  proxy?.$modal.confirm('是否确认导出数据？').then(() => {
    return export{Feature}(queryParams.value);
  }).then((response: any) => {
    download.excel(response, '{功能名称}.xlsx');
  }).catch(() => {});
}

// 导入
const upload = reactive<ImportOption>({
  open: false,
  title: '{功能名称}导入',
  isUploading: false,
  updateSupport: 0,
  headers: globalHeaders(),
  url: import.meta.env.VITE_APP_BASE_API + '/{module}/{feature}/importData'
});

function handleImport() {
  upload.title = '{功能名称}导入';
  upload.open = true;
}

function submitFileForm() {
  uploadRef.value?.submit();
}

function handleFileUploadProgress() {
  upload.isUploading = true;
}

function handleImportSuccess(response: any) {
  upload.open = false;
  upload.isUploading = false;
  uploadRef.value?.clearFiles();
  proxy?.$modal.msgSuccess(response.msg);
  getList();
}
</script>
```

### 8. 国际化（i18n）

RuoYi-Vue-Plus 已集成 i18n，页面中使用：

```vue
<template>
  <!-- 模板中 -->
  <span>{{ $t('common.save') }}</span>
</template>

<script setup lang="ts">
// 脚本中
import { useI18n } from 'vue-i18n';
const { t } = useI18n();
console.log(t('common.save'));
</script>
```

### 9. 页面间参数传递

```typescript
import { useRouter, useRoute } from 'vue-router';

const router = useRouter();
const route = useRoute();

// 跳转带参数
router.push({ path: '/module/feature/detail', query: { id: row.id } });

// 获取参数
const id = route.query.id;
```

### 10. useDialog Hook

RuoYi 内置 `useDialog` Hook：

```typescript
import useDialog from '@/hooks/useDialog';

const { visible, openDialog, closeDialog } = useDialog({ title: '标题' });
```

---

## 自验清单

代码完成后，**必须逐项检查**以下内容：

### TypeScript 级检查

- [ ] **类型定义完整**：所有 API 响应有对应的 TypeScript 接口
- [ ] **ref 显式类型**：`ref<Type>()` 声明时指定类型
- [ ] **事件参数类型**：`handleSelectionChange(selection: XxxVo[])` 有类型
- [ ] **无 any 滥用**：`form` 用 `ref<any>({})` 是可以的（表单动态字段），但 API 参数应有类型

### API 级检查

- [ ] **URL 一致**：前端 API URL 与后端 `@RequestMapping` 完全一致
- [ ] **HTTP 方法正确**：GET 查询、POST 新增、PUT 修改、DELETE 删除
- [ ] **返回类型正确**：列表 `TableDataInfo<Vo>`，操作 `R<T>`
- [ ] **导入路径正确**：`@/api/{module}/{feature}` 路径无拼写错误

### Element Plus 级检查

- [ ] **表格**：`<el-table>` 有 `border` 属性
- [ ] **表单校验**：`<el-form>` 有 `:rules`，`<el-form-item>` 有 `prop`
- [ ] **弹窗**：`<el-dialog>` 有 `append-to-body`
- [ ] **分页**：使用 `<pagination>` 组件，绑定 `v-model:page` 和 `v-model:limit`
- [ ] **工具栏**：使用 `<right-toolbar>` 组件

### 权限级检查

- [ ] **所有操作按钮**都有 `v-hasPermi` 指令
- [ ] **权限字符串**与后端 `sys_menu` 中的 `perms` 一致
- [ ] **批量操作按钮**有 `:disabled="multiple"` 防止空选择

### 功能级检查

- [ ] **搜索重置**：重置按钮调用 `resetFields()` 并重新查询
- [ ] **分页重置**：搜索后 `pageNum` 重置为 1
- [ ] **新增/修改区分**：`form.value.id` 存在走修改，否则走新增
- [ ] **删除确认**：使用 `proxy?.$modal.confirm()` 确认
- [ ] **操作反馈**：成功/失败有 `msgSuccess`/`msgError` 提示
- [ ] **列表刷新**：写操作成功后调用 `getList()` 刷新列表
- [ ] **加载状态**：列表查询有 `v-loading="loading"` 控制
- [ ] **时间格式化**：日期字段使用 `proxy?.parseTime()` 格式化
- [ ] **长文本省略**：表格列使用 `:show-overflow-tooltip="true"`

### 目录级检查

- [ ] **API 文件位置**：`src/api/{module}/{feature}/index.ts` + `types.ts`
- [ ] **页面文件位置**：`src/views/{module}/{feature}/index.vue`
- [ ] **文件命名**：Vue 文件 `kebab-case`（`user-profile.vue`），TypeScript `PascalCase` 接口

---

## 经验库写入规范

经验库路径：`doc/lessons-learned.md`

### 写入原则

> **原则性 > 数值性，模式级 > 页面级，可迁移 > 可复制**

### 写入格式

```markdown
### [简短标题]

- **场景**：什么情况下遇到的问题
- **根因**：为什么会出问题
- **方案**：正确的做法是什么
- **判断标准**：去掉具体数值和页面名，这句话还能指导决策吗？（✅ 能 / ❌ 不能）
```

### 正确示例

```markdown
### 表格列表写操作后必须刷新列表

- **场景**：新增/修改/删除操作成功后，列表数据未更新
- **根因**：缺少 `getList()` 调用
- **方案**：所有写操作的 `.then()` 回调中必须调用 `getList()` 刷新列表
- **判断标准**：✅ 能，适用于所有 CRUD 列表页
```

### 错误示例

```markdown
### 告警列表删除后要刷新

- **场景**：告警删除后列表没更新
- **根因**：没刷新
- **方案**：加上 getList()
- **判断标准**：❌ 不能，只适用于告警列表
```

---

## 参考代码

项目中的真实代码是最好的参考：

| 参考模块 | 路径 | 说明 |
|---------|------|------|
| 巡检-告警 | `src/views/inspection/alarm/` + `src/api/inspection/alarm/` | 完整列表+详情页面 |
| 巡检-报告 | `src/views/inspection/report/` + `src/api/inspection/report/` | 带子组件的页面 |
| 系统管理 | `src/views/system/` | 标准 CRUD 页面集合 |
| 字典 Store | `src/store/modules/dict.ts` | Pinia Store 示例 |
| useDialog | `src/hooks/useDialog.ts` | 组合式函数示例 |
| 全局组件 | `src/components/` | 所有公共组件源码 |

**开发时优先参考项目已有代码的模式和风格，保持一致性。**
