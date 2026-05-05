---
name: api-design
description: |
  RESTful API 设计规范 Skill。触发词包括"API设计"、"接口设计"、"接口文档"、"RESTful规范"、"API规范"、"设计接口"。用于指导和校验 RuoYi-Vue-Plus 项目的 API 接口设计，确保符合项目架构规范。
---

# RESTful API 设计规范 Skill

本 Skill 定义了 RuoYi-Vue-Plus 项目的 API 接口设计规范，用于指导接口设计、校验现有接口、生成 API 文档。

## 项目技术栈

- **后端：** Spring Boot 3.5.12 + Java 17 + MyBatis-Plus 3.5.16 + Sa-Token 1.44.0
- **前端：** Vue 3 + Element Plus 2.13.5 + Pinia + UnoCSS + TypeScript
- **后端路径：** `{BACKEND_DIR}/`
- **前端路径：** `{FRONTEND_DIR}/`

---

## 一、URL 规范

### 基础路径

```
/api/{module}/{feature}
```

- `module`：业务模块名（如 `system`、`monitor`、`tool`）
- `feature`：功能资源名（如 `user`、`role`、`menu`、`dict`）

### URL 命名规则

- 全部小写，单词间用 `-` 连接
- 资源用名词复数或集合语义
- 不包含动词，HTTP 方法本身表达操作

### HTTP 方法与操作映射

| HTTP 方法 | 操作 | URL 示例 | 说明 |
|-----------|------|----------|------|
| **GET** | 查询（列表） | `GET /api/system/user/list` | 分页查询列表 |
| **GET** | 查询（详情） | `GET /api/system/user/{id}` | 根据 ID 查询单条 |
| **POST** | 新增 | `POST /api/system/user` | 新增一条记录 |
| **PUT** | 修改 | `PUT /api/system/user` | 修改一条记录 |
| **DELETE** | 删除 | `DELETE /api/system/user/{ids}` | 根据 ID 删除 |

### 特殊操作后缀

| 后缀 | HTTP 方法 | URL 示例 | 说明 |
|------|-----------|----------|------|
| `/list` | GET | `GET /api/system/user/list` | 分页列表查询 |
| `/batch` | POST | `POST /api/system/user/batch` | 批量新增 |
| `/export` | POST | `POST /api/system/user/export` | 数据导出 |
| `/import` | POST | `POST /api/system/user/import` | 数据导入 |
| `/changeStatus` | PUT | `PUT /api/system/user/changeStatus` | 状态变更 |

### URL 设计示例

```
# 用户管理
GET    /api/system/user/list              # 分页查询用户列表
GET    /api/system/user/{userId}          # 查询用户详情
POST   /api/system/user                   # 新增用户
PUT    /api/system/user                   # 修改用户
DELETE /api/system/user/{userIds}         # 删除用户
PUT    /api/system/user/changeStatus      # 修改用户状态
PUT    /api/system/user/resetPwd          # 重置密码
POST   /api/system/user/export            # 导出用户
POST   /api/system/user/import            # 导入用户

# 角色管理
GET    /api/system/role/list              # 分页查询角色列表
GET    /api/system/role/{roleId}          # 查询角色详情
POST   /api/system/role                   # 新增角色
PUT    /api/system/role                   # 修改角色
DELETE /api/system/role/{roleIds}         # 删除角色
PUT    /api/system/role/changeStatus      # 修改角色状态
PUT    /api/system/role/dataScope         # 修改数据权限
GET    /api/system/role/optionSelect      # 角色下拉选择

# 字典管理
GET    /api/system/dict/type/list         # 字典类型列表
GET    /api/system/dict/type/{dictId}     # 字典类型详情
POST   /api/system/dict/type              # 新增字典类型
PUT    /api/system/dict/type              # 修改字典类型
DELETE /api/system/dict/type/{dictIds}    # 删除字典类型
GET    /api/system/dict/data/type/{dictType}  # 根据类型查询字典数据
```

---

## 二、请求 / 响应规范

### 统一响应体 `R<T>`

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": { ... }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `code` | int | 状态码，200 成功，其他为错误码 |
| `msg` | String | 提示信息 |
| `data` | T | 业务数据，可为 null |

### 分页响应 `TableDataInfo<T>`

