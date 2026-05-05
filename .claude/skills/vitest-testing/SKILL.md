---
name: vitest-testing
description: |
  Vitest + Vue Test Utils 前端单元/集成测试。给测试工程师Agent使用，当需要编写前端API层、Store层、组件的单元测试或集成测试时触发。覆盖RuoYi-Vue-Plus前端项目的特有模式（request.ts、useDict、v-hasPermi、动态路由等）。
---

# Vitest + Vue Test Utils 前端单元/集成测试

> 适配项目：RuoYi-Vue-Plus 5.6.0 前端
> 技术栈：Vue 3.5 + Element Plus 2.13.5 + Pinia 3.0.4 + UnoCSS + TypeScript + Vite 6.x + Vitest 4.x

**定位：** 本Skill专注前端测试的**具体写法**（怎么写代码），与 `test-automation` Skill 的流水线编排互补。

---

## 一、测试框架配置

### 1.1 安装测试依赖

```bash
# 在 plus-ui 目录下安装
cd {FRONTEND_DIR}

# 核心依赖
pnpm add -D vitest @vue/test-utils happy-dom @pinia/testing

# Vitest 相关工具（覆盖率、快照、UI）
pnpm add -D @vitest/coverage-v8 @vitest/snapshot
```

在 `package.json` 中添加测试脚本：

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui",
    "test:watch": "vitest watch"
  }
}
```

### 1.2 vitest.config.ts

在项目根目录创建 `vitest.config.ts`：

```typescript
import { defineConfig } from 'vitest/config';
import vue from '@vitejs/plugin-vue';
import { resolve } from 'path';

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
  test: {
    // 使用 happy-dom 模拟浏览器环境
    environment: 'happy-dom',
    // 全局 API（describe、it、expect 等），无需手动 import
    globals: true,
    // 包含的测试文件
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
    // 排除的目录
    exclude: ['node_modules', 'dist', 'src/**/*.d.ts'],
    // 覆盖率配置
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html', 'lcov'],
      include: ['src/**/*.{ts,vue}'],
      exclude: [
        'src/main.ts',
        'src/App.vue',
        'src/router/**',
        'src/**/*.d.ts',
        'src/**/*.test.ts',
        'src/**/*.spec.ts',
        'src/utils/auth.ts',
      ],
      thresholds: {
        branches: 60,
        functions: 60,
        lines: 60,
        statements: 60,
      },
    },
    // Setup 文件
    setupFiles: ['./src/test/setup.ts'],
  },
});
```

### 1.3 测试 Setup 文件

创建 `src/test/setup.ts`：

```typescript
// src/test/setup.ts

import { vi, beforeEach, afterEach } from 'vitest';
import { config } from '@vue/test-utils';

// ==================== 全局 Mock ====================

// Mock window.matchMedia
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: vi.fn().mockImplementation((query: string) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(),
    removeListener: vi.fn(),
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
});

// Mock getComputedStyle
Object.defineProperty(window, 'getComputedStyle', {
  value: () => ({
    getPropertyValue: () => '',
  }),
});

// Mock ResizeObserver
global.ResizeObserver = vi.fn().mockImplementation(() => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
}));

// Mock IntersectionObserver
global.IntersectionObserver = vi.fn().mockImplementation(() => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
}));

// Mock Element.prototype.closest
Element.prototype.closest = vi.fn(function (selector: string) {
  return this;
}) as any;

// Mock classList
Element.prototype.classList = {
  add: vi.fn(),
  remove: vi.fn(),
  contains: vi.fn(() => false),
  toggle: vi.fn(),
} as any;

// Mock scrollTo
window.scrollTo = vi.fn();

// Mock localStorage
const localStorageMock = (() => {
  let store: Record<string, string> = {};
  return {
    getItem: vi.fn((key: string) => store[key] ?? null),
    setItem: vi.fn((key: string, value: string) => {
      store[key] = value;
    }),
    removeItem: vi.fn((key: string) => {
      delete store[key];
    }),
    clear: vi.fn(() => {
      store = {};
    }),
  };
})();

Object.defineProperty(window, 'localStorage', { value: localStorageMock });

// Mock sessionStorage
Object.defineProperty(window, 'sessionStorage', { value: localStorageMock });

// ==================== 全局清理 ====================

afterEach(() => {
  vi.restoreAllMocks();
  vi.clearAllTimers();
});

// ==================== Vue Test Utils 全局配置 ====================

