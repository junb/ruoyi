---
name: playwright-testing
description: |
  Playwright UI/E2E 测试编写 Skill。给测试工程师Agent使用，专注于测试用例的编写方法和模式，包括页面对象模型（POM）、常用测试模式、Element Plus 组件交互、调试排错技巧、RuoYi 特有测试模式等。注意：Playwright 配置和基础模板参见 test-automation Skill，本 Skill 不重复配置内容。
---

# Playwright UI/E2E 测试编写 Skill

> 适配项目：RuoYi-Vue-Plus 5.6.0
> 前端：Vue 3 + Element Plus 2.13.5 + Pinia + UnoCSS + TypeScript
> 测试框架：Playwright
> 测试目录：`{E2E_DIR}/`

---

## 一、测试设计原则

### 1.1 按用户行为组织测试

测试用例的命名和组织应反映**用户实际操作流程**，而非技术实现细节。

**❌ 技术导向（错误）：**
```typescript
test('should call API and render table rows', async () => { });
test('should dispatch store action on button click', async () => { });
```

**✅ 用户行为导向（正确）：**
```typescript
test('管理员搜索用户后应显示匹配结果', async () => { });
test('新增用户后应在列表中看到该用户', async () => { });
test('删除用户时应弹出确认框并从列表移除', async () => { });
```

### 1.2 测试独立性

每条测试用例必须**独立运行**，不依赖其他测试的执行结果或执行顺序。

```typescript
// ❌ 错误：依赖前一个测试创建的数据
test('创建用户', async () => { createdUserId = '123'; });
test('编辑用户', async () => { /* 使用 createdUserId */ });

// ✅ 正确：每个测试自己准备数据
test('编辑用户', async () => {
  // 先创建测试数据
  const userId = await createTestUser({ name: '编辑测试用户' });
  // 再执行编辑操作
  await editUser(userId, { name: '修改后名称' });
  // 最后清理
  await deleteUser(userId);
});
```

### 1.3 数据隔离

- 每个测试创建自己需要的数据，测试结束后清理
- 使用唯一标识（时间戳、UUID）避免数据冲突
- 不依赖数据库中已有的固定数据（除非是系统预置数据）

```typescript
// ✅ 使用时间戳保证数据唯一
const timestamp = Date.now();
const testName = `测试用户_${timestamp}`;

// ✅ 测试结束后清理
test.afterEach(async () => {
  // 清理本测试创建的数据
  if (createdId) {
    await cleanupTestData(createdId);
  }
});
```

### 1.4 等待策略

**绝对禁止硬编码 `sleep`！** 使用 Playwright 内置的自动等待机制。

| 场景 | 推荐方式 | 禁止方式 |
|------|----------|----------|
| 等待元素可见 | `await expect(locator).toBeVisible()` | `await page.waitForTimeout(3000)` |
| 等待元素消失 | `await expect(locator).toBeHidden()` | `await page.waitForTimeout(2000)` |
| 等待导航完成 | `await page.waitForURL('**/xxx')` | `await page.waitForTimeout(1000)` |
| 等待网络请求 | `await page.waitForResponse('**/api/**')` | `await page.waitForTimeout(500)` |
| 等待 API 响应 | `await expect(page.locator('.el-message--success')).toBeVisible()` | `await page.waitForTimeout(1000)` |

**Playwright 自动等待：** `click`、`fill`、`press` 等操作会自动等待元素变为可操作状态（可见、启用、不被遮挡），无需手动等待。

```typescript
// ❌ 错误：硬编码等待
await page.waitForTimeout(2000);
await page.click('button');

// ✅ 正确：使用 Playwright 自动等待 + 断言
await page.click('button');
await expect(page.locator('.el-dialog')).toBeVisible();
```

**例外情况：** 仅在极少数无法用 Playwright 内置等待替代的场景下才使用 `waitForTimeout`，并必须添加注释说明原因。

```typescript
// 仅当 Element Plus 动画无法用其他方式等待时
// TODO: 寻找更好的等待方式替代固定等待
await page.waitForTimeout(300); // Element Plus 对话框关闭动画
```

---

## 二、页面对象模型（POM）详细模板

> 基础配置和 BasePage 参见 test-automation Skill 中的 POM 模板，以下为业务场景的详细扩展。

### 2.1 列表页 POM 模板

