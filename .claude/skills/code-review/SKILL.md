---
name: code-review
description: |
  代码审查 Skill。给测试工程师Agent使用，在开发完成后对代码进行系统性审查。覆盖安全、架构、规范、性能、可维护性五个维度，适配 RuoYi-Vue-Plus 5.6.0 后端和前端项目。当收到代码变更或需要审查代码质量时触发。
---

# 代码审查 Skill

> 适配项目：RuoYi-Vue-Plus 5.6.0
> 后端：Spring Boot 3.5.12 + Java 17 + MyBatis-Plus 3.5.16 + Sa-Token 1.44.0 + Redis + 多租户
> 前端：Vue 3 + Element Plus 2.13.5 + Pinia 3.0.4 + UnoCSS + TypeScript

---

## 一、审查流程

### 1.1 触发条件

| 场景 | 触发方式 |
|------|----------|
| 新功能开发完成 | 开发 Agent 提交代码后自动触发 |
| Bug 修复完成 | 修复提交后自动触发 |
| 代码重构 | 重构提交后自动触发 |
| 定期巡检 | 每周对核心模块进行一次全面审查 |

### 1.2 审查范围

```
变更文件（git diff） ──→ 直接审查
      │
      ▼
影响范围分析 ──→ 关联文件审查
      │
      ▼
公共依赖检查 ──→ ruoyi-common / 公共组件影响评估
```

**影响范围判断规则：**
- 修改 Entity → 审查关联的 Bo、Vo、Mapper、Service、Controller
- 修改 API 接口 → 审查前端对应的 API 层和页面组件
- 修改公共组件 → 审查所有引用该组件的页面
- 修改配置文件 → 审查依赖该配置的所有模块

### 1.3 审查顺序（严格执行）

```
第1轮：安全审查（一票否决）
   ↓ PASS
第2轮：架构审查
   ↓ PASS
第3轮：规范审查
   ↓ PASS
第4轮：性能审查
   ↓ PASS
第5轮：可维护性审查
   ↓
汇总报告
```

> **安全审查发现严重问题 → 直接 FAIL，不再继续后续轮次**

---

## 二、后端代码审查清单（RuoYi 适配）

### 2.1 安全审查

#### 2.1.1 SQL 注入防护

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| MyBatis 参数绑定 | 必须使用 `#{}`，禁止使用 `${}` 拼接用户输入 | 严重 |
| LambdaQueryWrapper | 使用 `LambdaQueryWrapper` 而非字符串字段名 | 严重 |
| 排序字段 | `orderBy` 的字段不能直接接收前端参数，需白名单校验 | 严重 |
| XML 动态 SQL | `${}` 只能用于表名/列名等非用户输入场景 | 严重 |

**示例 - 错误写法：**
```java
// ❌ 直接拼接用户输入
@Select("SELECT * FROM user WHERE name = '${name}'")
// ❌ orderBy 直接使用前端参数
lqw.orderBy(true, isAsc, bo.getOrderByColumn());
```

**示例 - 正确写法：**
```java
// ✅ 使用 #{} 参数绑定
@Select("SELECT * FROM user WHERE name = #{name}")
// ✅ orderBy 白名单校验
String[] allowedColumns = {"name", "create_time", "status"};
if (Arrays.asList(allowedColumns).contains(bo.getOrderByColumn())) {
    lqw.orderBy(true, isAsc, /* 安全的列引用 */);
}
```

#### 2.1.2 XSS 防护

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| @Xss 注解 | 接收用户文本输入的 Bo 字段应加 `@Xss` | 中等 |
| HtmlUtil 清洗 | 富文本内容需用 `HtmlUtil.cleanHtml()` 过滤 | 严重 |
| 输出编码 | 返回给前端的数据不含未转义的 HTML 标签 | 中等 |

**示例 - 正确写法：**
```java
@Xss(message = "名称包含非法字符")
@NotBlank(message = "名称不能为空")
private String name;

// 富文本字段清洗
entity.setContent(HtmlUtil.cleanHtml(bo.getContent()));
```