// 全局 stub（减少不必要的组件渲染）
config.global.stubs = {
  // 按需 stub 复杂组件，不要过度 stub
  // 'RouterLink': true,
  // 'RouterView': true,
};

// Suppress Vue 3 transition warnings
config.global.config.warnHandler = () => null;
```

### 1.4 测试目录结构

```
plus-ui/src/
├── test/
│   ├── setup.ts                           # 全局 Setup
│   ├── utils/
│   │   ├── render.ts                      # 自定义 render 辅助函数
│   │   ├── element-helper.ts              # Element Plus 组件辅助函数
│   │   ├── form-helper.ts                 # 表单填充辅助函数
│   │   └── api-mock.ts                    # API Mock 辅助函数
│   └── fixtures/
│       └── notice.ts                      # 测试数据 Fixtures
├── api/system/notice/
│   └── index.test.ts                      # API 层测试
├── store/modules/
│   └── user.test.ts                       # Store 层测试
└── views/system/notice/
    └── components/
        └── NoticeDialog.test.ts           # 组件测试
```

---

## 二、API 层测试

### 2.1 完整模板（基于 Notice API）

```typescript
// src/api/system/notice/index.test.ts

import { describe, it, expect, vi, beforeEach } from 'vitest';
import axios from 'axios';
import { listNotice, getNotice, addNotice, updateNotice, delNotice } from '@/api/system/notice';
import { NoticeForm, NoticeQuery, NoticeVO } from '@/api/system/notice/types';

// Mock axios
vi.mock('axios', () => {
  const mockAxios = {
    create: vi.fn(() => mockAxios),
    get: vi.fn(),
    post: vi.fn(),
    put: vi.fn(),
    delete: vi.fn(),
    defaults: {
      headers: { 'Content-Type': 'application/json;charset=utf-8' },
    },
    interceptors: {
      request: { use: vi.fn() },
      response: { use: vi.fn() },
    },
  };
  return { default: mockAxios };
});

// Mock request.ts 模块（避免真实网络请求和拦截器副作用）
vi.mock('@/utils/request', () => ({
  default: {
    get: vi.fn(),
    post: vi.fn(),
    put: vi.fn(),
    delete: vi.fn(),
  },
}));

import request from '@/utils/request';

const mockRequest = request as unknown as {
  get: ReturnType<typeof vi.fn>;
  post: ReturnType<typeof vi.fn>;
  put: ReturnType<typeof vi.fn>;
  delete: ReturnType<typeof vi.fn>;
};

// ==================== 测试数据 ====================

const mockNotice: NoticeVO = {
  noticeId: 1,
  noticeTitle: '测试公告',
  noticeType: '1',
  noticeContent: '测试内容',
  status: '0',
  remark: '',
  createByName: 'admin',
};

const mockNoticeList: NoticeVO[] = [
  mockNotice,
  { ...mockNotice, noticeId: 2, noticeTitle: '第二个公告' },
];

// ==================== 测试用例 ====================