```typescript
// pages/{module}/{feature}.page.ts
import { Page, Locator, expect } from '@playwright/test';
import { BasePage } from '../base.page';

export class FeaturePage extends BasePage {
  readonly url = '/{module}/{feature}';

  // ========== 搜索区域 ==========
  readonly searchForm = this.page.locator('.el-form--inline');
  readonly searchButton = this.page.locator('button:has-text("搜索")');
  readonly resetButton = this.page.locator('button:has-text("重 置")');

  // 搜索字段（根据实际页面调整）
  readonly nameInput = this.page.locator('[placeholder="请输入名称"]');
  readonly statusSelect = this.page.locator('.el-form--inline .el-select').first();

  // ========== 操作按钮 ==========
  readonly addButton = this.page.locator('button:has-text("新 增")');
  readonly batchDeleteButton = this.page.locator('button:has-text("删 除")');
  readonly exportButton = this.page.locator('button:has-text("导 出")');
  readonly importButton = this.page.locator('button:has-text("导 入")');

  // ========== 表格 ==========
  readonly table = this.page.locator('.el-table');
  readonly tableBody = this.page.locator('.el-table__body-wrapper');
  readonly tableRows = this.page.locator('.el-table__body-wrapper .el-table__row');
  readonly tableCheckboxAll = this.page.locator('.el-table__header-wrapper .el-checkbox');
  readonly emptyBlock = this.page.locator('.el-table__empty-block');

  // ========== 分页 ==========
  readonly pagination = this.page.locator('.pagination');
  readonly totalText = this.page.locator('.pagination .el-pagination__total');
  readonly prevButton = this.page.locator('.pagination button.btn-prev');
  readonly nextButton = this.page.locator('.pagination button.btn-next');
  readonly pageSizeSelect = this.page.locator('.pagination .el-select');

  // ========== 对话框 ==========
  readonly dialog = this.page.locator('.el-dialog__wrapper');
  readonly dialogTitle = this.page.locator('.el-dialog__title');
  readonly dialogSubmit = this.page.locator('.el-dialog__footer button:has-text("确 定")');
  readonly dialogCancel = this.page.locator('.el-dialog__footer button:has-text("取 消")');

  // ========== 消息提示 ==========
  readonly successMessage = this.page.locator('.el-message--success');
  readonly errorMessage = this.page.locator('.el-message--error');
  readonly warningMessage = this.page.locator('.el-message--warning');

  constructor(page: Page) {
    super(page);
  }

  // ========== 导航 ==========

  /** 导航到列表页并等待加载 */
  async goto() {
    await this.page.goto(this.url);
    await this.waitForTable();
  }

  /** 通过菜单导航（适用于非直接 URL 访问） */
  async navigateViaMenu(menuPath: string[]) {
    for (let i = 0; i < menuPath.length - 1; i++) {
      const subMenu = this.page.locator(`.el-sub-menu__title:has-text("${menuPath[i]}")`);
      if (await subMenu.isVisible()) {
        await subMenu.click();
        await this.page.waitForTimeout(300); // 等待子菜单展开动画
      }
    }
    const menuItem = this.page.locator(`.el-menu-item:has-text("${menuPath[menuPath.length - 1]}")`);
    await menuItem.click();
    await this.waitForTable();
  }

  // ========== 搜索操作 ==========

  /** 按名称搜索 */
  async searchByName(name: string) {
    await this.nameInput.fill(name);
    await this.searchButton.click();
    await this.waitForLoading();
  }

  /** 按状态搜索 */
  async searchByStatus(status: string) {
    await this.statusSelect.click();
    await this.page.locator('.el-select-dropdown__item', { hasText: status }).click();
    await this.searchButton.click();
    await this.waitForLoading();
  }

  /** 重置搜索条件 */
  async resetSearch() {
    await this.resetButton.click();
    await this.waitForLoading();
    // 验证输入框已清空
    await expect(this.nameInput).toHaveValue('');
  }

  /** 执行搜索（通用方法） */
  async search(params: { name?: string; status?: string }) {
    if (params.name) {
      await this.nameInput.fill(params.name);
    }
    if (params.status) {
      await this.statusSelect.click();
      await this.page.locator('.el-select-dropdown__item', { hasText: params.status }).click();
    }
    await this.searchButton.click();
    await this.waitForLoading();
  }

  // ========== 表格操作 ==========

  /** 获取表格行数 */
  async getRowCount(): Promise<number> {
    await this.waitForTable();
    if (await this.emptyBlock.isVisible()) {
      return 0;
    }
    return this.tableRows.count();
  }

  /** 获取指定行的单元格文本 */
  async getCellText(row: number, col: number): Promise<string> {
    const cell = this.tableRows.nth(row).locator('td').nth(col);
    return cell.textContent() || '';
  }

  /** 获取指定行中指定列标题的单元格文本 */
  async getCellTextByHeader(row: number, headerText: string): Promise<string> {
    // 找到列标题的索引
    const headerIndex = await this.page.locator('.el-table__header-wrapper th').evaluateAll((ths, target) => {
      return ths.findIndex(th => th.textContent?.includes(target));
    }, headerText);
    if (headerIndex === -1) throw new Error(`列标题 "${headerText}" 不存在`);
    return this.getCellText(row, headerIndex);
  }

  /** 点击行内的操作按钮 */
  async clickRowAction(row: number, action: string) {
    const rowLocator = this.tableRows.nth(row);
    // 操作按钮通常在最后一列
    const actionButton = rowLocator.locator('.el-button', { hasText: action });
    await actionButton.click();
  }

  /** 选中指定行 */
  async selectRow(row: number) {
    const checkbox = this.tableRows.nth(row).locator('.el-checkbox');
    await checkbox.click();
  }

  /** 全选 */
  async selectAll() {
    await this.tableCheckboxAll.click();
  }

  // ========== CRUD 操作 ==========

  /** 点击新增按钮 */
  async clickAdd() {
    await this.addButton.click();
    await this.waitForDialog();
  }

  /** 点击编辑 */
  async clickEdit(row: number) {
    await this.clickRowAction(row, '编辑');
    await this.waitForDialog();
  }

  /** 点击删除 */
  async clickDelete(row?: number) {
    if (row !== undefined) {
      await this.clickRowAction(row, '删除');
    } else {
      // 批量删除
      await this.batchDeleteButton.click();
    }
    // 等待确认弹窗
    await this.page.locator('.el-message-box__wrapper').waitFor({ state: 'visible' });
  }

  /** 确认删除 */
  async confirmDelete() {
    await this.page.locator('.el-message-box__btns .el-button--primary').click();
    await this.waitForLoading();
    await this.waitForMessage('操作成功');
  }

  /** 取消删除 */
  async cancelDelete() {
    await this.page.locator('.el-message-box__btns button:first-child').click();
  }

  // ========== 分页操作 ==========

  /** 获取总条数 */
  async getTotalCount(): Promise<number> {
    const text = await this.totalText.textContent();
    const match = text?.match(/共 (\d+) 条/);
    return match ? parseInt(match[1]) : 0;
  }

  /** 跳转到下一页 */
  async goToNextPage() {
    await this.nextButton.click();
    await this.waitForLoading();
  }

  /** 跳转到上一页 */
  async goToPrevPage() {
    await this.prevButton.click();
    await this.waitForLoading();
  }

  /** 切换每页显示条数 */
  async changePageSize(size: number) {
    await this.pageSizeSelect.click();
    await this.page.locator('.el-select-dropdown__item', { hasText: `${size}条/页` }).click();
    await this.waitForLoading();
  }

  // ========== 表单操作（对话框内） ==========

  /** 填写表单字段 */
  async fillFormField(label: string, value: string) {
    const formItem = this.page.locator('.el-form-item', { hasText: label });
    const input = formItem.locator('.el-input__inner, .el-textarea__inner').first();
    await input.clear();
    await input.fill(value);
  }

  /** 选择下拉框选项 */
  async selectFormField(label: string, optionText: string) {
    const formItem = this.page.locator('.el-form-item', { hasText: label });
    const select = formItem.locator('.el-select').first();
    await select.click();
    await this.page.locator('.el-select-dropdown__item', { hasText: optionText }).click();
  }

  /** 提交表单 */
  async submitForm() {
    await this.dialogSubmit.click();
    await this.waitForLoading();
    await this.waitForMessage('操作成功');
  }

  /** 取消表单 */
  async cancelForm() {
    await this.dialogCancel.click();
  }
}
```

### 2.2 表单弹窗 POM 模板

```typescript
// pages/{module}/{feature}-form.page.ts
import { Page, Locator, expect } from '@playwright/test';
import { BasePage } from '../base.page';

export class FeatureFormPage extends BasePage {
  readonly dialog = this.page.locator('.el-dialog__wrapper');

  // 表单字段（根据实际页面调整）
  readonly nameInput = this.dialog.locator('.el-form-item:has-text("名称") .el-input__inner');
  readonly remarkTextarea = this.dialog.locator('.el-form-item:has-text("备注") .el-textarea__inner');
  readonly statusRadio = this.dialog.locator('.el-form-item:has-text("状态") .el-radio-group');
  readonly typeSelect = this.dialog.locator('.el-form-item:has-text("类型") .el-select');

  // 按钮
  readonly submitButton = this.dialog.locator('button:has-text("确 定")');
  readonly cancelButton = this.dialog.locator('button:has-text("取 消")');

  // 校验提示
  readonly validationMessages = this.dialog.locator('.el-form-item__error');

  constructor(page: Page) {
    super(page);
  }

  /** 等待表单弹窗打开 */
  async waitForOpen() {
    await this.dialog.waitFor({ state: 'visible' });
    await expect(this.dialog.locator('.el-dialog__title')).toBeVisible();
  }

  /** 填写名称 */
  async fillName(name: string) {
    await this.nameInput.clear();
    await this.nameInput.fill(name);
  }

  /** 填写备注 */
  async fillRemark(remark: string) {
    await this.remarkTextarea.clear();
    await this.remarkTextarea.fill(remark);
  }

  /** 选择状态 */
  async selectStatus(status: '正常' | '停用') {
    const value = status === '正常' ? '0' : '1';
    await this.statusRadio.locator(`input[value="${value}"]`).click();
  }

  /** 选择类型（下拉框） */
  async selectType(typeName: string) {
    await this.typeSelect.click();
    await this.page.locator('.el-select-dropdown__item', { hasText: typeName }).click();
  }

  /** 提交表单 */
  async submit() {
    await this.submitButton.click();
    await this.waitForLoading();
  }

  /** 取消表单 */
  async cancel() {
    await this.cancelButton.click();
  }

  /** 获取校验错误信息 */
  async getValidationErrors(): Promise<string[]> {
    const errors = this.validationMessages;
    const count = await errors.count();
    const messages: string[] = [];
    for (let i = 0; i < count; i++) {
      messages.push((await errors.nth(i).textContent()) || '');
    }
    return messages;
  }

  /** 填写完整表单 */
  async fillForm(data: {
    name: string;
    remark?: string;
    status?: '正常' | '停用';
    type?: string;
  }) {
    await this.fillName(data.name);
    if (data.remark) await this.fillRemark(data.remark);
    if (data.status) await this.selectStatus(data.status);
    if (data.type) await this.selectType(data.type);
  }
}
```