#### 2.1.3 权限校验

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| @SaCheckPermission | 每个 Controller 方法必须有权限注解 | 严重 |
| 权限字符串格式 | `{module}:{feature}:{action}` 格式一致 | 中等 |
| 匿名接口 | `@SaIgnore` 仅用于登录等公开接口 | 严重 |
| 数据权限 | 涉及数据隔离的查询需检查数据权限配置 | 严重 |

**检查方法：**
```java
// ✅ 正确：每个接口都有权限注解
@SaCheckPermission("system:user:query")
@GetMapping("/list")
public TableDataInfo<SysUserVo> list(...) { }

// ❌ 错误：缺少权限注解
@GetMapping("/list")  // 任何人都能访问！
public TableDataInfo<SysUserVo> list(...) { }
```

#### 2.1.4 敏感数据脱敏

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| @Sensitive 注解 | 手机号、身份证、邮箱等字段需脱敏 | 中等 |
| 日志输出 | 日志中不打印密码、Token、密钥等敏感信息 | 严重 |
| 响应字段 | Vo 中不返回密码原文、盐值等 | 严重 |

**示例 - 正确写法：**
```java
@Sensitive(strategy = SensitiveStrategy.PHONE)
private String phone;

@Sensitive(strategy = SensitiveStrategy.ID_CARD)
private String idCard;
```

#### 2.1.5 越权访问

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 租户隔离 | Entity 继承 `TenantEntity` 时，MyBatis-Plus 自动注入租户条件 | 严重 |
| 数据权限 | `@DataScope` 注解正确配置部门/角色数据权限 | 严重 |
| ID 校验 | 修改/删除操作需校验记录是否属于当前用户/租户 | 严重 |
| 水平越权 | 根据用户 ID 查询时，不能让用户 A 查到用户 B 的数据 | 严重 |

**检查方法：**
```java
// ✅ 正确：修改前校验数据归属
public Boolean updateByBo(FeatureBo bo) {
    Feature entity = featureMapper.selectById(bo.getId());
    if (entity == null) {
        throw new ServiceException("记录不存在");
    }
    // MyBatis-Plus 租户插件已自动过滤，查不到其他租户数据
    // ...
}

// ❌ 错误：直接根据前端传的 ID 操作，无归属校验
public Boolean updateByBo(FeatureBo bo) {
    return featureMapper.updateById(convert(bo)) > 0;
}
```

#### 2.1.6 接口限流

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| @RateLimiter | 登录、短信、导出等接口需加限流 | 中等 |
| 限流阈值 | 根据业务场景设置合理阈值（默认 10次/分钟） | 轻微 |

**示例 - 正确写法：**
```java
@RateLimiter(count = 10, time = 60, message = "操作过于频繁，请稍后再试")
@PostMapping("/login")
public R<LoginVo> login(@RequestBody LoginBo bo) { }
```

### 2.2 架构审查

#### 2.2.1 分层规范

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| Controller 层 | 只做参数接收和响应封装，不含业务逻辑 | 中等 |
| Service 层 | 业务逻辑在此层，不直接操作 HttpServletRequest | 中等 |
| Mapper 层 | 只做数据访问，不含业务判断 | 中等 |
| 跨层调用 | Controller 不得直接调用 Mapper | 严重 |

#### 2.2.2 对象分层

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| Entity | 仅用于数据库映射，不暴露给 Controller | 中等 |
| Bo | 入参对象，含校验注解 | 中等 |
| Vo | 出参对象，不含敏感字段 | 中等 |
| 混用检查 | Controller 参数不得直接使用 Entity | 中等 |

**检查方法：**
```java
// ✅ 正确：Controller 使用 Bo 入参，返回 Vo
@GetMapping("/{id}")
public R<FeatureDetailVo> detail(@PathVariable Long id) { }

// ❌ 错误：Controller 直接使用 Entity
@GetMapping("/{id}")
public R<Feature> detail(@PathVariable Long id) { }
```

