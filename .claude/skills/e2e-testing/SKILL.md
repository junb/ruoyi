---
name: e2e-testing
description: |
  E2E端到端测试 — 前后端完整环境 + Playwright，覆盖完整业务流程验证、测试流水线编排、报告格式、失败处理与回归策略。
---

# E2E端到端测试

> 适配项目：RuoYi-Vue-Plus 5.6.0
> 后端：Spring Boot 3.5.12 + Java 17 + MyBatis-Plus + Sa-Token
> 前端：Vue 3 + Element Plus 2.13.5 + Pinia + UnoCSS + TypeScript

---

## 一、5层测试体系概览

```
┌─────────────────────────────────────────────────────────┐
│  第5层：AI高级测试（边界用例生成 + 变更影响分析）         │
├─────────────────────────────────────────────────────────┤
│  第4层：E2E端到端测试（前后端启动 + Playwright）         │
├─────────────────────────────────────────────────────────┤
│  第3层：UI自动化测试（Playwright 页面交互验证）           │
├─────────────────────────────────────────────────────────┤
│  第2层：集成测试（Spring Boot Test + API联调测试）       │
├─────────────────────────────────────────────────────────┤
│  第1层：单元测试（后端 mvn test + 前端 npx vitest）     │
└─────────────────────────────────────────────────────────┘
```

| 层级 | 名称 | 工具 | 执行时间 | 覆盖范围 |
|------|------|------|----------|----------|
| Layer 1 | 单元测试 | JUnit 5 + Mockito / Vitest | 秒级 | 单个函数/组件 |
| Layer 2 | 集成测试 | Spring Boot Test / API测试 | 分钟级 | 模块间协作 |
| Layer 3 | UI自动化测试 | Playwright | 分钟级 | 页面交互 |
| Layer 4 | E2E端到端测试 | Playwright + 完整环境 | 分钟级 | 完整业务流程 |
| Layer 5 | AI高级测试 | Agent分析 + 代码生成 | 分钟级 | 边界/异常场景 |

> **说明：** 各层测试由独立Skill负责——
> - L1 单元测试：`junit-testing`（后端）、`vitest-testing`（前端）
> - L2 集成测试：`integration-testing`
> - L3 UI自动化测试：`ui-testing`
> - L4 E2E端到端测试：`e2e-testing`（本Skill）
> - L5 AI高级测试：`ai-test-gen`

---

## 二、本地测试环境

### 2.1 项目路径

| 组件 | 路径 |
|------|------|
| 后端 | `{BACKEND_DIR}/` |
| 前端 | `{FRONTEND_DIR}/` |
| E2E测试 | `{E2E_DIR}/` |
| 文档 | `{DOC_DIR}/` |

### 2.2 后端环境

```bash
# 启动后端（开发环境，端口 8080）
cd {BACKEND_DIR}/
mvn spring-boot:run -pl ruoyi-admin

# 或指定profile
mvn spring-boot:run -pl ruoyi-admin -Dspring-boot.run.profiles=dev
```

- **端口：** 8080
- **上下文路径：** `/`
- **数据库：** MySQL（由 `application-druid.yml` 配置）
- **Redis：** 默认 localhost:6379

### 2.3 前端环境

```bash
# 安装依赖（首次）
cd {FRONTEND_DIR}/
npm install

# 启动前端开发服务器（端口 80）
npm run dev
```

- **端口：** 80（`.env.development` 中 `VITE_APP_PORT=80`）
- **API代理：** `/dev-api` → `http://localhost:8080`
- **访问地址：** `http://localhost`

### 2.4 测试数据库配置

测试时使用开发数据库，无需额外配置。如需独立测试数据库，在 `application-druid.yml` 中添加 `test` profile。

### 2.5 后端模块清单