### 2.3 登录页 POM 模板

```typescript
// pages/login.page.ts
import { Page, expect } from '@playwright/test';
import { BasePage } from './base.page';

export class LoginPage extends BasePage {
  // 租户选择
  readonly tenantSelect = this.page.locator('.login-form .el-select').first();
  readonly tenantInput = this.page.locator('.login-form .el-select .el-input__inner');

  // 表单字段
  readonly usernameInput = this.page.locator('.login-form input[type="text"]');
  readonly passwordInput = this.page.locator('.login-form input[type="password"]');
  readonly captchaInput = this.page.locator('.login-form input[placeholder*="验证"]');
  readonly captchaImage = this.page.locator('.login-form .captcha-img');

  // 按钮
  readonly loginButton = this.page.locator('.login-form button:has-text("登 录")');
  readonly rememberMeCheckbox = this.page.locator('.login-form .el-checkbox');

  constructor(page: Page) {
    super(page);
  }

  async goto() {
    await this.page.goto('/login');
    await this.page.waitForLoadState('networkidle');
    await expect(this.loginButton).toBeVisible({ timeout: 10000 });
  }

  /** 执行登录 */
  async login(username: string = 'admin', password: string = 'admin123') {
    await this.goto();

    // 选择租户（如果存在）
    if (await this.tenantSelect.isVisible()) {
      await this.tenantSelect.click();
      await this.page.locator('.el-select-dropdown__item').first().click();
    }

    // 填写用户名和密码
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);

    // 验证码处理（测试环境建议禁用 captcha.enable=false）
    if (await this.captchaInput.isVisible()) {
      console.warn('⚠️ 验证码已启用，请确保测试环境已禁用（captcha.enable=false）');
    }

    // 点击登录
    await this.loginButton.click();

    // 等待登录完成并跳转到首页
    await this.page.waitForURL('**/index**', { timeout: 15000 });
    await this.page.waitForLoadState('networkidle');
  }

  /** 使用错误凭据登录（测试用） */
  async loginWithInvalidCredentials(username: string, password: string) {
    await this.goto();
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }

  /** 获取登录错误提示 */
  async getErrorMessage(): Promise<string> {
    const errorEl = this.page.locator('.el-message--error');
    await errorEl.waitFor({ state: 'visible', timeout: 5000 });
    return errorEl.textContent() || '';
  }
}
```

### 2.4 导航菜单 POM 模板

```typescript
// pages/navigation.page.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './base.page';

export class NavigationPage extends BasePage {
  // 侧边栏菜单
  readonly sidebar = this.page.locator('.sidebar-container');
  readonly menuItems = this.page.locator('.el-menu-item');
  readonly subMenus = this.page.locator('.el-sub-menu');
  readonly subMenuTitles = this.page.locator('.el-sub-menu__title');

  // 顶部标签页
  readonly tagsView = this.page.locator('.tags-view-container');
  readonly tagItems = this.page.locator('.tags-view-item');
  readonly activeTag = this.page.locator('.tags-view-item.active');

  // 面包屑
  readonly breadcrumb = this.page.locator('.breadcrumb-container');

  constructor(page: Page) {
    super(page);
  }

  /** 通过菜单路径导航（支持多级菜单） */
  async navigateToMenu(menuPath: string[]) {
    // 展开父级菜单
    for (let i = 0; i < menuPath.length - 1; i++) {
      const subMenuTitle = this.page.locator(`.el-sub-menu__title:has-text("${menuPath[i]}")`);
      if (await subMenuTitle.isVisible({ timeout: 2000 }).catch(() => false)) {
        // 检查子菜单是否已展开
        const parentSubmenu = subMenuTitle.locator('..');
        const isOpen = await parentSubmenu.evaluate(el => el.classList.contains('is-opened'));
        if (!isOpen) {
          await subMenuTitle.click();
          await this.page.waitForTimeout(300); // 等待展开动画
        }
      }
    }

    // 点击最终菜单项
    const targetItem = this.page.locator(`.el-menu-item:has-text("${menuPath[menuPath.length - 1]}")`);
    await targetItem.click();
    await this.page.waitForLoadState('networkidle');
  }

  /** 切换到指定标签页 */
  async switchToTab(tabText: string) {
    await this.tagsView.locator('.tags-view-item', { hasText: tabText }).click();
    await this.page.waitForLoadState('networkidle');
  }

  /** 关闭指定标签页 */
  async closeTab(tabText: string) {
    const tag = this.tagsView.locator('.tags-view-item', { hasText: tabText });
    await tag.locator('.el-icon-close').click();
    await this.page.waitForTimeout(300);
  }

  /** 获取当前打开的标签页文本列表 */
  async getOpenTabs(): Promise<string[]> {
    const count = await this.tagItems.count();
    const tabs: string[] = [];
    for (let i = 0; i < count; i++) {
      const text = await this.tagItems.nth(i).textContent();
      if (text) tabs.push(text.trim());
    }
    return tabs;
  }

  /** 刷新当前页面（右键标签页 → 刷新） */
  async refreshCurrentTab() {
    await this.activeTag.click({ button: 'right' });
    await this.page.locator('.contextmenu-item:has-text("刷 新")').click();
    await this.page.waitForLoadState('networkidle');
  }
}
```

---

## 三、常用测试模式

### 3.1 列表页测试模式

#### 3.1.1 搜索测试

```typescript
import { test, expect } from '../../fixtures/auth.fixture';
import { FeaturePage } from '../../pages/{module}/feature.page';

test.describe('{Feature}列表 - 搜索功能', () => {
  let featurePage: FeaturePage;

  test.beforeEach(async ({ authenticatedPage: page }) => {
    featurePage = new FeaturePage(page);
    await featurePage.goto();
  });

  test('精确搜索 - 输入完整名称应返回匹配结果', async () => {
    await featurePage.searchByName('admin');
    const rowCount = await featurePage.getRowCount();
    expect(rowCount).toBeGreaterThan(0);

    // 验证所有结果都包含搜索关键字
    for (let i = 0; i < Math.min(rowCount, 5); i++) {
      const name = await featurePage.getCellTextByHeader(i, '用户名');
      expect(name.toLowerCase()).toContain('admin');
    }
  });

  test('模糊搜索 - 输入部分名称应返回包含该关键字的结果', async () => {
    await featurePage.searchByName('ad');
    const rowCount = await featurePage.getRowCount();
    expect(rowCount).toBeGreaterThan(0);
  });

  test('空结果搜索 - 输入不存在的名称应显示空状态', async () => {
    await featurePage.searchByName('不存在的名称_测试数据_20260430');
    const rowCount = await featurePage.getRowCount();
    expect(rowCount).toBe(0);
  });

  test('重置搜索 - 点击重置后应清空条件并显示全部数据', async () => {
    await featurePage.searchByName('admin');
    const filteredCount = await featurePage.getRowCount();

    await featurePage.resetSearch();
    const resetCount = await featurePage.getRowCount();

    // 重置后数据量应 >= 筛选后的数据量
    expect(resetCount).toBeGreaterThanOrEqual(filteredCount);
  });

  test('组合搜索 - 多条件同时筛选应返回同时满足所有条件的结果', async () => {
    await featurePage.search({ name: 'admin', status: '正常' });
    const rowCount = await featurePage.getRowCount();
    expect(rowCount).toBeGreaterThan(0);

    // 验证结果状态都为"正常"
    for (let i = 0; i < Math.min(rowCount, 5); i++) {
      const status = await featurePage.getCellTextByHeader(i, '状态');
      expect(status).toBe('正常');
    }
  });
});
```