#### 2.2.3 依赖关系

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 循环依赖 | Service A 注入 Service B，Service B 又注入 Service A → 需重构 | 严重 |
| 模块依赖 | 业务模块不应依赖 ruoyi-admin | 中等 |
| 工具类复用 | 优先使用 `ruoyi-common` 中的工具类，不重复造轮子 | 轻微 |

#### 2.2.4 工具类复用检查

RuoYi 提供的常用工具类（应复用而非自己实现）：

| 工具类 | 路径 | 用途 |
|--------|------|------|
| `StringUtils` | `ruoyi-common-core` | 字符串判空、格式化 |
| `StreamUtils` | `ruoyi-common-core` | 集合流操作 |
| `BeanUtil` | `ruoyi-common-core` | Bean 拷贝 |
| `ServiceException` | `ruoyi-common-core` | 业务异常 |
| `MapstructUtils` | `ruoyi-common-core` | 对象转换 |
| `RedisUtils` | `ruoyi-common-redis` | Redis 操作 |
| `ExcelUtil` | `ruoyi-common-excel` | Excel 导入导出 |
| `LoginHelper` | `ruoyi-common-satoken` | 获取当前登录用户 |

### 2.3 规范审查

#### 2.3.1 响应体规范

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| R<T> 统一响应 | 所有接口返回 `R<T>` 或 `TableDataInfo<T>` | 严重 |
| 禁止裸返回 | 不直接返回 String、Map、Entity 等裸对象 | 中等 |
| toAjax 方法 | 写操作使用 `toAjax()` 封装返回 | 轻微 |

**检查方法：**
```java
// ✅ 正确
public R<FeatureDetailVo> detail(...) { return R.ok(vo); }
public R<Void> add(...) { return toAjax(service.insertByBo(bo)); }
public TableDataInfo<FeatureVo> list(...) { return service.selectPageList(bo, pageQuery); }

// ❌ 错误
public Feature detail(...) { return entity; }
public Map<String, Object> list(...) { return map; }
```

#### 2.3.2 操作日志

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| @Log 注解 | 所有写操作（增/删/改/导出）必须有 `@Log` | 中等 |
| title 属性 | `title` 值与功能名称一致 | 轻微 |
| businessType | INSERT / UPDATE / DELETE / EXPORT / IMPORT 正确区分 | 轻微 |

**检查方法：**
```java
// ✅ 正确
@Log(title = "用户管理", businessType = BusinessType.INSERT)
@PostMapping
public R<Void> add(...) { }

// ❌ 错误：写操作缺少 @Log
@PostMapping
public R<Void> add(...) { }
```

#### 2.3.3 参数校验

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| @Validated | Controller 类上加 `@Validated` | 中等 |
| Bo 校验注解 | 必填字段 `@NotBlank`/`@NotNull`，长度 `@Size` | 中等 |
| @PathVariable 校验 | 路径参数加 `@NotNull`/`@NotEmpty` | 轻微 |
| 分组校验 | 新增和修改的校验规则不同时使用分组校验 | 轻微 |

#### 2.3.4 异常处理

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 不吞异常 | `try-catch` 中不能为空或只打印日志不处理 | 严重 |
| 业务异常 | 使用 `ServiceException` 抛出业务错误 | 中等 |
| 异常日志 | catch 中用 `log.error("描述", e)` 记录完整堆栈 | 中等 |
| 全局异常 | 不在 Controller 中 try-catch（由全局异常处理器统一处理） | 轻微 |

**示例 - 错误写法：**
```java
// ❌ 吞异常
try {
    doSomething();
} catch (Exception e) {
    // 什么都不做
}

// ❌ 只打印不处理
try {
    doSomething();
} catch (Exception e) {
    e.printStackTrace();
}

// ✅ 正确处理
try {
    doSomething();
} catch (BusinessException e) {
    log.warn("业务异常: {}", e.getMessage());
    throw e; // 或 return R.fail(e.getMessage());
} catch (Exception e) {
    log.error("系统异常", e);
    throw new ServiceException("系统繁忙，请稍后再试");
}
```