```
ruoyi-modules/
├── ruoyi-system/      # 系统管理（用户、角色、菜单、部门等）
├── ruoyi-demo/        # 示例模块
├── ruoyi-generator/   # 代码生成
├── ruoyi-inspection/  # 巡检模块（业务核心）
├── ruoyi-job/         # 定时任务
├── ruoyi-knowledge/   # 知识库
└── ruoyi-workflow/    # 工作流
```

### 2.6 环境前置检查

```bash
# 检查 Java 版本（需 Java 17）
java -version

# 检查 Maven
mvn -version

# 检查 Node.js
node -version

# 检查 Redis 是否运行
redis-cli ping

# 检查 MySQL 是否运行
mysql -u root -p -e "SELECT 1"

# 检查端口占用
lsof -i :8080    # 后端
lsof -i :80      # 前端
```

---

## 三、E2E端到端测试

### 3.1 环境启动

```bash
# 终端1：启动后端
cd {BACKEND_DIR}/
mvn spring-boot:run -pl ruoyi-admin

# 终端2：启动前端
cd {FRONTEND_DIR}/
npm run dev

# 终端3：运行E2E测试
cd {E2E_DIR}/
npx playwright test
```

**自动启动配置（追加到 `playwright.config.ts`）：**

```typescript
webServer: [
  {
    command: 'cd {BACKEND_DIR} && mvn spring-boot:run -pl ruoyi-admin',
    port: 8080,
    reuseExistingServer: true,
    timeout: 120_000,
  },
  {
    command: 'cd {FRONTEND_DIR} && npm run dev',
    port: 80,
    reuseExistingServer: true,
    timeout: 60_000,
  },
],
```

### 3.2 E2E端到端流程测试模板

```typescript
// e2e/tests/{module}/{feature}-flow.spec.ts
import { test, expect } from '../../fixtures/auth.fixture';
import { waitForTableData } from '../../utils/helpers';

test.describe('{Feature}模块 - 端到端流程测试', () => {
  test('完整流程：登录 → 新增 → 查询 → 编辑 → 删除', async ({ authenticatedPage: page }) => {
    // ===== 步骤1：导航到列表页 =====
    await page.goto('/{module}/{feature}');
    await page.waitForLoadState('networkidle');
    await expect(page.locator('.el-table')).toBeVisible({ timeout: 15000 });

    // ===== 步骤2：点击新增 =====
    await page.click('button:has-text("新 增")');
    await expect(page.locator('.el-dialog__title')).toBeVisible();

    // ===== 步骤3：填写表单 =====
    const timestamp = Date.now();
    const testName = `E2E测试_${timestamp}`;

    // 根据实际表单字段调整
    await page.fill('.el-input__inner:first-child', testName);

    // ===== 步骤4：提交表单 =====
    await page.click('button:has-text("确 定")');
    await expect(page.locator('.el-message--success')).toBeVisible({ timeout: 5000 });

    // ===== 步骤5：查询验证 =====
    await page.fill('input[placeholder*="请输入"]', testName);
    await page.click('button:has-text("搜 索")');
    await page.waitForTimeout(1000);
    const row = page.locator('.el-table__body-wrapper .el-table__row').first();
    await expect(row).toContainText(testName);

    // ===== 步骤6：编辑 =====
    await row.locator('button:has-text("编 辑")').click();
    await expect(page.locator('.el-dialog__title')).toBeVisible();
    // 修改字段...
    await page.click('button:has-text("确 定")');
    await expect(page.locator('.el-message--success')).toBeVisible({ timeout: 5000 });

    // ===== 步骤7：删除 =====
    await row.locator('button:has-text("删 除")').click();
    await page.click('.el-message-box__btns button:has-text("确 定")');
    await expect(page.locator('.el-message--success')).toBeVisible({ timeout: 5000 });
  });
});
```

---

## 四、测试流水线编排

### 4.1 完整流水线流程