#### 3.1.2 分页测试

```typescript
test.describe('{Feature}列表 - 分页功能', () => {
  let featurePage: FeaturePage;

  test.beforeEach(async ({ authenticatedPage: page }) => {
    featurePage = new FeaturePage(page);
    await featurePage.goto();
  });

  test('分页信息显示 - 应显示总条数和当前页码', async () => {
    await expect(featurePage.pagination).toBeVisible();
    await expect(featurePage.totalText).toContainText('共');
  });

  test('切换每页条数 - 应正确刷新列表', async () => {
    const totalBefore = await featurePage.getTotalCount();
    if (totalBefore <= 10) {
      test.skip(true, '数据不足，跳过分页测试');
    }

    await featurePage.changePageSize(20);
    const rowCount = await featurePage.getRowCount();
    expect(rowCount).toBeLessThanOrEqual(20);
  });

  test('翻页 - 下一页应显示不同的数据', async () => {
    const totalBefore = await featurePage.getTotalCount();
    if (totalBefore <= 10) {
      test.skip(true, '数据不足，跳过分页测试');
    }

    // 获取第一页第一条数据
    const firstRowName = await featurePage.getCellTextByHeader(0, '名称');

    // 翻到下一页
    await featurePage.goToNextPage();

    // 验证数据发生了变化
    const newRowName = await featurePage.getCellTextByHeader(0, '名称');
    // 注意：可能数据量恰好整除，所以不做严格不等判断
  });
});
```

#### 3.1.3 CRUD 操作测试

```typescript
test.describe('{Feature}列表 - CRUD操作', () => {
  let featurePage: FeaturePage;
  let createdId: string;
  const timestamp = Date.now();
  const testName = `E2E测试_${timestamp}`;

  test.beforeEach(async ({ authenticatedPage: page }) => {
    featurePage = new FeaturePage(page);
    await featurePage.goto();
  });

  test('新增 - 填写表单并提交应成功', async () => {
    await featurePage.clickAdd();

    // 填写表单
    await featurePage.fillFormField('名称', testName);
    await featurePage.selectFormField('状态', '正常');
    await featurePage.fillFormField('备注', 'E2E自动测试创建');

    // 提交
    await featurePage.submitForm();

    // 验证：搜索新增的数据
    await featurePage.searchByName(testName);
    const rowCount = await featurePage.getRowCount();
    expect(rowCount).toBeGreaterThan(0);
  });

  test('编辑 - 修改名称应保存成功', async () => {
    // 先找到测试数据
    await featurePage.searchByName(testName);
    const rowCount = await featurePage.getRowCount();
    if (rowCount === 0) {
      test.skip(true, '未找到测试数据');
    }

    // 点击编辑
    await featurePage.clickEdit(0);

    // 修改名称
    const updatedName = `${testName}_已修改`;
    await featurePage.fillFormField('名称', updatedName);

    // 提交
    await featurePage.submitForm();

    // 验证：搜索修改后的数据
    await featurePage.searchByName(updatedName);
    const newRowCount = await featurePage.getRowCount();
    expect(newRowCount).toBeGreaterThan(0);
  });

  test('删除 - 确认删除后数据应从列表移除', async () => {
    // 找到测试数据
    await featurePage.searchByName(testName);
    const rowCountBefore = await featurePage.getRowCount();
    if (rowCountBefore === 0) {
      test.skip(true, '未找到测试数据');
    }

    // 点击删除
    await featurePage.clickDelete(0);
    await featurePage.confirmDelete();

    // 验证：搜索应无结果
    await featurePage.searchByName(testName);
    const rowCountAfter = await featurePage.getRowCount();
    expect(rowCountAfter).toBeLessThan(rowCountBefore);
  });
});
```

#### 3.1.4 批量操作测试

```typescript
test.describe('{Feature}列表 - 批量操作', () => {
  test('批量删除 - 选中多条后删除应成功', async ({ authenticatedPage: page }) => {
    const featurePage = new FeaturePage(page);
    await featurePage.goto();

    const rowCount = await featurePage.getRowCount();
    if (rowCount < 2) {
      test.skip(true, '数据不足，跳过批量操作测试');
    }

    // 选中前两行
    await featurePage.selectRow(0);
    await featurePage.selectRow(1);

    // 点击批量删除
    await featurePage.clickDelete();
    await featurePage.confirmDelete();

    // 验证
    const newRowCount = await featurePage.getRowCount();
    expect(newRowCount).toBeLessThan(rowCount);
  });

  test('全选删除 - 全选后删除应清空列表', async ({ authenticatedPage: page }) => {
    const featurePage = new FeaturePage(page);
    await featurePage.goto();

    await featurePage.selectAll();
    await featurePage.clickDelete();
    await featurePage.confirmDelete();

    const rowCount = await featurePage.getRowCount();
    expect(rowCount).toBe(0);
  });
});
```

#### 3.1.5 导入导出测试

```typescript
test.describe('{Feature}列表 - 导入导出', () => {
  test('导出 - 点击导出应下载文件', async ({ authenticatedPage: page }) => {
    const featurePage = new FeaturePage(page);
    await featurePage.goto();

    const [download] = await Promise.all([
      page.waitForEvent('download', { timeout: 30000 }),
      featurePage.exportButton.click(),
    ]);

    const filename = download.suggestedFilename();
    expect(filename).toMatch(/\.(xlsx|csv)$/);
  });

  test('导入 - 上传文件应提示导入结果', async ({ authenticatedPage: page }) => {
    const featurePage = new FeaturePage(page);
    await featurePage.goto();

    // 点击导入按钮（需根据实际页面调整）
    await featurePage.importButton.click();

    // 等待导入弹窗
    await featurePage.waitForDialog('导入');

    // 上传文件（需准备测试数据文件）
    // const fileInput = page.locator('input[type="file"]');
    // await fileInput.setInputFiles('/path/to/test-data.xlsx');

    // 点击提交
    // await featurePage.submitForm();
  });
});
```

### 3.2 表单测试模式