#### 2.3.5 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 类名 | PascalCase | `SysUserServiceImpl` |
| 方法名 | camelCase，动词开头 | `selectPageList`、`insertByBo` |
| 变量名 | camelCase | `userName`、`pageSize` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| 包名 | 全小写 | `org.dromara.module` |

**Controller 方法命名约定：**
| 操作 | 方法名 | HTTP 方法 |
|------|--------|-----------|
| 列表查询 | `list` | GET |
| 详情查询 | `detail` 或 `getInfo` | GET |
| 新增 | `add` | POST |
| 修改 | `edit` | PUT |
| 删除 | `delete` 或 `remove` | DELETE |
| 导出 | `export` | POST |

#### 2.3.6 注释规范

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 类注释 | 每个 public 类有 Javadoc 说明用途 | 轻微 |
| 方法注释 | Controller 方法有 `@param`、`@return` | 轻微 |
| 关键逻辑 | 复杂算法、特殊处理有行内注释 | 轻微 |
| TODO 标记 | 临时方案标记 `// TODO [作者] 说明` | 轻微 |

### 2.4 性能审查

#### 2.4.1 N+1 查询

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 循环查数据库 | `for` 循环中调用 Mapper 方法 → 批量查询替代 | 严重 |
| 关联查询 | 多次单条查询 → JOIN 或批量 `IN` 查询 | 中等 |

**示例 - 错误写法：**
```java
// ❌ N+1 查询
List<FeatureVo> list = featureMapper.selectVoList();
for (FeatureVo vo : list) {
    // 每条记录查一次数据库！
    UserVo user = userMapper.selectVoById(vo.getCreateBy());
    vo.setCreateByName(user.getName());
}

// ✅ 批量查询
List<FeatureVo> list = featureMapper.selectVoList();
Set<Long> userIds = list.stream().map(FeatureVo::getCreateBy).collect(Collectors.toSet());
Map<Long, UserVo> userMap = userMapper.selectVoBatchIds(userIds)
    .stream().collect(Collectors.toMap(UserVo::getId, v -> v));
list.forEach(vo -> vo.setCreateByName(userMap.get(vo.getCreateBy())?.getName()));
```

#### 2.4.2 大事务

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| @Transactional 范围 | 事务方法中不调用外部 HTTP、不操作文件、不发消息 | 严重 |
| 事务传播 | 只读方法用 `@Transactional(readOnly = true)` | 轻微 |
| 事务粒度 | 事务尽量小，只包裹必须原子性的操作 | 中等 |

**示例 - 错误写法：**
```java
// ❌ 大事务：事务中包含外部调用
@Transactional
public void processOrder(OrderBo bo) {
    orderMapper.insert(entity);
    // 外部 HTTP 调用可能超时，导致事务长时间挂起
    smsService.sendSms(phone, "订单创建成功");
    emailService.sendEmail(email, "订单创建成功");
}

// ✅ 正确：事务只包裹数据库操作
public void processOrder(OrderBo bo) {
    saveOrder(bo); // 事务方法
    // 事务外执行外部调用
    try {
        smsService.sendSms(phone, "订单创建成功");
    } catch (Exception e) {
        log.warn("短信发送失败，不影响主流程", e);
    }
}

@Transactional
public void saveOrder(OrderBo bo) {
    orderMapper.insert(entity);
}
```

#### 2.4.3 缓存使用

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 缓存穿透 | 查询结果为空时也缓存（短时间） | 中等 |
| 缓存击穿 | 热点 Key 加互斥锁或使用永不过期+异步刷新 | 中等 |
| 缓存一致性 | 更新数据后清除/更新对应缓存 | 严重 |
| 缓存 Key 设计 | 包含租户 ID，避免跨租户数据泄露 | 严重 |

#### 2.4.4 索引使用

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 查询条件 | WHERE 条件字段是否有索引 | 中等 |
| 全表扫描 | 大表查询避免 `SELECT *` 和无 WHERE 条件 | 中等 |
| 模糊查询 | `LIKE '%xxx%'` 无法走索引，考虑全文索引 | 轻微 |
| 联合索引 | 遵循最左前缀原则 | 轻微 |