```
代码变更
  │
  ▼
检测变更文件（git diff）
  │
  ├─ 仅后端变更 ↓
  │   Layer 1: mvn test（对应模块）
  │   Layer 2: 集成测试
  │   Layer 5: AI边界测试
  │
  ├─ 仅前端变更 ↓
  │   Layer 1: npx vitest run
  │   Layer 3: UI测试
  │   Layer 5: AI边界测试
  │
  └─ 前后端都变更 ↓
      Layer 1: mvn test + npx vitest run
      Layer 2: 集成测试
      Layer 3: UI测试
      Layer 4: E2E测试
      Layer 5: AI边界测试
  │
  ▼
汇总报告 → PASS/FAIL
```

### 4.2 执行命令速查

```bash
# Layer 1: 单元测试
cd {BACKEND_DIR}
mvn test -pl ruoyi-modules/ruoyi-{module} -am
cd {FRONTEND_DIR} && npx vitest run

# Layer 2: 集成测试
cd {BACKEND_DIR}
mvn test -pl ruoyi-admin -Dtest=*IntegrationTest

# Layer 3: UI测试
cd {FRONTEND_DIR}
npx playwright test --project=ui

# Layer 4: E2E测试（需先启动前后端）
cd {FRONTEND_DIR}
npx playwright test --project=e2e

# Layer 5: AI高级测试（由Agent执行）
# → 分析变更 → 生成用例 → 执行
```

---

## 五、测试报告格式

### 5.1 单层测试报告

```markdown
# {Layer名称}测试报告

## 测试目标
{模块名/功能名}

## 测试结果：PASS / FAIL

### 统计
- 总用例数：{N}
- 通过：{X}
- 失败：{Y}
- 跳过：{Z}

### 失败详情（如有）
| # | 用例 | 错误信息 | 位置 |
|---|------|----------|------|
| 1 | ... | ... | ... |
```

### 5.2 汇总报告

```markdown
# 测试汇总报告

## 测试目标：{模块名}

| 层级 | 结果 | 用例数 | 通过 | 失败 | 耗时 |
|------|------|--------|------|------|------|
| L1 单元测试 | PASS/FAIL | ... | ... | ... | ... |
| L2 集成测试 | PASS/FAIL | ... | ... | ... | ... |
| L3 UI测试 | PASS/FAIL | ... | ... | ... | ... |
| L4 E2E测试 | PASS/FAIL | ... | ... | ... | ... |
| L5 AI高级测试 | PASS/FAIL | ... | ... | ... | ... |

## 最终判定：PASS / FAIL
```

---

## 六、失败处理

### 6.1 失败归因规则

| 失败层级 | 常见原因 | 建议方向 |
|----------|----------|----------|
| L1 单元测试 | 逻辑错误、Mock配置错误、断言不正确 | 检查业务逻辑和测试用例 |
| L2 集成测试 | 数据库数据问题、环境配置、API参数 | 检查测试数据和接口参数 |
| L3 UI测试 | 元素定位失败、异步加载未等待、浏览器兼容 | 更新选择器、增加等待 |
| L4 E2E测试 | 前后端接口不一致、状态依赖、数据残留 | 检查API契约和测试隔离 |
| L5 AI测试 | 边界条件遗漏、并发问题、权限组合遗漏 | 补充用例 |

### 6.2 回归测试策略

Bug修复后的回归范围：
- **最小回归**：只跑修复模块的L1+L2测试
- **标准回归**：跑修复模块的L1+L2+L3测试 + 关联模块的L1测试
- **完整回归**：跑全量L1+L2+L3+L4测试（重大Bug或架构变更时）

---

## 七、经验库写入规范

```markdown
## {类别}
- [{日期}] {原则性描述}。{具体场景说明}
```

**示例：**
```markdown
## Playwright
- [260430] RuoYi表格操作必须等待 networkidle + el-table 可见后再交互。直接定位行内按钮容易因异步渲染失败
- [260430] Element Plus 对话框关闭有动画延迟，提交后需等待 .el-message 出现而非立即验证对话框消失
```