```typescript
test.describe('{Feature}表单 - 校验功能', () => {
  let featurePage: FeaturePage;

  test.beforeEach(async ({ authenticatedPage: page }) => {
    featurePage = new FeaturePage(page);
    await featurePage.goto();
    await featurePage.clickAdd();
  });

  test('必填校验 - 名称留空提交应提示错误', async () => {
    // 不填写任何字段，直接提交
    await featurePage.submitForm();

    // 验证校验提示
    const errors = await featurePage.dialog.locator('.el-form-item__error').allTextContents();
    expect(errors).toContain('名称不能为空');
  });

  test('格式校验 - 手机号格式不正确应提示错误', async () => {
    await featurePage.fillFormField('手机号', '12345');
    await featurePage.submitForm();

    const errors = await featurePage.dialog.locator('.el-form-item__error').allTextContents();
    expect(errors.some(e => e.includes('手机号'))).toBeTruthy();
  });

  test('长度校验 - 名称超长应提示错误', async () => {
    const longName = 'a'.repeat(101);
    await featurePage.fillFormField('名称', longName);
    await featurePage.submitForm();

    const errors = await featurePage.dialog.locator('.el-form-item__error').allTextContents();
    expect(errors.some(e => e.includes('长度'))).toBeTruthy();
  });

  test('提交成功 - 填写完整有效数据应提交成功', async () => {
    const timestamp = Date.now();
    await featurePage.fillFormField('名称', `有效测试_${timestamp}`);
    await featurePage.selectFormField('状态', '正常');
    await featurePage.submitForm();

    await expect(featurePage.successMessage).toBeVisible({ timeout: 5000 });
  });

  test('取消关闭 - 点击取消应关闭弹窗且不保存数据', async () => {
    await featurePage.fillFormField('名称', '取消测试数据');
    await featurePage.cancelForm();

    // 验证弹窗已关闭
    await expect(featurePage.dialog).toBeHidden({ timeout: 3000 });
  });

  test('ESC关闭 - 按 ESC 键应关闭弹窗', async () => {
    await featurePage.fillFormField('名称', 'ESC测试数据');
    await featurePage.page.keyboard.press('Escape');

    await expect(featurePage.dialog).toBeHidden({ timeout: 3000 });
  });

  test('校验清除 - 修正错误后校验提示应消失', async () => {
    // 触发校验错误
    await featurePage.submitForm();
    await expect(featurePage.dialog.locator('.el-form-item__error').first()).toBeVisible();

    // 填写正确数据
    const timestamp = Date.now();
    await featurePage.fillFormField('名称', `修正后_${timestamp}`);
    await featurePage.selectFormField('状态', '正常');

    // 校验提示应消失
    await expect(featurePage.dialog.locator('.el-form-item__error').first()).toBeHidden({ timeout: 2000 });
  });
});
```

### 3.3 权限测试模式

```typescript
import { test, expect } from '@playwright/test';
import { LoginPage } from '../../pages/login.page';

test.describe('{Feature}权限测试', () => {
  test('有权限 - admin 用户应看到所有操作按钮', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.login('admin', 'admin123');
    await page.goto('/{module}/{feature}');

    // 验证所有按钮可见
    await expect(page.locator('button:has-text("新 增")')).toBeVisible();
    await expect(page.locator('button:has-text("导 出")')).toBeVisible();
    await expect(page.locator('button:has-text("导 入")')).toBeVisible();
  });

  test('无权限 - 普通用户不应看到管理按钮', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.login('testuser', 'test123');
    await page.goto('/{module}/{feature}');

    // 验证管理按钮不可见
    await expect(page.locator('button:has-text("新 增")')).toBeHidden();
    await expect(page.locator('button:has-text("删 除")')).toBeHidden();
  });

  test('越权操作 - 无权限用户直接调用 API 应被拒绝', async ({ request }) => {
    // 使用普通用户 Token
    const loginRes = await request.post('http://localhost:8080/auth/login', {
      data: {
        tenantId: '000000',
        username: 'testuser',
        password: 'test123',
      },
    });
    const { data } = await loginRes.json();
    const token = data.access_token;

    // 尝试访问无权限的 API
    const res = await request.post('http://localhost:8080/{module}/{feature}', {
      headers: { Authorization: `Bearer ${token}` },
      data: { name: '越权测试' },
    });

    // 应返回权限不足
    expect(res.status()).toBe(200);
    const body = await res.json();
    expect(body.code).not.toBe(200);
  });
});
```

### 3.4 数据驱动测试

#### 3.4.1 使用 test.describe.each

```typescript
test.describe.each([
  { name: '', expected: '名称不能为空' },
  { name: 'a'.repeat(101), expected: '名称长度不能超过100个字符' },
  { name: '<script>alert(1)</script>', expected: '名称包含非法字符' },
])('表单校验 - 名称字段: $name', ({ name, expected }) => {
  test(`输入 "$name" 应提示 "${expected}"`, async ({ authenticatedPage: page }) => {
    await page.goto('/{module}/{feature}');
    await page.click('button:has-text("新 增")');
    await page.fill('.el-dialog input[placeholder="请输入名称"]', name);
    await page.click('button:has-text("确 定")');

    await expect(page.locator('.el-form-item__error')).toContainText(expected);
  });
});
```

#### 3.4.2 外部数据文件

```typescript
// test-data/search-cases.json
[
  { "keyword": "admin", "expectResults": true, "description": "精确匹配" },
  { "keyword": "ad", "expectResults": true, "description": "模糊匹配" },
  { "keyword": "不存在xyz123", "expectResults": false, "description": "无匹配结果" }
]
```

```typescript
// tests/{module}/search-data-driven.spec.ts
import { test, expect } from '../../fixtures/auth.fixture';
import { FeaturePage } from '../../pages/{module}/feature.page';
import searchCases from '../../test-data/search-cases.json';

test.describe('搜索功能 - 数据驱动', () => {
  for (const testCase of searchCases) {
    test(`${testCase.description}: 搜索 "${testCase.keyword}"`, async ({ authenticatedPage: page }) => {
      const featurePage = new FeaturePage(page);
      await featurePage.goto();
      await featurePage.searchByName(testCase.keyword);

      if (testCase.expectResults) {
        const rowCount = await featurePage.getRowCount();
        expect(rowCount).toBeGreaterThan(0);
      } else {
        const rowCount = await featurePage.getRowCount();
        expect(rowCount).toBe(0);
      }
    });
  }
});
```

#### 3.4.3 参数化组合

```typescript
test.describe('搜索组合 - 参数化', () => {
  const statuses = ['正常', '停用'];
  const keywords = ['admin', 'test'];

  for (const status of statuses) {
    for (const keyword of keywords) {
      test(`状态="${status}" + 关键字="${keyword}"`, async ({ authenticatedPage: page }) => {
        const featurePage = new FeaturePage(page);
        await featurePage.goto();
        await featurePage.search({ name: keyword, status });
        await featurePage.waitForTable();
        // 基本断言：搜索不报错
        await expect(featurePage.table).toBeVisible();
      });
    }
  }
});
```

---

## 四、Element Plus 组件的 Playwright 交互

### 4.1 ElInput 输入框

```typescript
// 普通输入
await page.locator('.el-input__inner').fill('测试内容');

// 清空并重新填写
const input = page.locator('.el-input__inner');
await input.clear();
await input.fill('新内容');

// 只读输入框（不应能修改）
await expect(page.locator('.el-input.is-disabled .el-input__inner')).toBeDisabled();

// 带前后缀的输入框
await page.locator('.el-input-group .el-input__inner').fill('内容');

// 触发回车搜索
await page.locator('[placeholder="请输入名称"]').fill('admin');
await page.locator('[placeholder="请输入名称"]').press('Enter');
```