describe('Notice API', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  describe('listNotice - 查询公告列表', () => {
    it('应该发送 GET 请求并返回公告列表', async () => {
      mockRequest.get.mockResolvedValueOnce({
        data: { rows: mockNoticeList, total: 2 },
      });

      const result = await listNotice({ pageNum: 1, pageSize: 10 });

      expect(mockRequest.get).toHaveBeenCalledWith('/system/notice/list', {
        params: { pageNum: 1, pageSize: 10 },
      });
      expect(result).toBeDefined();
    });

    it('应该传递查询参数', async () => {
      const query: NoticeQuery = {
        pageNum: 1,
        pageSize: 10,
        noticeTitle: '测试',
        noticeType: '1',
        status: '0',
      };

      mockRequest.get.mockResolvedValueOnce({
        data: { rows: [], total: 0 },
      });

      await listNotice(query);

      expect(mockRequest.get).toHaveBeenCalledWith('/system/notice/list', {
        params: query,
      });
    });

    it('请求失败时应该抛出错误', async () => {
      mockRequest.get.mockRejectedValueOnce(new Error('网络错误'));

      await expect(listNotice({ pageNum: 1, pageSize: 10 })).rejects.toThrow('网络错误');
    });
  });

  describe('getNotice - 查询公告详情', () => {
    it('应该根据 ID 查询公告', async () => {
      mockRequest.get.mockResolvedValueOnce({
        data: { data: mockNotice },
      });

      const result = await getNotice(1);

      expect(mockRequest.get).toHaveBeenCalledWith('/system/notice/1');
    });

    it('应该支持字符串类型 ID', async () => {
      mockRequest.get.mockResolvedValueOnce({
        data: { data: mockNotice },
      });

      await getNotice('1');

      expect(mockRequest.get).toHaveBeenCalledWith('/system/notice/1');
    });
  });

  describe('addNotice - 新增公告', () => {
    it('应该发送 POST 请求', async () => {
      const formData: NoticeForm = {
        noticeTitle: '新增公告',
        noticeType: '1',
        noticeContent: '新增内容',
        status: '0',
      };

      mockRequest.post.mockResolvedValueOnce({
        data: { code: 200 },
      });

      await addNotice(formData);

      expect(mockRequest.post).toHaveBeenCalledWith('/system/notice', formData);
    });
  });

  describe('updateNotice - 修改公告', () => {
    it('应该发送 PUT 请求', async () => {
      const formData: NoticeForm = {
        noticeId: 1,
        noticeTitle: '更新公告',
        noticeType: '1',
        noticeContent: '更新内容',
        status: '0',
      };

      mockRequest.put.mockResolvedValueOnce({
        data: { code: 200 },
      });

      await updateNotice(formData);

      expect(mockRequest.put).toHaveBeenCalledWith('/system/notice', formData);
    });
  });

  describe('delNotice - 删除公告', () => {
    it('应该发送 DELETE 请求（单个 ID）', async () => {
      mockRequest.delete.mockResolvedValueOnce({
        data: { code: 200 },
      });

      await delNotice(1);

      expect(mockRequest.delete).toHaveBeenCalledWith('/system/notice/1');
    });

    it('应该发送 DELETE 请求（数组 ID）', async () => {
      mockRequest.delete.mockResolvedValueOnce({
        data: { code: 200 },
      });

      await delNotice([1, 2, 3]);

      expect(mockRequest.delete).toHaveBeenCalledWith('/system/notice/1,2,3');
    });

    it('应该发送 DELETE 请求（字符串 ID）', async () => {
      mockRequest.delete.mockResolvedValueOnce({
        data: { code: 200 },
      });

      await delNotice('1');

      expect(mockRequest.delete).toHaveBeenCalledWith('/system/notice/1');
    });
  });
});
```

### 2.2 通用 API 测试模板

将 `{feature}` / `{Feature}` 替换为实际模块名即可：

```typescript
// src/api/system/{feature}/index.test.ts

import { describe, it, expect, vi, beforeEach } from 'vitest';
import {
  list{Feature},
  get{Feature},
  add{Feature},
  update{Feature},
  del{Feature},
} from '@/api/system/{feature}';
import { {Feature}Form, {Feature}Query, {Feature}VO } from '@/api/system/{feature}/types';

vi.mock('@/utils/request', () => ({
  default: {
    get: vi.fn(),
    post: vi.fn(),
    put: vi.fn(),
    delete: vi.fn(),
  },
}));

import request from '@/utils/request';
const mockRequest = request as any;

const mockData: {Feature}VO = { /* 构造测试数据 */ };

describe('{Feature} API', () => {
  beforeEach(() => vi.clearAllMocks());

  it('list{Feature} - 分页查询', async () => {
    mockRequest.get.mockResolvedValue({ data: { rows: [mockData], total: 1 } });
    await list{Feature}({ pageNum: 1, pageSize: 10 });
    expect(mockRequest.get).toHaveBeenCalledWith('/system/{feature}/list', expect.objectContaining({
      params: expect.any(Object),
    }));
  });

  it('get{Feature} - 详情查询', async () => {
    mockRequest.get.mockResolvedValue({ data: { data: mockData } });
    await get{Feature}(1);
    expect(mockRequest.get).toHaveBeenCalledWith('/system/{feature}/1');
  });

  it('add{Feature} - 新增', async () => {
    mockRequest.post.mockResolvedValue({ data: { code: 200 } });
    await add{Feature}({} as {Feature}Form);
    expect(mockRequest.post).toHaveBeenCalledWith('/system/{feature}', expect.any(Object));
  });

  it('update{Feature} - 更新', async () => {
    mockRequest.put.mockResolvedValue({ data: { code: 200 } });
    await update{Feature}({} as {Feature}Form);
    expect(mockRequest.put).toHaveBeenCalledWith('/system/{feature}', expect.any(Object));
  });

  it('del{Feature} - 删除', async () => {
    mockRequest.delete.mockResolvedValue({ data: { code: 200 } });
    await del{Feature}(1);
    expect(mockRequest.delete).toHaveBeenCalledWith('/system/{feature}/1');
  });
});
```

---

## 三、Store 层测试

### 3.1 完整模板（基于 User Store）

```typescript
// src/store/modules/user.test.ts