```json
{
  "code": 200,
  "msg": "查询成功",
  "rows": [ ... ],
  "total": 100
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `rows` | List\<T\> | 当前页数据列表 |
| `total` | long | 总记录数 |

### 请求参数规范

**查询参数（GET 请求）：** 使用 Bo（Business Object）对象接收，通过 `@Validated` 注解校验。

```java
@GetMapping("/list")
public TableDataInfo<UserVo> list(UserBo bo, PageQuery pageQuery) {
    return userService.selectPageUserList(bo, pageQuery);
}
```

**请求体（POST/PUT 请求）：** 使用 Bo 对象，JSON 格式提交。

```java
@PostMapping
public R<Void> add(@Validated @RequestBody UserBo bo) {
    userService.insertUser(bo);
    return R.ok();
}
```

### Bo 对象规范

- 使用 `@Validated` 注解启用校验
- 使用 JSR 303 注解进行字段校验（`@NotBlank`、`@NotNull`、`@Size`、`@Email` 等）
- Bo 对象只包含入参，不包含业务逻辑

```java
public class UserBo extends BaseEntity {
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 20, message = "用户名长度为 2-20 个字符")
    private String userName;

    @NotBlank(message = "密码不能为空")
    private String password;

    @Email(message = "邮箱格式不正确")
    private String email;

    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String phonenumber;
}
```

### Vo 对象规范

- Vo（View Object）用于响应，只返回前端需要的字段
- 敏感字段（密码等）不应出现在 Vo 中
- 日期字段使用 `@JsonFormat` 指定格式

```java
public class UserVo {
    private Long userId;
    private String userName;
    private String nickName;
    private String email;
    private String phonenumber;
    private String status;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private Date createTime;
}
```

---

## 三、权限标识符规范

### 格式

```
{module}:{feature}:{action}
```

### action 取值

| action | 含义 | 对应操作 |
|--------|------|----------|
| `query` | 查询 | 列表查询、详情查询 |
| `add` | 新增 | 新增记录 |
| `edit` | 修改 | 修改记录 |
| `remove` | 删除 | 删除记录 |
| `export` | 导出 | 数据导出 |
| `import` | 导入 | 数据导入 |

### 权限标识符示例

```
system:user:query      # 用户查询
system:user:add        # 用户新增
system:user:edit       # 用户修改
system:user:remove     # 用户删除
system:user:export     # 用户导出
system:user:import     # 用户导入
system:role:query      # 角色查询
system:role:add        # 角色新增
system:role:edit       # 角色修改
system:role:remove     # 角色删除
system:menu:query      # 菜单查询
system:menu:add        # 菜单新增
system:menu:edit       # 菜单修改
system:menu:remove     # 菜单删除
```

### 后端权限注解

```java
@SaCheckPermission("system:user:list")
@GetMapping("/list")
public TableDataInfo<UserVo> list(UserBo bo, PageQuery pageQuery) { ... }

@SaCheckPermission("system:user:query")
@GetMapping("/{userId}")
public R<UserVo> getInfo(@PathVariable Long userId) { ... }

@SaCheckPermission("system:user:add")
@PostMapping
public R<Void> add(@Validated @RequestBody UserBo bo) { ... }

@SaCheckPermission("system:user:edit")
@PutMapping
public R<Void> edit(@Validated @RequestBody UserBo bo) { ... }

@SaCheckPermission("system:user:remove")
@DeleteMapping("/{userIds}")
public R<Void> remove(@PathVariable Long[] userIds) { ... }
```

### 前端权限指令

```typescript
// 按钮级别权限控制
<el-button v-hasPermi="['system:user:add']">新增</el-button>
<el-button v-hasPermi="['system:user:edit']">修改</el-button>
<el-button v-hasPermi="['system:user:remove']">删除</el-button>
<el-button v-hasPermi="['system:user:export']">导出</el-button>
```

---

## 四、分页参数规范

### 请求参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `pageNum` | int | 否 | 1 | 当前页码 |
| `pageSize` | int | 否 | 10 | 每页条数 |

### 前端调用示例

```typescript
// 使用 PageQuery
const params = {
  pageNum: 1,
  pageSize: 10,
  userName: '张三'  // 其他查询条件
}

const res = await listUser(params)
```

### 后端接收示例

```java
@GetMapping("/list")
public TableDataInfo<UserVo> list(UserBo bo, PageQuery pageQuery) {
    return userService.selectPageUserList(bo, pageQuery);
}
```

---

## 五、API 文档模板

每个功能模块的接口设计按以下模板输出：

```markdown
## {Feature} 管理