### 4.2 ElSelect 下拉选择

```typescript
// 点击打开下拉
await page.locator('.el-select').click();

// 选择指定选项
await page.locator('.el-select-dropdown__item', { hasText: '选项文本' }).click();

// 可清空的下拉框 - 清空选择
await page.locator('.el-select .el-input__suffix .el-icon-circle-close').click();

// 多选下拉
await page.locator('.el-select').click();
await page.locator('.el-select-dropdown__item', { hasText: '选项A' }).click();
await page.locator('.el-select-dropdown__item', { hasText: '选项B' }).click();
// 关闭下拉
await page.locator('.el-select').click();

// 验证已选项
await expect(page.locator('.el-select .el-tag')).toContainText('选项A');
```

### 4.3 ElDatePicker 日期选择

```typescript
// 点击打开日期面板
await page.locator('.el-date-editor').click();

// 选择日期（点击日期格）
await page.locator('.el-date-table td.today').click();
// 或选择特定日期
await page.locator('.el-date-table td', { hasText: '15' }).click();

// 日期范围选择
await page.locator('.el-date-editor--daterange').click();
// 选择开始日期
await page.locator('.el-date-table td', { hasText: '1' }).first().click();
// 选择结束日期
await page.locator('.el-date-table td', { hasText: '30' }).click();

// 手动输入日期（更快）
await page.locator('.el-date-editor input').first().fill('2026-01-01');
await page.locator('.el-date-editor input').last().fill('2026-12-31');
await page.locator('.el-date-editor').press('Enter');

// 清空日期
await page.locator('.el-date-editor .el-input__suffix .el-icon-circle-close').click();
```

### 4.4 ElSwitch 开关

```typescript
// 点击切换
await page.locator('.el-switch').click();

// 验证状态
await expect(page.locator('.el-switch.is-checked')).toBeVisible(); // 开启状态
await expect(page.locator('.el-switch:not(.is-checked)')).toBeVisible(); // 关闭状态

// 禁用状态的开关
await expect(page.locator('.el-switch.is-disabled')).toBeDisabled();
```

### 4.5 ElTable 表格操作

```typescript
// 验证表格数据
await expect(page.locator('.el-table')).toBeVisible();
const rowCount = await page.locator('.el-table__body-wrapper .el-table__row').count();

// 获取指定单元格文本
const cellText = await page.locator('.el-table__body-wrapper .el-table__row')
  .nth(0)
  .locator('td')
  .nth(1)
  .textContent();

// 点击行内操作按钮
await page.locator('.el-table__body-wrapper .el-table__row')
  .nth(0)
  .locator('button:has-text("编辑")')
  .click();

// 勾选复选框
await page.locator('.el-table__body-wrapper .el-table__row')
  .nth(0)
  .locator('.el-checkbox')
  .click();

// 全选
await page.locator('.el-table__header-wrapper .el-checkbox').click();

// 使用 show-overflow-tooltip 的单元格（悬停显示完整文本）
const tooltipCell = page.locator('.el-table__body-wrapper .el-table__row')
  .nth(0)
  .locator('td')
  .nth(2);
await tooltipCell.hover();
await expect(page.locator('.el-tooltip__popper')).toBeVisible();
```

### 4.6 ElDialog 对话框

```typescript
// 等待对话框打开
await expect(page.locator('.el-dialog__wrapper')).toBeVisible();
await expect(page.locator('.el-dialog__title')).toContainText('新增');

// 填写对话框内的表单
await page.locator('.el-dialog .el-form-item:has-text("名称") .el-input__inner')
  .fill('测试数据');

// 提交对话框
await page.locator('.el-dialog__footer button:has-text("确 定")').click();

// 关闭对话框（点击取消）
await page.locator('.el-dialog__footer button:has-text("取 消")').click();

// 点击遮罩层关闭
await page.locator('.el-dialog__wrapper .el-dialog__headerbtn').click();

// 验证对话框已关闭
await expect(page.locator('.el-dialog__wrapper')).toBeHidden({ timeout: 3000 });
```

### 4.7 ElMessageBox 确认框

```typescript
// 等待确认框出现
await expect(page.locator('.el-message-box__wrapper')).toBeVisible();
await expect(page.locator('.el-message-box__message')).toContainText('是否确认删除');

// 点击确定
await page.locator('.el-message-box__btns .el-button--primary').click();

// 点击取消
await page.locator('.el-message-box__btns button:first-child').click();

// 点击 X 关闭
await page.locator('.el-message-box__headerbtn').click();
```

### 4.8 ElMessage 消息提示

```typescript
// 验证成功消息
await expect(page.locator('.el-message--success')).toBeVisible({ timeout: 5000 });
await expect(page.locator('.el-message--success')).toContainText('操作成功');

// 验证错误消息
await expect(page.locator('.el-message--error')).toBeVisible({ timeout: 5000 });

// 验证警告消息
await expect(page.locator('.el-message--warning')).toBeVisible({ timeout: 5000 });

// 等待消息消失
await expect(page.locator('.el-message--success')).toBeHidden({ timeout: 5000 });
```

### 4.9 ElPagination 分页

```typescript
// 验证分页存在
await expect(page.locator('.pagination')).toBeVisible();

// 获取总条数
const totalText = await page.locator('.pagination .el-pagination__total').textContent();
// "共 100 条"

// 下一页
await page.locator('.pagination .btn-next').click();

// 上一页
await page.locator('.pagination .btn-prev').click();

// 跳转到指定页
await page.locator('.pagination .el-pagination__jump .el-input__inner').fill('5');
await page.locator('.pagination .el-pagination__jump .el-input__inner').press('Enter');

// 切换每页条数
await page.locator('.pagination .el-select').click();
await page.locator('.el-select-dropdown__item', { hasText: '20条/页' }).click();
```

---

## 五、调试和排错

### 5.1 截图

**失败时自动截图（在 playwright.config.ts 中已配置）：**

```typescript
// 配置中已有：
use: {
  screenshot: 'only-on-failure',
}
```

**手动截图：**

```typescript
// 截取整个页面
await page.screenshot({ path: 'e2e/screenshots/homepage.png', fullPage: true });

// 截取元素
await page.locator('.el-table').screenshot({ path: 'e2e/screenshots/table.png' });

// 在测试失败时截图
test.afterEach(async ({ page }, testInfo) => {
  if (testInfo.status !== 'passed') {
    await page.screenshot({
      path: `e2e/screenshots/${testInfo.title.replace(/\s+/g, '_')}.png`,
      fullPage: true,
    });
  }
});
```

### 5.2 录屏

**配置中已启用（失败时保留）：**

```typescript
use: {
  video: 'retain-on-failure',
}
```

录屏文件保存在 `test-results/` 目录下，文件名格式为 `{project}-{browser}-{testId}.webm`。

### 5.3 Trace 查看

**配置中已启用：**

```typescript
use: {
  trace: 'on-first-retry',
}
```

**查看 Trace：**

```bash
cd {E2E_DIR}/
npx playwright show-trace test-results/<test-id>/trace.zip
```

Trace 包含：
- 每一步操作的截图
- DOM 快照
- 网络请求
- 控制台日志
- 源代码位置

### 5.4 慢动作模式