import { describe, it, expect, vi, beforeEach } from 'vitest';
import { setActivePinia, createPinia } from 'pinia';
import { useUserStore } from '@/store/modules/user';

// Mock API
vi.mock('@/api/login', () => ({
  login: vi.fn(),
  logout: vi.fn(),
  getInfo: vi.fn(),
}));

vi.mock('@/utils/auth', () => ({
  getToken: vi.fn(() => 'mock-token'),
  setToken: vi.fn(),
  removeToken: vi.fn(),
}));

import { login as loginApi, logout as logoutApi, getInfo as getUserInfo } from '@/api/login';
import { getToken, setToken, removeToken } from '@/utils/auth';

const mockLoginApi = vi.mocked(loginApi);
const mockLogoutApi = vi.mocked(logoutApi);
const mockGetUserInfo = vi.mocked(getUserInfo);
const mockGetToken = vi.mocked(getToken);
const mockSetToken = vi.mocked(setToken);
const mockRemoveToken = vi.mocked(removeToken);

describe('UserStore', () => {
  beforeEach(() => {
    // 每个 test 创建新的 Pinia 实例，避免状态污染
    setActivePinia(createPinia());
    vi.clearAllMocks();
    mockGetToken.mockReturnValue('mock-token');
  });

  // ==================== 初始状态 ====================

  describe('初始状态', () => {
    it('应该有正确的默认值', () => {
      const store = useUserStore();

      expect(store.token).toBe('mock-token');
      expect(store.name).toBe('');
      expect(store.nickname).toBe('');
      expect(store.userId).toBe('');
      expect(store.tenantId).toBe('');
      expect(store.avatar).toBe('');
      expect(store.roles).toEqual([]);
      expect(store.permissions).toEqual([]);
    });
  });

  // ==================== 登录 ====================

  describe('login - 登录', () => {
    it('登录成功后应该更新 token', async () => {
      mockLoginApi.mockResolvedValueOnce({
        data: { access_token: 'new-token-123' },
      });

      const store = useUserStore();
      await store.login({
        username: 'admin',
        password: 'admin123',
        tenantId: '000000',
      });

      expect(mockSetToken).toHaveBeenCalledWith('new-token-123');
      expect(store.token).toBe('new-token-123');
    });

    it('登录失败后不应该更新 token', async () => {
      mockLoginApi.mockRejectedValueOnce(new Error('密码错误'));

      const store = useUserStore();
      await expect(
        store.login({ username: 'admin', password: 'wrong', tenantId: '000000' })
      ).rejects.toThrow('密码错误');

      expect(store.token).toBe('mock-token');
    });
  });

  // ==================== 获取用户信息 ====================

  describe('getInfo - 获取用户信息', () => {
    it('应该正确设置用户信息', async () => {
      mockGetUserInfo.mockResolvedValueOnce({
        data: {
          user: {
            userName: 'admin',
            nickName: '管理员',
            userId: 1,
            avatar: '',
            tenantId: '000000',
          },
          roles: ['admin'],
          permissions: ['system:user:list', 'system:user:add'],
        },
      });

      const store = useUserStore();
      await store.getInfo();

      expect(store.name).toBe('admin');
      expect(store.nickname).toBe('管理员');
      expect(store.userId).toBe(1);
      expect(store.tenantId).toBe('000000');
      expect(store.roles).toEqual(['admin']);
      expect(store.permissions).toEqual(['system:user:list', 'system:user:add']);
    });

    it('无角色时应该设置默认角色 ROLE_DEFAULT', async () => {
      mockGetUserInfo.mockResolvedValueOnce({
        data: {
          user: {
            userName: 'test',
            nickName: '测试用户',
            userId: 2,
            avatar: '',
            tenantId: '000000',
          },
          roles: [],
          permissions: [],
        },
      });

      const store = useUserStore();
      await store.getInfo();

      expect(store.roles).toEqual(['ROLE_DEFAULT']);
    });

    it('获取失败应该抛出异常', async () => {
      mockGetUserInfo.mockRejectedValueOnce(new Error('token 过期'));

      const store = useUserStore();
      await expect(store.getInfo()).rejects.toThrow('token 过期');
    });
  });

  // ==================== 注销 ====================

  describe('logout - 注销', () => {
    it('注销后应该清空所有状态', async () => {
      mockLogoutApi.mockResolvedValueOnce({});

      const store = useUserStore();
      // 先模拟已登录状态
      store.token = 'some-token';
      store.roles = ['admin'];
      store.permissions = ['*:*:*'];
      store.name = 'admin';

      await store.logout();

      expect(store.token).toBe('');
      expect(store.roles).toEqual([]);
      expect(store.permissions).toEqual([]);
      expect(store.name).toBe('');
      expect(mockRemoveToken).toHaveBeenCalled();
    });
  });

  // ==================== setAvatar ====================

  describe('setAvatar - 设置头像', () => {
    it('应该更新头像地址', () => {
      const store = useUserStore();
      store.setAvatar('https://example.com/avatar.jpg');
      expect(store.avatar).toBe('https://example.com/avatar.jpg');
    });
  });
});
```

### 3.2 通用 Store 测试模板

```typescript
// src/store/modules/{feature}.test.ts