### 列表查询

- **URL**: `GET /api/{module}/{feature}/list`
- **权限**: `{module}:{feature}:query`
- **描述**: 分页查询{Feature}列表
- **请求参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| pageNum | int | 否 | 页码，默认 1 |
| pageSize | int | 否 | 每页条数，默认 10 |
| {field1} | String | 否 | {说明} |
| {field2} | int | 否 | {说明} |

- **响应**: `R<TableDataInfo<{Vo}>>`
- **响应示例**:

```json
{
  "code": 200,
  "msg": "查询成功",
  "rows": [
    {
      "id": 1,
      "name": "示例",
      "status": "0",
      "createTime": "2025-01-01 00:00:00"
    }
  ],
  "total": 50
}
```

### 详情查询

- **URL**: `GET /api/{module}/{feature}/{id}`
- **权限**: `{module}:{feature}:query`
- **描述**: 根据 ID 查询{Feature}详情
- **路径参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| {id} | long | 是 | 记录 ID |

- **响应**: `R<{Vo}>`

### 新增

- **URL**: `POST /api/{module}/{feature}`
- **权限**: `{module}:{feature}:add`
- **描述**: 新增一条{Feature}记录
- **请求体** ({Bo}):

| 参数名 | 类型 | 必填 | 校验规则 | 说明 |
|--------|------|------|----------|------|
| {field1} | String | 是 | @NotBlank, @Size(2,20) | {说明} |
| {field2} | int | 否 | - | {说明} |

- **响应**: `R<Void>`

### 修改

- **URL**: `PUT /api/{module}/{feature}`
- **权限**: `{module}:{feature}:edit`
- **描述**: 修改{Feature}记录
- **请求体** ({Bo}): 同新增（包含 ID）
- **响应**: `R<Void>`

### 删除

- **URL**: `DELETE /api/{module}/{feature}/{ids}`
- **权限**: `{module}:{feature}:remove`
- **描述**: 根据 ID 删除{Feature}记录（支持批量）
- **路径参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| ids | String | 是 | 记录 ID，多个用逗号分隔 |

- **响应**: `R<Void>`
```

---

## 六、接口设计检查清单

设计接口时，逐项检查以下要点：

- [ ] **URL 符合 RESTful 规范**：GET 查询、POST 新增、PUT 修改、DELETE 删除
- [ ] **URL 命名规范**：全小写，单词用 `-` 连接，不含动词
- [ ] **权限标识符格式正确**：`{module}:{feature}:{action}`
- [ ] **后端使用 `@SaCheckPermission` 注解**：与权限标识符对应
- [ ] **请求参数使用 Bo 对象**：配合 `@Validated` 注解校验
- [ ] **响应使用 Vo 对象**：不直接返回实体类
- [ ] **统一响应体**：使用 `R<T>` 包装
- [ ] **列表接口支持分页**：接收 `PageQuery`，返回 `TableDataInfo<T>`
- [ ] **导出接口有权限控制**：`{module}:{feature}:export`
- [ ] **导入接口有权限控制**：`{module}:{feature}:import`
- [ ] **操作日志记录**：关键操作使用 `@Log` 注解记录
- [ ] **字段命名统一**：前后端均使用驼峰命名
- [ ] **日期格式统一**：`yyyy-MM-dd HH:mm:ss`

---

## 七、常见 API 设计模式

### 7.1 树形结构接口

适用于菜单、部门、分类等树形数据：

```markdown
### 树形列表查询
- **URL**: `GET /api/{module}/{feature}/treeselect`
- **权限**: `{module}:{feature}:query`
- **响应**: `R<List<{Vo}>>`（Vo 中包含 `children` 字段）

### 构建下拉树
- **URL**: `GET /api/{module}/{feature}/treeselect`
- **描述**: 获取树形下拉选择数据
```

```java
// 后端示例
@GetMapping("/treeselect")
public R<List<TreeSelectVo>> treeselect() {
    return R.ok(menuService.selectMenuTreeList());
}
```

```typescript
// 前端示例：el-tree-select 组件
<el-tree-select
  v-model="parentId"
  :data="menuOptions"
  :props="{ value: 'id', label: 'label', children: 'children' }"