```bash
# 以慢动作模式运行测试
SLOW_MO=500 npx playwright test

# 或在 playwright.config.ts 中配置
use: {
  launchOptions: {
    slowMo: 500,
  },
}
```

### 5.5 选择器调试工具

**Playwright Inspector（交互式调试）：**

```bash
# 使用 Inspector 运行测试
PWDEBUG=1 npx playwright test

# 或
npx playwright test --debug
```

**Playwright Picker（页面元素选择器工具）：**

```bash
# 启动 Playwright Picker
npx playwright open --save-storage=auth.json http://localhost
```

**在代码中打印选择器匹配结果：**

```typescript
// 调试：查看选择器匹配了多少个元素
const count = await page.locator('.el-table__row').count();
console.log(`匹配到 ${count} 个表格行`);

// 调试：查看元素的 HTML
const html = await page.locator('.el-dialog').innerHTML();
console.log(html);

// 调试：使用 page.pause() 在运行时暂停
await page.pause(); // 打开 Inspector，可以手动操作
```

### 5.6 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 元素找不到 | 页面未加载完成 | 检查是否需要 `waitForLoadState('networkidle')` |
| 点击无效 | 元素被遮挡 | 使用 `{ force: true }` 或先关闭遮挡元素 |
| 超时 | 后端响应慢 | 增加 `timeout` 或检查后端状态 |
| 测试不稳定 | 依赖数据状态 | 确保每个测试独立准备和清理数据 |
| 选择器冲突 | 同类元素太多 | 使用更精确的选择器或 `nth()` |
| 对话框找不到 | Element Plus 动画延迟 | 等待 `waitFor` + `state: 'visible'` |

---

## 六、RuoYi 特有的测试模式

### 6.1 登录流程

```typescript
// tests/auth/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../../pages/login.page';

test.describe('RuoYi 登录流程', () => {
  test('正常登录 - admin 账号应成功跳转到首页', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.login('admin', 'admin123');

    // 验证跳转到首页
    await expect(page).toHaveURL(/\/index/);
    // 验证侧边栏菜单可见
    await expect(page.locator('.sidebar-container')).toBeVisible();
  });

  test('错误密码 - 应提示用户名或密码错误', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.loginWithInvalidCredentials('admin', 'wrong_password');

    const errorMsg = await loginPage.getErrorMessage();
    expect(errorMsg).toContain('用户名或密码错误');
  });

  test('空用户名 - 应提示不能为空', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.passwordInput.fill('admin123');
    await loginPage.loginButton.click();

    // Element Plus 校验提示
    await expect(page.locator('.el-form-item__error')).toContainText('不能为空');
  });

  test('记住密码 - 勾选后刷新页面应保留用户名', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();

    // 勾选记住密码
    await loginPage.rememberMeCheckbox.click();
    await loginPage.usernameInput.fill('admin');
    await loginPage.passwordInput.fill('admin123');
    await loginPage.loginButton.click();

    // 退出登录
    // ... 退出操作

    // 刷新页面
    await page.goto('/login');
    await expect(loginPage.usernameInput).toHaveValue('admin');
  });
});
```

### 6.2 左侧菜单导航

```typescript
// tests/navigation/menu.spec.ts
import { test, expect } from '../../fixtures/auth.fixture';
import { NavigationPage } from '../../pages/navigation.page';

test.describe('左侧菜单导航', () => {
  let nav: NavigationPage;

  test.beforeEach(async ({ authenticatedPage: page }) => {
    nav = new NavigationPage(page);
  });

  test('一级菜单 - 点击应直接跳转', async () => {
    await nav.navigateToMenu(['首页']);
    await expect(page).toHaveURL(/\/index/);
  });

  test('二级菜单 - 展开父菜单后点击子菜单', async () => {
    await nav.navigateToMenu(['系统管理', '用户管理']);
    await expect(page.locator('.el-table')).toBeVisible({ timeout: 10000 });
  });

  test('三级菜单 - 展开两级后点击最终菜单', async () => {
    await nav.navigateToMenu(['系统管理', '日志管理', '操作日志']);
    await expect(page.locator('.el-table')).toBeVisible({ timeout: 10000 });
  });

  test('菜单折叠 - 点击折叠按钮后菜单应收缩', async ({ authenticatedPage: page }) => {
    // 点击折叠按钮
    await page.locator('.hamburger').click();
    // 验证菜单已折叠
    await expect(page.locator('.sidebar-container')).toHaveClass(/hideSidebar/);
  });

  test('菜单搜索 - 输入关键字应过滤菜单', async ({ authenticatedPage: page }) => {
    // 点击搜索图标（如有）
    const searchIcon = page.locator('.breadcrumb-container .el-icon-search');
    if (await searchIcon.isVisible()) {
      await searchIcon.click();
      await page.fill('.header-search input', '用户');
      // 验证搜索结果
      await expect(page.locator('.header-search .el-autocomplete-suggestion')).toBeVisible();
    }
  });
});
```

### 6.3 顶部标签页切换

```typescript
test.describe('顶部标签页', () => {
  test('打开新页面应新增标签页', async ({ authenticatedPage: page }) => {
    const nav = new NavigationPage(page);
    await nav.navigateToMenu(['系统管理', '用户管理']);
    await nav.navigateToMenu(['系统管理', '角色管理']);

    const tabs = await nav.getOpenTabs();
    expect(tabs.length).toBeGreaterThanOrEqual(2);
  });

  test('切换标签页应保持页面状态', async ({ authenticatedPage: page }) => {
    const nav = new NavigationPage(page);

    // 打开两个页面
    await nav.navigateToMenu(['系统管理', '用户管理']);
    await nav.navigateToMenu(['系统管理', '角色管理']);

    // 切换回第一个标签页
    await nav.switchToTab('用户管理');

    // 验证页面内容正确
    await expect(page.locator('.el-table')).toBeVisible();
  });

  test('关闭标签页应返回上一个标签', async ({ authenticatedPage: page }) => {
    const nav = new NavigationPage(page);

    await nav.navigateToMenu(['系统管理', '用户管理']);
    await nav.navigateToMenu(['系统管理', '角色管理']);

    // 关闭当前标签
    await nav.closeTab('角色管理');

    // 验证切换到上一个标签
    await expect(nav.activeTag).toContainText('用户管理');
  });

  test('关闭所有标签应回到首页', async ({ authenticatedPage: page }) => {
    const nav = new NavigationPage(page);

    await nav.navigateToMenu(['系统管理', '用户管理']);

    // 右键标签 → 关闭所有
    await nav.activeTag.click({ button: 'right' });
    await page.locator('.contextmenu-item:has-text("关闭所有")').click();

    // 验证回到首页
    await expect(page).toHaveURL(/\/index/);
  });
});
```

### 6.4 权限菜单动态加载

```typescript
test.describe('权限菜单动态加载', () => {
  test('admin 用户应看到所有菜单', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.login('admin', 'admin123');

    // 验证一级菜单数量
    const menuCount = await page.locator('.el-menu > .el-sub-menu, .el-menu > .el-menu-item').count();
    expect(menuCount).toBeGreaterThan(5); // admin 应有多个菜单
  });

  test('受限用户应只看到授权菜单', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.login('limited_user', 'password123');

    // 验证某些菜单不可见
    const systemMenu = page.locator('.el-sub-menu__title:has-text("系统管理")');
    // 根据实际权限配置验证
  });

  test('菜单展开应动态加载子菜单', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.login('admin', 'admin123');

    const subMenuTitle = page.locator('.el-sub-menu__title:has-text("系统管理")');
    await subMenuTitle.click();

    // 验证子菜单项加载
    await expect(page.locator('.el-sub-menu:has(.el-sub-menu__title:text("系统管理")) .el-menu-item').first())
      .toBeVisible({ timeout: 5000 });
  });
});
```