import { describe, it, expect, vi, beforeEach } from 'vitest';
import { setActivePinia, createPinia } from 'pinia';
import { use{Feature}Store } from '@/store/modules/{feature}';

// Mock 依赖的 API
vi.mock('@/api/{feature}', () => ({
  list{Feature}: vi.fn(),
  get{Feature}: vi.fn(),
}));

describe('{Feature}Store', () => {
  beforeEach(() => {
    setActivePinia(createPinia());
    vi.clearAllMocks();
  });

  describe('state', () => {
    it('应该有正确的初始值', () => {
      const store = use{Feature}Store();
      // 断言初始状态
      expect(store.list).toEqual([]);
    });
  });

  describe('actions', () => {
    it('获取列表数据', async () => {
      const mockData = [{ id: 1, name: 'test' }];
      // 设置 mock 返回值...

      const store = use{Feature}Store();
      // await store.getList();

      expect(store.list).toEqual(mockData);
    });
  });
});
```

---

## 四、组件测试

### 4.1 测试工具函数

#### 4.1.1 自定义 render 辅助函数

```typescript
// src/test/utils/render.ts

import { mount, VueWrapper } from '@vue/test-utils';
import { Component, VNode } from 'vue';
import { createPinia, setActivePinia } from 'pinia';
import ElementPlus from 'element-plus';
import 'element-plus/dist/index.css';

interface RenderOptions {
  props?: Record<string, any>;
  slots?: Record<string, VNode | string>;
  global?: Record<string, any>;
  plugins?: any[];
}

/**
 * 自定义 render 函数，预配置 Pinia + Element Plus
 */
export function render(component: Component, options: RenderOptions = {}) {
  const pinia = createPinia();
  setActivePinia(pinia);

  return mount(component, {
    props: options.props,
    slots: options.slots,
    global: {
      plugins: [pinia, ElementPlus, ...(options.plugins || [])],
      stubs: {
        RouterLink: true,
        RouterView: true,
        ...(options.global?.stubs || {}),
      },
      mocks: {
        $t: (key: string) => key,
        $route: { query: {}, params: {} },
        $router: { push: vi.fn(), replace: vi.fn(), back: vi.fn() },
        ...(options.global?.mocks || {}),
      },
      provide: {
        ...(options.global?.provide || {}),
      },
    },
  });
}

/**
 * 等待下一个 tick
 */
export async function nextTick() {
  await new Promise((resolve) => setTimeout(resolve, 0));
}

/**
 * 等待指定时间
 */
export async function waitFor(ms: number) {
  await new Promise((resolve) => setTimeout(resolve, ms));
}

/**
 * 触发 Element Plus 组件事件
 */
export async function emitEvent(wrapper: VueWrapper, eventName: string, payload?: any) {
  await wrapper.vm.$emit(eventName, payload);
  await wrapper.vm.$nextTick();
}
```

#### 4.1.2 Element Plus 辅助函数

```typescript
// src/test/utils/element-helper.ts

import { VueWrapper, DOMWrapper } from '@vue/test-utils';

/**
 * 查找 Element Plus Dialog 并等待打开
 */
export async function findDialog(wrapper: VueWrapper, title?: string) {
  // Element Plus Dialog 渲染在 body 下的 Teleport 中
  const dialog = document.querySelector('.el-dialog') as HTMLElement;
  expect(dialog).toBeTruthy();
  if (title) {
    expect(dialog.querySelector('.el-dialog__title')?.textContent).toContain(title);
  }
  return dialog;
}

/**
 * 关闭 Element Plus Dialog
 */
export async function closeDialog() {
  const closeBtn = document.querySelector('.el-dialog__closebtn') as HTMLElement;
  if (closeBtn) {
    closeBtn.click();
    await new Promise((resolve) => setTimeout(resolve, 100));
  }
}