/>
```

### 7.2 字典数据接口

适用于下拉选择、状态标签等：

```markdown
### 根据字典类型查询
- **URL**: `GET /api/system/dict/data/type/{dictType}`
- **权限**: 无（字典数据通常为公共接口）
- **响应**: `R<List<DictDataVo>>`

### DictDataVo 结构：
| 字段 | 类型 | 说明 |
|------|------|------|
| dictValue | String | 字典值 |
| dictLabel | String | 字典标签 |
| dictSort | int | 排序 |
| listClass | String | 样式类型（primary/success/warning/danger） |
```

```typescript
// 前端使用字典
const { proxy } = getCurrentInstance()
const dictOptions = ref<any[]>([])

// 获取字典数据
proxy.getDicts("sys_normal_disable").then(res => {
  dictOptions.value = res.data
})

// 在模板中使用字典标签
<dict-tag :options="dictOptions" :value="row.status" />
```

### 7.3 文件上传下载接口

```markdown
### 文件上传
- **URL**: `POST /api/resource/oss/upload`
- **权限**: 需要登录
- **参数**: multipart/form-data，字段名 `file`
- **响应**: `R<OssVo>`（包含文件 URL）

### 文件下载
- **URL**: `GET /api/resource/oss/download/{ossId}`
- **权限**: 需要登录
- **响应**: 文件流（application/octet-stream）
```

### 7.4 下拉选择接口

用于前端 `el-select` 组件的数据源：

```markdown
### 下拉选择列表
- **URL**: `GET /api/{module}/{feature}/optionSelect`
- **权限**: `{module}:{feature}:query`
- **描述**: 获取下拉选择数据（不分页，全量返回）
- **响应**: `R<List<OptionVo>>`

### OptionVo 结构：
| 字段 | 类型 | 说明 |
|------|------|------|
| value | String/Long | 选项值 |
| label | String | 选项标签 |
```

### 7.5 数据权限接口

用于控制不同角色只能看到自己权限范围内的数据：

```java
// 后端：在 Service 层通过数据权限注解或 AOP 处理
@DataScope(deptAlias = "d", userAlias = "u")
public List<UserVo> selectUserList(UserBo bo) { ... }

// 或通过 MyBatis-Plus 拦截器实现数据范围过滤
```

---

## 八、错误码规范

### 标准错误码

| 错误码 | 含义 | 说明 |
|--------|------|------|
| 200 | 成功 | 操作成功 |
| 400 | 请求错误 | 参数校验失败等 |
| 401 | 未认证 | Token 过期或无效 |
| 403 | 无权限 | 没有访问权限 |
| 404 | 未找到 | 资源不存在 |
| 500 | 服务器错误 | 系统内部异常 |

### 业务错误码

业务模块可自定义错误码，建议使用 6 位数字，前 2 位为模块编码：

```
100001 - 用户模块：用户名已存在
100002 - 用户模块：手机号已注册
200001 - 角色模块：角色已分配用户，不能删除
200002 - 角色模块：角色编码已存在
```

---

## 九、操作日志规范

关键操作（新增、修改、删除、导出、导入等）必须记录操作日志：

```java
@Log(title = "用户管理", businessType = BusinessType.INSERT)
@SaCheckPermission("system:user:add")
@PostMapping
public R<Void> add(@Validated @RequestBody UserBo bo) { ... }

@Log(title = "用户管理", businessType = BusinessType.UPDATE)
@SaCheckPermission("system:user:edit")
@PutMapping
public R<Void> edit(@Validated @RequestBody UserBo bo) { ... }

@Log(title = "用户管理", businessType = BusinessType.DELETE)
@SaCheckPermission("system:user:remove")
@DeleteMapping("/{userIds}")
public R<Void> remove(@PathVariable Long[] userIds) { ... }

@Log(title = "用户管理", businessType = BusinessType.EXPORT)
@SaCheckPermission("system:user:export")
@PostMapping("/export")
public void export(UserBo bo, HttpServletResponse response) { ... }
```

### `BusinessType` 取值

| 值 | 含义 |
|----|------|
| `OTHER` | 其他 |
| `INSERT` | 新增 |
| `UPDATE` | 修改 |
| `DELETE` | 删除 |
| `EXPORT` | 导出 |
| `IMPORT` | 导入 |
| `GRANT` | 授权 |
| `CLEAN` | 清空 |
| `GENCODE` | 生成代码 |