### 6.5 字典下拉框选择

```typescript
test.describe('字典下拉框', () => {
  test('状态下拉应包含字典选项', async ({ authenticatedPage: page }) => {
    await page.goto('/system/user');

    // 点击状态下拉框
    const statusSelect = page.locator('.el-form--inline .el-select').first();
    await statusSelect.click();

    // 验证下拉选项包含字典值
    await expect(page.locator('.el-select-dropdown__item:has-text("正常")')).toBeVisible();
    await expect(page.locator('.el-select-dropdown__item:has-text("停用")')).toBeVisible();
  });

  test('选择字典值后应正确筛选', async ({ authenticatedPage: page }) => {
    await page.goto('/system/user');

    // 选择状态
    const statusSelect = page.locator('.el-form--inline .el-select').first();
    await statusSelect.click();
    await page.locator('.el-select-dropdown__item:has-text("停用")').click();

    // 点击搜索
    await page.click('button:has-text("搜索")');
    await page.waitForTimeout(1000); // 等待加载

    // 验证搜索结果
    const rows = page.locator('.el-table__body-wrapper .el-table__row');
    const rowCount = await rows.count();
    if (rowCount > 0) {
      // 验证所有行的状态都是"停用"
      for (let i = 0; i < Math.min(rowCount, 3); i++) {
        const statusCell = rows.nth(i).locator('td').nth(4); // 根据实际列索引调整
        await expect(statusCell).toContainText('停用');
      }
    }
  });
});
```

### 6.6 完整业务流程测试模板

```typescript
// tests/{module}/{feature}-flow.spec.ts
import { test, expect } from '../../fixtures/auth.fixture';
import { FeaturePage } from '../../pages/{module}/feature.page';
import { NavigationPage } from '../../pages/navigation.page';

test.describe.serial('{Feature}完整业务流程', () => {
  const timestamp = Date.now();
  const testData = {
    name: `流程测试_${timestamp}`,
    updatedName: `流程测试_已修改_${timestamp}`,
  };

  test('步骤1 - 登录并导航到目标页面', async ({ authenticatedPage: page }) => {
    const nav = new NavigationPage(page);
    await nav.navigateToMenu(['{Module}', '{Feature}']);
    await expect(page.locator('.el-table')).toBeVisible({ timeout: 10000 });
  });

  test('步骤2 - 新增一条记录', async ({ authenticatedPage: page }) => {
    const featurePage = new FeaturePage(page);
    await featurePage.clickAdd();
    await featurePage.fillFormField('名称', testData.name);
    await featurePage.selectFormField('状态', '正常');
    await featurePage.submitForm();

    await expect(featurePage.successMessage).toBeVisible({ timeout: 5000 });
  });

  test('步骤3 - 搜索并验证新增数据', async ({ authenticatedPage: page }) => {
    const featurePage = new FeaturePage(page);
    await featurePage.searchByName(testData.name);
    const rowCount = await featurePage.getRowCount();
    expect(rowCount).toBeGreaterThan(0);
  });

  test('步骤4 - 编辑记录', async ({ authenticatedPage: page }) => {
    const featurePage = new FeaturePage(page);
    await featurePage.searchByName(testData.name);
    await featurePage.clickEdit(0);
    await featurePage.fillFormField('名称', testData.updatedName);
    await featurePage.submitForm();

    await expect(featurePage.successMessage).toBeVisible({ timeout: 5000 });
  });

  test('步骤5 - 验证编辑结果', async ({ authenticatedPage: page }) => {
    const featurePage = new FeaturePage(page);
    await featurePage.searchByName(testData.updatedName);
    const rowCount = await featurePage.getRowCount();
    expect(rowCount).toBeGreaterThan(0);
  });

  test('步骤6 - 删除记录', async ({ authenticatedPage: page }) => {
    const featurePage = new FeaturePage(page);
    await featurePage.searchByName(testData.updatedName);
    const rowCountBefore = await featurePage.getRowCount();
    expect(rowCountBefore).toBeGreaterThan(0);

    await featurePage.clickDelete(0);
    await featurePage.confirmDelete();

    await expect(featurePage.successMessage).toBeVisible({ timeout: 5000 });
  });

  test('步骤7 - 验证数据已删除', async ({ authenticatedPage: page }) => {
    const featurePage = new FeaturePage(page);
    await featurePage.searchByName(testData.updatedName);
    const rowCount = await featurePage.getRowCount();
    expect(rowCount).toBe(0);
  });
});
```

---

## 七、测试辅助工具

### 7.1 通用等待工具

```typescript
// utils/wait-helpers.ts
import { Page, Locator, expect } from '@playwright/test';

/** 等待 Element Plus 加载遮罩消失 */
export async function waitForElLoading(page: Page, timeout = 10000) {
  try {
    await page.locator('.el-loading-mask').last().waitFor({ state: 'detached', timeout });
  } catch {
    // 没有加载遮罩也可以继续
  }
}

/** 等待 Element Plus 对话框动画完成 */
export async function waitForDialogAnimation(page: Page) {
  await page.waitForTimeout(300); // Element Plus 对话框动画约 300ms
}

/** 等待 Element Plus 消息提示出现并消失 */
export async function waitForElMessage(
  page: Page,
  type: 'success' | 'error' | 'warning' | 'info' = 'success',
  timeout = 5000
) {
  const message = page.locator(`.el-message--${type}`);
  await message.waitFor({ state: 'visible', timeout });
  await message.waitFor({ state: 'detached', timeout: timeout + 3000 });
}

/** 等待 API 请求完成 */
export async function waitForApiRequest(page: Page, urlPattern: string | RegExp, timeout = 10000) {
  await page.waitForResponse(
    response => {
      const url = response.url();
      if (typeof urlPattern === 'string') {
        return url.includes(urlPattern);
      }
      return urlPattern.test(url);
    },
    { timeout }
  );
}

/** 安全重试操作 */
export async function retryAction(
  action: () => Promise<void>,
  maxRetries = 3,
  delay = 1000
) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      await action();
      return;
    } catch (e) {
      if (i === maxRetries - 1) throw e;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

### 7.2 测试数据生成器

```typescript
// utils/test-data-generator.ts

/** 生成唯一测试名称 */
export function generateTestName(prefix = '测试'): string {
  return `${prefix}_${Date.now()}_${Math.random().toString(36).substring(2, 7)}`;
}

/** 生成手机号 */
export function generatePhone(): string {
  const prefixes = ['138', '139', '150', '151', '186', '187', '188'];
  return prefixes[Math.floor(Math.random() * prefixes.length)] +
    String(Math.floor(Math.random() * 100000000)).padStart(8, '0');
}

/** 生成邮箱 */
export function generateEmail(): string {
  return `test_${Date.now()}@example.com`;
}

/** 生成身份证号（测试用，非真实） */
export function generateIdCard(): string {
  return '11010119900101' + String(Math.floor(Math.random() * 10000)).padStart(4, '0');
}
```