---

## 三、前端代码审查清单（RuoYi 适配）

### 3.1 安全审查

#### 3.1.1 XSS 防护

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| v-html 使用 | 禁止对用户输入使用 `v-html`，如必须使用需先清洗 | 严重 |
| 用户输入渲染 | 插值表达式 `{{ }}` 已自动转义，确认无绕过 | 中等 |
| URL 跳转 | `window.open()` / `router.push()` 的 URL 需校验协议（防止 `javascript:`） | 中等 |

**示例 - 错误写法：**
```vue
<!-- ❌ 直接渲染用户输入 -->
<div v-html="userComment"></div>

<!-- ✅ 使用 DOMPurify 清洗后渲染 -->
<div v-html="DOMPurify.sanitize(userComment)"></div>
```

#### 3.1.2 API 权限保护

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 接口认证 | 所有 API 请求通过 `request.ts` 统一携带 Token | 中等 |
| 前端权限 | 敏感操作按钮有 `v-hasPermi` 指令控制 | 中等 |
| 接口参数 | 前端校验不能替代后端校验（前端校验可被绕过） | 中等 |

#### 3.1.3 敏感信息

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 密码 | 前端不存储明文密码，登录后只存 Token | 严重 |
| Token 存储 | 使用 `auth.ts` 工具管理，不直接操作 localStorage | 轻微 |
| Source Map | 生产环境不暴露 Source Map | 轻微 |
| 调试信息 | 不在控制台打印敏感数据（Token、用户密码等） | 轻微 |

### 3.2 架构审查

#### 3.2.1 组件拆分

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 单文件大小 | 单个 `.vue` 文件不超过 500 行（不含模板） | 中等 |
| 职责单一 | 每个组件只做一件事 | 轻微 |
| 复用组件 | 超过 2 个页面使用的组件提取到 `components/` | 轻微 |
| 页面私有组件 | 仅一个页面使用的组件放 `views/{module}/{feature}/components/` | 轻微 |

#### 3.2.2 Store 使用

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 全局 vs 局部 | 仅跨页面共享的状态放 Pinia Store，页面内状态用 `ref`/`reactive` | 轻微 |
| Store 命名 | `use{Module}Store`，文件名 `{module}.ts` | 轻微 |
| 修改方式 | 通过 action/函数修改，不直接 `store.xxx = yyy`（Setup Store 除外） | 轻微 |

#### 3.2.3 API 封装

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| API 目录 | 所有 API 调用封装在 `src/api/{module}/{feature}/index.ts` | 中等 |
| 禁止裸调用 | 页面中禁止直接写 `axios.get(...)` 或 `fetch(...)` | 中等 |
| 类型定义 | API 类型定义在 `types.ts` 中 | 轻微 |

### 3.3 规范审查

#### 3.3.1 TypeScript 类型

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| any 滥用 | API 参数和返回值不应为 `any`（表单对象除外） | 中等 |
| ref 类型 | `ref<Type>(initialValue)` 显式声明类型 | 轻微 |
| 接口定义 | 每个模块有独立的 `types.ts` | 轻微 |
| 事件类型 | 事件处理函数参数有类型标注 | 轻微 |

**检查方法：**
```typescript
// ❌ 过多 any
const list = ref<any[]>([]);
const handleSubmit = (data: any) => { };

// ✅ 有类型
const list = ref<FeatureVo[]>([]);
const handleSubmit = (data: FeatureForm) => { };
```

#### 3.3.2 Element Plus 规范

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 表格 border | `<el-table>` 必须有 `border` 属性 | 轻微 |
| 弹窗 append-to-body | `<el-dialog>` 必须有 `append-to-body` | 中等 |
| 表单校验 | `<el-form>` 有 `:rules`，`<el-form-item>` 有 `prop` | 中等 |
| 分页组件 | 使用 `<pagination>` 全局组件 | 轻微 |

