---
name: integration-testing
description: 集成测试 — Spring Boot Test + API联调测试
---

# 集成测试

> 适配项目：RuoYi-Vue-Plus 5.6.0
> 后端：Spring Boot 3.5.12 + Java 17 + MyBatis-Plus + Sa-Token

---

## 一、概述

集成测试（Layer 2）验证模块间协作是否正确，覆盖 Service 层与数据库的真实交互、API 接口联调等场景。

> **依赖关系：** 集成测试在单元测试（Layer 1）之上，需要 Spring 上下文启动和数据库连接。
> **相关 Skill：**
> - L1 单元测试：`junit-testing`、`vitest-testing`
> - L3 UI自动化测试：`ui-testing`
> - L4 E2E端到端测试：`e2e-testing`

---

## 二、后端 Spring Boot Test

### 2.1 执行命令

```bash
cd {BACKEND_DIR}/

# 运行集成测试（需启动Spring上下文）
mvn test -pl ruoyi-modules/ruoyi-{module} -Dtest="*IntegrationTest" -am

# 只运行集成测试
mvn test -Dtest="*IntegrationTest"
```

### 2.2 集成测试模板

```java
package org.dromara.{module}.service;

import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import org.dromara.{module}.domain.bo.{Feature}Bo;
import org.dromara.{module}.domain.vo.{Feature}Vo;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles("dev")
@DisplayName("{Feature}模块集成测试")
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class {Feature}ServiceIntegrationTest {

    @Autowired
    private I{Feature}Service {feature}Service;

    private static Long createdId;

    @Test
    @Order(1)
    @DisplayName("集成 - 新增记录")
    void insertByBo_shouldCreateRecord() {
        {Feature}Bo bo = new {Feature}Bo();
        bo.setName("集成测试名称");
        Boolean result = {feature}Service.insertByBo(bo);
        assertTrue(result);
    }

    @Test
    @Order(2)
    @DisplayName("集成 - 查询列表")
    void selectPage_shouldReturnResults() {
        {Feature}Vo queryVo = new {Feature}Vo();
        queryVo.setName("集成测试");
        Page<{Feature}Vo> page = {feature}Service.selectPage(queryVo);
        assertNotNull(page);
        assertTrue(page.getRecords().size() > 0);
    }

    @Test
    @Order(3)
    @DisplayName("集成 - 查询详情")
    void selectDetailById_shouldReturnDetail() {
        {Feature}Vo result = {feature}Service.selectDetailById(createdId);
        assertNotNull(result);
        assertEquals("集成测试名称", result.getName());
    }

    @Test
    @Order(4)
    @DisplayName("集成 - 更新记录")
    void updateByBo_shouldUpdateRecord() {
        {Feature}Bo bo = new {Feature}Bo();
        bo.setId(createdId);
        bo.setName("更新后名称");
        Boolean result = {feature}Service.updateByBo(bo);
        assertTrue(result);
    }

    @Test
    @Order(5)
    @DisplayName("集成 - 删除记录")
    void deleteWithValidByIds_shouldDeleteRecord() {
        Boolean result = {feature}Service.deleteWithValidByIds(new Long[]{createdId});
        assertTrue(result);
    }
}
```

### 2.3 测试目录结构

```
ruoyi-modules/ruoyi-{module}/src/test/java/org/dromara/{module}/
├── service/
│   └── {Feature}ServiceIntegrationTest.java   # Service集成测试
├── mapper/
│   └── {Feature}MapperTest.java               # Mapper测试（可选）
└── controller/
    └── {Feature}ControllerTest.java           # Controller测试（可选）
```

---

## 三、API联调测试

### 3.1 测试模板

```typescript
// e2e/api/{module}.api.test.ts
import { test, expect } from '@playwright/test';

const BASE_URL = 'http://localhost:8080';

test.describe('{Module} API联调测试', () => {
  let token = '';
  let headers: Record<string, string>;

  test.beforeAll(async ({ request }) => {
    // 获取登录Token
    const loginRes = await request.post(`${BASE_URL}/auth/login`, {
      data: {
        tenantId: '000000',
        username: 'admin',
        password: 'admin123',
      },
    });
    const loginData = await loginRes.json();
    token = loginData.data.access_token;
    headers = {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
      'clientid': 'e5cd7e4891bf95d1d19206ce24a7b32e',
    };
  });

  test('GET /api/{module}/{feature}/list - 查询列表', async ({ request }) => {
    const res = await request.get(`${BASE_URL}/api/{module}/{feature}/list`, {
      headers,
      params: { pageNum: 1, pageSize: 10 },
    });
    expect(res.status()).toBe(200);
    const data = await res.json();
    expect(data.code).toBe(200);
    expect(data.data).toHaveProperty('rows');
    expect(data.data).toHaveProperty('total');
  });

  test('POST /api/{module}/{feature} - 新增记录', async ({ request }) => {
    const res = await request.post(`${BASE_URL}/api/{module}/{feature}`, {
      headers,
      data: { name: 'API测试数据' },
    });
    expect(res.status()).toBe(200);
    const data = await res.json();
    expect(data.code).toBe(200);
  });

  test('PUT /api/{module}/{feature} - 更新记录', async ({ request }) => {
    const res = await request.put(`${BASE_URL}/api/{module}/{feature}`, {
      headers,
      data: { id: 1, name: '更新后数据' },
    });
    expect(res.status()).toBe(200);
    const data = await res.json();
    expect(data.code).toBe(200);
  });

  test('DELETE /api/{module}/{feature}/1 - 删除记录', async ({ request }) => {
    const res = await request.delete(`${BASE_URL}/api/{module}/{feature}/1`, {
      headers,
    });
    expect(res.status()).toBe(200);
    const data = await res.json();
    expect(data.code).toBe(200);
  });
});
```

### 3.2 执行命令

```bash
cd {E2E_DIR}/
npx playwright test tests/api/{module}.api.test.ts
```

---

## 四、注意事项

- 集成测试需要真实的数据库和 Redis 环境，确保开发环境已启动
- `@ActiveProfiles("dev")` 指定使用开发配置
- 测试数据会在测试完成后残留，建议在测试中自行清理或使用事务回滚
- API联调测试依赖后端服务运行，确保 `{BACKEND_DIR}` 已启动