/**
 * 获取 ElMessageBox 确认弹窗
 */
export function getMessageBox() {
  return document.querySelector('.el-message-box') as HTMLElement;
}

/**
 * 确认 MessageBox
 */
export async function confirmMessageBox() {
  const confirmBtn = document.querySelector('.el-message-box__btns .el-button--primary') as HTMLElement;
  if (confirmBtn) {
    confirmBtn.click();
    await new Promise((resolve) => setTimeout(resolve, 100));
  }
}

/**
 * 取消 MessageBox
 */
export async function cancelMessageBox() {
  const cancelBtn = document.querySelector('.el-message-box__btns .el-button:not(.el-button--primary)') as HTMLElement;
  if (cancelBtn) {
    cancelBtn.click();
    await new Promise((resolve) => setTimeout(resolve, 100));
  }
}

/**
 * 查找 El-Table 中的指定行
 */
export function findTableRow(tableWrapper: VueWrapper, rowIndex: number) {
  return tableWrapper.findAll('.el-table__row')[rowIndex];
}

/**
 * 获取 El-Table 的所有行
 */
export function getTableRows(tableWrapper: VueWrapper) {
  return tableWrapper.findAll('.el-table__row');
}

/**
 * 点击 El-Table 行的操作按钮
 */
export async function clickTableRowAction(tableWrapper: VueWrapper, rowIndex: number, buttonText: string) {
  const row = findTableRow(tableWrapper, rowIndex);
  const btn = row.findAll('button').find((btn) => btn.text().includes(buttonText));
  if (btn) {
    await btn.trigger('click');
  }
}

/**
 * 查找 El-Select 并选择指定选项
 */
export async function selectOption(wrapper: VueWrapper | DOMWrapper<Element>, optionText: string) {
  const select = wrapper.find('.el-select');
  await select.trigger('click');
  await new Promise((resolve) => setTimeout(resolve, 100));

  const options = document.querySelectorAll('.el-select-dropdown__item');
  for (const option of options) {
    if (option.textContent?.includes(optionText)) {
      (option as HTMLElement).click();
      await new Promise((resolve) => setTimeout(resolve, 100));
      return;
    }
  }
}

/**
 * 查找 El-Notification
 */
export function getNotification() {
  return document.querySelector('.el-notification') as HTMLElement;
}

/**
 * 获取 El-Message
 */
export function getElMessage() {
  return document.querySelector('.el-message') as HTMLElement;
}
```

#### 4.1.3 表单填充辅助函数

```typescript
// src/test/utils/form-helper.ts

import { VueWrapper, DOMWrapper } from '@vue/test-utils';

/**
 * 填充 El-Input
 */
export async function fillInput(
  wrapper: VueWrapper | DOMWrapper<Element>,
  selector: string,
  value: string
) {
  const input = wrapper.find(selector).find('input');
  await input.setValue(value);
  await input.trigger('input');
  await input.trigger('change');
}

/**
 * 填充 El-Input（通过占位符查找）export async function fillInputByPlaceholder(page: ReturnType<typeof mount>, placeholder: string, value: string) {
  const input = page.find(`input[placeholder="${placeholder}"]`);
  if (input.exists()) {
    await input.setValue(value);
    await input.trigger('input');
  }
}

/**
 * 填充 El-Input（通过 modelValue）
 */
export async function fillInput(wrapper: VueWrapper, value: string) {
  const input = wrapper.find('input');
  await input.setValue(value);
  await input.trigger('input');
}

/**
 * 选择 El-Select 选项
 */
export async function selectOption(wrapper: VueWrapper, value: string | number) {
  const select = wrapper.find('.el-select');
  await select.trigger('click');
  await nextTick();
  const option = document.querySelector(`.el-select-dropdown__item[data-value="${value}"]`);
  if (option) (option as HTMLElement).click();
  await nextTick();
}

/**
 * 切换 El-Switch
 */
export async function toggleSwitch(wrapper: VueWrapper) {
  const switchEl = wrapper.find('.el-switch');
  await switchEl.trigger('click');
  await nextTick();
}

/**
 * 点击 El-DatePicker 并选择日期
 */
export async function selectDate(wrapper: VueWrapper, date: Date) {
  const picker = wrapper.find('.el-date-editor');
  await picker.trigger('click');
  await nextTick();
  // 日期选择器会在body下渲染，需要从document查找
  const cell = document.querySelector('.el-date-table td.available');
  if (cell) (cell as HTMLElement).click();
  await nextTick();
}