#### 3.3.3 UnoCSS 规范

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 禁止内联 style | `style="margin-top: 10px"` → 改用 UnoCSS 类名 `mt-[10px]` | 轻微 |
| 禁止内联 CSS | `<style>` 中能用原子类的样式优先用原子类 | 轻微 |
| 类名拼写 | 使用标准的 Tailwind/UnoCSS 类名 | 轻微 |

#### 3.3.4 权限指令

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 操作按钮 | 增/删/改/导出按钮必须有 `v-hasPermi` | 严重 |
| 权限字符串 | 与后端 `sys_menu.perms` 一致 | 中等 |
| 角色权限 | 管理员专属操作用 `v-hasRole` | 轻微 |

#### 3.3.5 表单校验

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 必填校验 | 必填字段有 `required: true` 规则 | 中等 |
| 格式校验 | 邮箱、手机号、URL 等有格式校验 | 轻微 |
| 异步校验 | 唯一性检查等使用异步校验器 | 轻微 |
| 提交校验 | 提交前调用 `formRef.value?.validate()` | 中等 |

### 3.4 性能审查

#### 3.4.1 大列表渲染

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| 虚拟滚动 | 超过 1000 条数据使用虚拟滚动 | 中等 |
| 分页加载 | 默认使用分页，不一次性加载全部数据 | 中等 |
| v-for key | `v-for` 必须有 `:key`，使用唯一 ID 而非 index | 轻微 |

#### 3.4.2 组件渲染优化

| 检查项 | 审查要点 | 严重度 |
|--------|----------|--------|
| v-if vs v-show | 频繁切换用 `v-show`，条件渲染用 `v-if` | 轻微 |
| 计算属性 | 模板中的复杂计算用 `computed` 而非方法调用 | 轻微 |
| 组件懒加载 | 非首屏组件使用 `defineAsyncComponent` | 轻微 |

**示例 - 错误写法：**
```vue
<!-- ❌ 每次渲染都重新计算 -->
<div>{{ getFilteredList().map(i => i.name).join(', ') }}</div>

<!-- ✅ 使用计算属性缓存结果 -->
<div>{{ filteredNames }}</div>

<script setup>
const filteredNames = computed(() => 
  getFilteredList().map(i => i.name).join(', ')
);
</script>
```

---

## 四、审查报告模板

```markdown
# 代码审查报告

## 审查目标：{模块/功能名称}
## 审查日期：{YYYY-MM-DD}
## 审查范围：{变更文件列表 + 影响范围}
## 审查人：{Agent名称}

## 判定：PASS / FAIL

### 审查摘要

| 轮次 | 结果 | 发现问题数 |
|------|------|-----------|
| 第1轮 安全审查 | PASS/FAIL | {N} |
| 第2轮 架构审查 | PASS/FAIL | {N} |
| 第3轮 规范审查 | PASS/FAIL | {N} |
| 第4轮 性能审查 | PASS/FAIL | {N} |
| 第5轮 可维护性审查 | PASS/FAIL | {N} |

### 问题清单

| # | 严重度 | 类别 | 文件:行号 | 问题描述 | 修改建议 |
|---|--------|------|-----------|----------|----------|
| 1 | 严重 | 安全 | UserController.java:45 | 缺少 @SaCheckPermission 注解 | 添加 @SaCheckPermission("system:user:query") |
| 2 | 中等 | 规范 | UserBo.java:22 | 缺少 @NotBlank 校验注解 | 添加 @NotBlank(message = "用户名不能为空") |
| 3 | 轻微 | 性能 | UserServiceImpl.java:89 | 循环中单条查询数据库 | 改为批量查询，使用 IN 条件 |

### 严重度说明

- **严重**：必须修复后才能合并，可能导致安全漏洞或系统故障
- **中等**：建议修复，影响代码质量或可维护性
- **轻微**：可选优化，不影响功能但可以改进

### 亮点（可选）

- 列出代码中做得好的地方，正面反馈
```

---

## 五、常见问题速查表

### 5.1 后端常见问题