/**
 * 填写完整表单（批量）
 * @param wrapper 组件wrapper
 * @param fields 字段配置 [{placeholder/value} | {selector/value}]
 */
export async function fillForm(wrapper: VueWrapper, fields: Array<{ placeholder?: string; selector?: string; value: string }>) {
  for (const field of fields) {
    if (field.placeholder) {
      await fillInputByPlaceholder(wrapper, field.placeholder, field.value);
    } else if (field.selector) {
      const el = wrapper.find(field.selector);
      if (el.exists()) {
        await el.setValue(field.value);
        await el.trigger('input');
      }
    }
  }
  await nextTick();
}

/**
 * API Mock 辅助函数
 */
import { vi } from 'vitest';
import type { Mock } from 'vitest';

/**
 * 创建 API Mock
 * @param moduleName API模块路径
 * @param methodName 方法名
 * @param returnValue 模拟返回值
 */
export function mockApiMethod(moduleName: string, methodName: string, returnValue: any): Mock {
  return vi.fn().mockResolvedValue(returnValue);
}

/**
 * 创建失败的 API Mock
 */
export function mockApiError(moduleName: string, methodName: string, message: string): Mock {
  return vi.fn().mockRejectedValue(new Error(message));
}

---

## 五、常见场景测试

### 5.1 路由守卫测试

```typescript
// tests/router/permission.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { router } from '@/router';
import { useUserStore } from '@/store/modules/user';

// Mock pinia
vi.mock('@/store/modules/user', () => ({
  useUserStore: vi.fn(),
}));

describe('路由权限守卫', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('未登录用户访问需认证页面应跳转登录页', async () => {
    vi.mocked(useUserStore).mockReturnValue({
      token: '',
      roles: [],
      getUserId: () => '',
    } as any);

    await router.push('/system/user');
    expect(router.currentRoute.value.path).toBe('/login');
  });

  it('已登录用户访问登录页应跳转首页', async () => {
    vi.mocked(useUserStore).mockReturnValue({
      token: 'test-token',
      roles: ['admin'],
      getUserId: () => '1',
      getInfo: vi.fn(),
    } as any);

    await router.push('/login');
    expect(router.currentRoute.value.path).toBe('/index');
  });
});
```

### 5.2 字典回显测试

```typescript
// tests/components/dict-tag.test.ts
import { describe, it, expect, vi } from 'vitest';
import { mount } from '@vue/test-utils';
import DictTag from '@/components/DictTag/index.vue';
import { useDict } from '@/hooks/useDict';

// Mock useDict
vi.mock('@/hooks/useDict', () => ({
  useDict: vi.fn(() => ({
    dict: ref([
      { label: '正常', value: '0', elTagType: 'success' },
      { label: '停用', value: '1', elTagType: 'danger' },
    ]),
  })),
}));

import { ref } from 'vue';

describe('DictTag 组件', () => {
  it('应正确渲染字典标签', () => {
    const wrapper = mount(DictTag, {
      props: {
        options: [
          { label: '正常', value: '0', elTagType: 'success' },
          { label: '停用', value: '1', elTagType: 'danger' },
        ],
        value: '0',
      },
    });

    expect(wrapper.text()).toContain('正常');
  });

  it('未知值应显示原始值', () => {
    const wrapper = mount(DictTag, {
      props: {
        options: [
          { label: '正常', value: '0', elTagType: 'success' },
        ],
        value: '2',
      },
    });

    expect(wrapper.text()).toContain('2');
  });
});
```

### 5.3 权限指令测试

```typescript
// tests/directives/hasPermi.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { mount } from '@vue/test-utils';
import { DirectiveBinding } from 'vue';
import { hasPermi } from '@/directive/permission/hasPermi';
import { useUserStore } from '@/store/modules/user';

vi.mock('@/store/modules/user', () => ({
  useUserStore: vi.fn(),
}));

describe('v-hasPermi 指令', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('有权限时应显示元素', () => {
    vi.mocked(useUserStore).mockReturnValue({
      permissions: ['system:user:list'],
    } as any);

    const el = document.createElement('button');
    const binding: DirectiveBinding = {
      instance: null,
      value: 'system:user:list',
      oldValue: undefined,
      modifiers: {},
      arg: undefined,
      dir: hasPermi,
    } as unknown as DirectiveBinding;

    hasPermi.mount!(el, binding);
    expect(el.style.display).not.toBe('none');
  });

  it('无权限时应隐藏元素', () => {
    vi.mocked(useUserStore).mockReturnValue({
      permissions: ['system:role:list'],
    } as any);

    const el = document.createElement('button');
    el.style.display = '';
    const binding: DirectiveBinding = {
      instance: null,
      value: 'system:user:list',
      oldValue: undefined,
      modifiers: {},
      arg: undefined,
      dir: hasPermi,
    } as unknown as DirectiveBinding;

    hasPermi.mount!(el, binding);
    expect(el.style.display).toBe('none');
  });
});
```

### 5.4 文件上传测试

```typescript
// tests/components/upload.test.ts
import { describe, it, expect, vi } from 'vitest';
import { mount } from '@vue/test-utils';
import UploadImg from '@/components/UploadImg/index.vue';

// Mock Element Plus ElMessage
vi.mock('element-plus', () => ({
  ElMessage: { success: vi.fn(), error: vi.fn() },
}));

describe('UploadImg 组件', () => {
  it('应正确渲染上传区域', () => {
    const wrapper = mount(UploadImg, {
      props: {
        modelValue: '',
      },
    });

    expect(wrapper.find('.el-upload').exists()).toBe(true);
  });

  it('有值时应显示预览图', () => {
    const wrapper = mount(UploadImg, {
      props: {
        modelValue: 'https://example.com/image.png',
      },
    });

    expect(wrapper.find('img').exists()).toBe(true);
  });
});
```

---

## 六、RuoYi 前端特有测试要点

### 6.1 request.ts 错误处理测试

```typescript
// tests/utils/request.test.ts
import { describe, it, expect, vi } from 'vitest';
import axios from 'axios';
import service from '@/utils/request';
import { ElMessage, ElMessageBox } from 'element-plus';

vi.mock('axios');
vi.mock('element-plus', () => ({
  ElMessage: { success: vi.fn(), error: vi.fn(), warning: vi.fn() },
  ElMessageBox: { alert: vi.fn() },
}));

describe('request.ts', () => {
  it('401 应跳转登录页', async () => {
    vi.mocked(axios).mockRejectedValueOnce({ response: { status: 401 } });
    await expect(service.get('/test')).rejects.toBeDefined();
  });

  it('500 应显示错误消息', async () => {
    vi.mocked(axios).mockRejectedValueOnce({
      response: { status: 500, data: { msg: '服务器错误' } },
    });
    await expect(service.get('/test')).rejects.toBeDefined();
    expect(ElMessage.error).toHaveBeenCalled();
  });
});
```

### 6.2 useDict Hook 测试

```typescript
// tests/hooks/useDict.test.ts
import { describe, it, expect, vi } from 'vitest';
import { useDict } from '@/hooks/useDict';

// Mock API
vi.mock('@/api/system/dict/data', () => ({
  getDicts: vi.fn().mockResolvedValue([
    { dictLabel: '男', dictValue: '0', dictSort: 1 },
    { dictLabel: '女', dictValue: '1', dictSort: 2 },
  ]),
}));

describe('useDict', () => {
  it('应正确加载字典数据', async () => {
    const { dict } = await useDict('sys_user_sex');
    expect(dict.value).toHaveLength(2);
    expect(dict.value[0].dictLabel).toBe('男');
  });
});
```

### 6.3 动态路由测试

```typescript
// tests/router/dynamic.test.ts
import { describe, it, expect } from 'vitest';

describe('动态路由生成', () => {
  it('应根据后端菜单数据生成路由', () => {
    const menus = [
      {
        name: 'System',
        path: '/system',
        component: 'Layout',
        children: [
          { name: 'User', path: 'user', component: 'system/user/index' },
        ],
      },
    ];

    // 验证路由结构
    expect(menus[0].path).toBe('/system');
    expect(menus[0].children[0].component).toBe('system/user/index');
  });
});
```

---

## 七、自验清单

完成 Vitest 测试后，检查以下项：

- [ ] 测试文件放在 `src/__tests__/` 或 `tests/` 目录下
- [ ] 测试命名清晰（describe描述模块，it描述具体行为）
- [ ] 每个测试独立（不依赖其他测试的执行顺序）
- [ ] Mock 在 beforeEach 中重置（vi.clearAllMocks()）
- [ ] 异步操作使用 await（不遗漏 async/await）
- [ ] 断言覆盖正常路径和异常路径
- [ ] Element Plus 组件交互使用 nextTick 等待渲染
- [ ] 表单校验测试覆盖所有校验规则
- [ ] 权限相关测试覆盖有权限/无权限场景
- [ ] API Mock 覆盖成功和失败场景