| # | 问题 | 严重度 | 标准修复方式 |
|---|------|--------|-------------|
| 1 | Controller 方法缺少 `@SaCheckPermission` | 严重 | 添加对应权限注解 |
| 2 | 写操作缺少 `@Log` 注解 | 中等 | 添加 `@Log(title="xxx", businessType=BusinessType.XXX)` |
| 3 | 写操作缺少 `@RepeatSubmit` | 中等 | 添加 `@RepeatSubmit` 防重提交 |
| 4 | Controller 直接返回 Entity | 中等 | 改为返回 Vo，使用 `R<T>` 包装 |
| 5 | Service 中 `try-catch` 吞异常 | 严重 | 抛出 `ServiceException` 或记录日志后重新抛出 |
| 6 | 循环内调用 Mapper 查询 | 严重 | 改为批量查询 + Map 映射 |
| 7 | Bo 字段缺少校验注解 | 中等 | 添加 `@NotBlank`/`@NotNull`/`@Size` 等注解 |
| 8 | `@Transactional` 包含外部调用 | 严重 | 缩小事务范围，外部调用放到事务外 |
| 9 | SQL 使用 `${}` 拼接用户输入 | 严重 | 改为 `#{}` 参数绑定 |
| 10 | 没有使用 `LambdaQueryWrapper` | 中等 | 改为 Lambda 方式，避免字符串字段名硬编码 |
| 11 | 缓存更新后未清除旧缓存 | 严重 | 使用 `@CacheEvict` 或手动清除 |
| 12 | 返回 `R.ok(data)` 时 data 为 null | 轻微 | 使用 `R.ok()` 无参形式或确保 data 非空 |

### 5.2 前端常见问题

| # | 问题 | 严重度 | 标准修复方式 |
|---|------|--------|-------------|
| 1 | 操作按钮缺少 `v-hasPermi` | 严重 | 添加 `v-hasPermi="['module:feature:action']"` |
| 2 | `<el-dialog>` 缺少 `append-to-body` | 中等 | 添加 `append-to-body` 属性 |
| 3 | 页面中直接写 `axios.get(...)` | 中等 | 封装到 `src/api/` 目录 |
| 4 | `<el-table>` 缺少 `border` | 轻微 | 添加 `border` 属性 |
| 5 | `<el-form>` 缺少 `:rules` | 中等 | 添加校验规则 |
| 6 | 写操作后未刷新列表 | 中等 | `.then()` 中调用 `getList()` |
| 7 | 搜索后未重置 `pageNum` | 中等 | `handleQuery` 中设置 `queryParams.pageNum = 1` |
| 8 | 使用 `v-html` 渲染用户输入 | 严重 | 使用 `DOMPurify.sanitize()` 清洗 |
| 9 | 内联 `style` 属性 | 轻微 | 改用 UnoCSS 原子类 |
| 10 | `v-for` 使用 index 作为 key | 轻微 | 改为使用唯一 ID |
| 11 | API 返回值类型为 `any` | 轻微 | 定义 TypeScript 接口 |
| 12 | 删除操作未二次确认 | 中等 | 使用 `proxy?.$modal.confirm()` |

### 5.3 快速判定标准

**直接 PASS 条件（全部满足）：**
- 无严重问题
- 中等问题 ≤ 2 个
- 所有问题都有明确的修复方案

**直接 FAIL 条件（满足任一）：**
- 存在严重问题
- 中等问题 > 5 个
- 安全审查未通过

---

## 六、审查执行命令

### 6.1 获取变更文件

```bash
# 查看最近一次提交的变更
cd {BACKEND_DIR}/
git diff --name-only HEAD~1

# 查看未提交的变更
git diff --name-only

# 查看指定分支的差异
git diff --name-only main..feature-branch
```

### 6.2 前端变更

```bash
cd {FRONTEND_DIR}/
git diff --name-only HEAD~1
```

### 6.3 审查工作流

```
1. 获取变更文件列表
2. 逐个文件读取代码
3. 按审查顺序（安全→架构→规范→性能→可维护性）逐项检查
4. 记录问题到问题清单
5. 生成审查报告（PASS/FAIL）
6. 如 FAIL，将问题反馈给开发 Agent 修复
7. 修复后重新审查
```
