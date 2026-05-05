---
name: db-design
description: |
  数据库设计规范 Skill。给后端工程师 Agent 使用，在 RuoYi-Vue-Plus 框架下设计数据库表结构时触发。覆盖命名规范、建表模板、数据类型选择、索引设计、多租户设计、MyBatis-Plus 映射、ER 图描述等内容。
---

# 数据库设计规范 Skill

> 版本：v1.0 | 适配 RuoYi-Vue-Plus 5.6.0
> 数据库：MySQL 8.0+ | ORM：MyBatis-Plus 3.5.16

---

## 一、命名规范

### 1.1 表名

| 规则 | 说明 | 示例 |
|------|------|------|
| 全小写 | 表名统一小写 | `sys_user`、`biz_order` |
| 下划线分隔 | 多词用 `_` 连接 | `sys_user_role` |
| 前缀分类 | `sys_` 系统表、`biz_` 业务表 | `sys_config`、`biz_order_item` |
| 见名知意 | 表名能表达业务含义 | `inspection_task`（非 `task`） |

### 1.2 字段名

| 规则 | 说明 | 示例 |
|------|------|------|
| 全小写下划线 | 与表名风格一致 | `create_time`、`user_name` |
| 布尔字段 | `is_` 前缀或 `_flag` 后缀 | `is_enabled`、`del_flag` |
| 时间字段 | `_time` 后缀 | `create_time`、`expire_time` |
| 日期字段 | `_date` 后缀 | `birth_date` |
| 数量字段 | `_count` / `_num` 后缀 | `view_count`、`retry_num` |
| 金额字段 | `_amount` / `_price` 后缀 | `total_amount`、`unit_price` |
| 状态字段 | `status` 或 `_status` | `status`、`order_status` |
| 类型字段 | `type` 或 `_type` | `task_type`、`notify_type` |

### 1.3 索引名

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| 普通索引 | `idx_{表名}_{字段名}` | `idx_order_create_time` |
| 唯一索引 | `uk_{表名}_{字段名}` | `uk_user_phone` |
| 联合索引 | `idx_{表名}_{字段1}_{字段2}` | `idx_order_user_status` |

> **注意：** 索引名需在 64 字符以内（MySQL 限制）。

---

## 二、通用字段（RuoYi 标准）

### 2.1 审计字段（所有业务表必须包含）

| 字段名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `create_dept` | bigint | NULL | 创建部门 |
| `create_by` | bigint | NULL | 创建者（用户ID） |
| `create_time` | datetime | NULL | 创建时间 |
| `update_by` | bigint | NULL | 更新者 |
| `update_time` | datetime | NULL | 更新时间 |
| `del_flag` | char(1) | '0' | 删除标志（0正常 2删除） |
| `tenant_id` | varchar(20) | '000000' | 租户编号（多租户表必须） |

### 2.2 审计字段说明

- **create_dept**：创建人所属部门，用于数据权限过滤
- **del_flag**：RuoYi 使用 `0/2` 而非 `0/1`（`2` 代表删除）
- **tenant_id**：默认值 `000000` 为超级管理员租户

### 2.3 不需要租户隔离的表

以下表可以不加 `tenant_id`：
- 系统级配置表（如 `sys_config`、`sys_dict_type`）
- 租户管理表本身（如 `sys_tenant`）
- 系统内置数据（如 `sys_menu`、`sys_dept`）

---

## 三、建表 SQL 模板

### 3.1 多租户业务表（最常用）

```sql
-- ----------------------------
-- {表中文说明}
-- ----------------------------
CREATE TABLE IF NOT EXISTS {table_name} (
    -- 主键
    id              BIGINT       NOT NULL COMMENT '主键ID',

    -- 业务字段（根据实际需求调整）
    name            VARCHAR(100) NOT NULL COMMENT '名称',
    status          CHAR(1)      DEFAULT '0' COMMENT '状态（0正常 1停用）',
    remark          VARCHAR(500) DEFAULT '' COMMENT '备注',

    -- 审计字段（标准，勿删）
    create_dept     BIGINT       COMMENT '创建部门',
    create_by       BIGINT       COMMENT '创建者',
    create_time     DATETIME     COMMENT '创建时间',
    update_by       BIGINT       COMMENT '更新者',
    update_time     DATETIME     COMMENT '更新时间',
    del_flag        CHAR(1)      DEFAULT '0' COMMENT '删除标志（0代表存在 2代表删除）',
    tenant_id       VARCHAR(20)  DEFAULT '000000' COMMENT '租户编号',

    -- 索引
    PRIMARY KEY (id),
    KEY idx_create_time (create_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='{表中文说明}';
```

### 3.2 非租户表（系统表）

```sql
-- ----------------------------
-- {表中文说明}（无租户隔离）
-- ----------------------------
CREATE TABLE IF NOT EXISTS {table_name} (
    id              BIGINT       NOT NULL COMMENT '主键ID',

    -- 业务字段
    ...

    -- 审计字段（无 tenant_id）
    create_dept     BIGINT       COMMENT '创建部门',
    create_by       BIGINT       COMMENT '创建者',
    create_time     DATETIME     COMMENT '创建时间',
    update_by       BIGINT       COMMENT '更新者',
    update_time     DATETIME     COMMENT '更新时间',
    del_flag        CHAR(1)      DEFAULT '0' COMMENT '删除标志（0代表存在 2代表删除）',

    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='{表中文说明}';
```

### 3.3 关联表（多对多）

```sql
-- ----------------------------
-- {关联表中文说明}
-- ----------------------------
CREATE TABLE IF NOT EXISTS {table_name} (
    id              BIGINT       NOT NULL COMMENT '主键ID',
    {entity_a_id}   BIGINT       NOT NULL COMMENT '{实体A}ID',
    {entity_b_id}   BIGINT       NOT NULL COMMENT '{实体B}ID',

    -- 审计字段
    tenant_id       VARCHAR(20)  DEFAULT '000000' COMMENT '租户编号',

    PRIMARY KEY (id),
    UNIQUE KEY uk_{entity_a}_{entity_b} ({entity_a_id}, {entity_b_id}),
    KEY idx_{entity_b}_id ({entity_b_id})
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='{关联表中文说明}';
```

---

## 四、数据类型选择规范

### 4.1 常用类型映射

| 业务场景 | MySQL 类型 | Java 类型 | 说明 |
|----------|-----------|-----------|------|
| 主键 | bigint | Long | 雪花算法生成 |
| 名称/标题 | varchar(100-500) | String | 按实际长度选择 |
| 内容/描述 | text | String | 超长文本用 mediumtext |
| 金额 | decimal(10,2) | BigDecimal | 禁止用 float/double |
| 状态/类型 | char(1) | String | RuoYi 惯例用 char(1) |
| 数量/次数 | int | Integer | 非负用 int unsigned |
| 布尔 | char(1) | String | '0'/'1'，不用 boolean |
| 时间 | datetime | Date | 含时分秒 |
| 日期 | date | Date | 仅年月日 |
| JSON 数据 | json | String / Object | MySQL 5.7+ 支持 |
| 文件路径 | varchar(500) | String | URL 或相对路径 |
| 邮箱 | varchar(100) | String | |
| 手机号 | varchar(20) | String | 考虑国际号码 |
| IP 地址 | varchar(50) | String | IPv6 最长 45 位 |
| 枚举值 | tinyint | Integer | 少量固定选项 |

### 4.2 字符集

```sql
-- 统一使用 utf8mb4（支持 emoji）
DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci
```

### 4.3 类型选择原则

| 原则 | 说明 |
|------|------|
| 够用即可 | varchar(50) 够了就不要 varchar(500) |
| 精确优先 | 金额必须用 decimal，禁止 float |
| 时间统一 | 统一用 datetime，不用 timestamp（避免 2038 问题） |
| 状态用 char(1) | RuoYi 惯例，`0/1` 表示状态，方便字典管理 |
| 主键统一 | 全部用 bigint + 雪花算法，不用自增 |

---

## 五、索引设计规范

### 5.1 索引设计原则

| 原则 | 说明 |
|------|------|
| 区分度优先 | 联合索引中区分度高的字段放前面 |
| 覆盖索引 | 尽量让查询只走索引不回表 |
| 避免冗余 | `idx_a_b` 已包含 `idx_a`，不必重复建 |
| 控制数量 | 单表索引不超过 5 个 |
| 短字段优先 | 索引字段越短越好，减少索引体积 |

### 5.2 必须建索引的场景

| 场景 | 索引类型 | 示例 |
|------|----------|------|
| 主键 | PRIMARY KEY（自动） | `PRIMARY KEY (id)` |
| 唯一约束 | UNIQUE KEY | `uk_user_phone (phone)` |
| 外键关联 | 普通索引 | `idx_order_user_id (user_id)` |
| 常用查询条件 | 普通索引 | `idx_order_status (status)` |
| 范围查询 | 普通索引 | `idx_order_create_time (create_time)` |
| 排序字段 | 普通索引 | `idx_order_create_time (create_time)` |

### 5.3 联合索引设计

```sql
-- ✅ 正确：区分度高的 status 在前
KEY idx_order_status_time (status, create_time)

-- ❌ 错误：create_time 区分度低，应放后面
KEY idx_order_time_status (create_time, status)
```

**最左前缀原则：**
- `idx_a_b_c (a, b, c)` 支持 `a`、`a+b`、`a+b+c` 查询
- 不支持 `b`、`c`、`b+c` 查询

### 5.4 不建议建索引的场景

| 场景 | 原因 |
|------|------|
| 小表（< 1000 行） | 全表扫描更快 |
| 区分度低的字段 | 如性别、状态（值只有几个） |
| 频繁更新的字段 | 索引维护成本高 |
| TEXT/BLOB 字段 | 前缀索引可用，全字段索引不推荐 |

---

## 六、多租户设计

### 6.1 租户字段规范

```sql
-- 所有业务表必须包含
tenant_id VARCHAR(20) DEFAULT '000000' COMMENT '租户编号'
```

### 6.2 MyBatis-Plus 自动租户过滤

RuoYi 框架通过 MyBatis-Plus 插件自动在 SQL 中注入 `WHERE tenant_id = ?`：

```java
// Entity 继承 TenantEntity 即可自动生效
@Data
@EqualsAndHashCode(callSuper = true)
@TableName("biz_order")
public class BizOrder extends TenantEntity {
    // tenant_id 字段由 TenantEntity 提供，无需手动定义
}
```

### 6.3 忽略租户过滤（@TenantIgnore）

```java
// 场景1：超级管理员查看所有租户数据
@TenantIgnore
public List<BizOrderVo> selectAllForAdmin() {
    return orderMapper.selectVoList();
}

// 场景2：指定租户执行操作
TenantHelper.execute("000001", () -> {
    return orderMapper.selectVoList();
});
```

### 6.4 需要忽略租户的场景

| 场景 | 说明 |
|------|------|
| 系统管理 | 超级管理员跨租户查看数据 |
| 数据迁移 | 批量迁移某租户数据 |
| 统计报表 | 汇总所有租户数据 |
| 定时任务 | 按租户逐个执行任务 |

---

## 七、ER 图描述模板

### 7.1 实体描述

```markdown
## 实体：{表名}（{中文名}）

| 字段名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | bigint | 是 | 主键 |
| name | varchar(100) | 是 | 名称 |
| status | char(1) | 否 | 状态（0正常 1停用） |
| ... | ... | ... | ... |
```

### 7.2 关系描述

```markdown
## 实体关系

### biz_order（订单） ──→ sys_user（用户）
- **关系类型：** 多对一（N:1）
- **外键字段：** biz_order.user_id → sys_user.id
- **说明：** 一个用户可以有多个订单

### biz_order（订单） ──→ biz_order_item（订单明细）
- **关系类型：** 一对多（1:N）
- **外键字段：** biz_order_item.order_id → biz_order.id
- **说明：** 一个订单包含多个明细项

### sys_user（用户） ──→ sys_role（角色）
- **关系类型：** 多对多（M:N）
- **中间表：** sys_user_role
- **说明：** 一个用户可以有多个角色
```

### 7.3 完整 ER 图示例

```markdown
## 订单模块 ER 图

```
┌─────────────┐       ┌──────────────────┐       ┌──────────────┐
│  sys_user   │       │   biz_order      │       │ biz_order_item│
├─────────────┤  1:N  ├──────────────────┤  1:N  ├──────────────┤
│ id (PK)     │──────→│ id (PK)          │──────→│ id (PK)      │
│ username    │       │ user_id (FK)     │       │ order_id(FK) │
│ ...         │       │ order_no         │       │ product_name │
│             │       │ total_amount     │       │ quantity     │
│             │       │ status           │       │ unit_price   │
└─────────────┘       │ ...              │       └──────────────┘
                      └──────────────────┘
                              │
                              │ N:1
                              ▼
                      ┌──────────────┐
                      │  biz_product │
                      ├──────────────┤
                      │ id (PK)      │
                      │ name         │
                      │ price        │
                      └──────────────┘
```
```

---

## 八、MyBatis-Plus 映射要点

### 8.1 Entity 注解映射

```java
@Data
@EqualsAndHashCode(callSuper = true)
@TableName("biz_order")          // 表名映射
public class BizOrder extends TenantEntity {

    @TableId(value = "id")        // 主键映射，雪花算法
    private Long id;

    private String orderNo;       // 驼峰自动转下划线 order_no

    @TableField("total_amount")   // 显式指定列名（非必须，驼峰转换默认生效）
    private BigDecimal totalAmount;

    @TableLogic                   // 逻辑删除字段
    private String delFlag;

    @TableField(exist = false)    // 非数据库字段
    private String statusLabel;
}
```

### 8.2 注解说明

| 注解 | 用途 | 说明 |
|------|------|------|
| `@TableName` | 指定表名 | `@TableName("biz_order")` |
| `@TableId` | 指定主键 | `value = "id"`，默认雪花算法 |
| `@TableField` | 字段映射 | 可指定列名、是否为数据库字段 |
| `@TableLogic` | 逻辑删除 | 自动拼接 `WHERE del_flag = '0'` |
| `@TableField(exist = false)` | 非数据库字段 | 虚拟字段，不参与 CRUD |

### 8.3 主键生成策略

```java
// 默认雪花算法（推荐）
@TableId(value = "id")
private Long id;

// 显式指定
@TableId(value = "id", type = IdType.ASSIGN_ID)
private Long id;
```

> **RuoYi 默认使用雪花算法，无需手动指定。**

### 8.4 逻辑删除

```java
// Entity 中
@TableLogic
private String delFlag;

// MyBatis-Plus 自动行为：
// 查询：WHERE del_flag = '0'
// 删除：UPDATE SET del_flag = '2' WHERE id = ?
// 不会执行物理删除
```

---

## 九、数据库设计检查清单

### 9.1 基础检查

- [ ] 表名符合命名规范（小写下划线、有业务前缀）
- [ ] 字段名符合命名规范（小写下划线、见名知意）
- [ ] 所有表都有审计字段（create_dept/create_by/create_time/update_by/update_time/del_flag）
- [ ] 业务表都有 tenant_id 字段
- [ ] 主键统一使用 bigint
- [ ] 字符集统一 utf8mb4
- [ ] 每个字段都有 COMMENT
- [ ] 表有 COMMENT

### 9.2 类型检查

- [ ] 金额字段使用 decimal，不使用 float/double
- [ ] 时间字段使用 datetime，不使用 timestamp
- [ ] 状态/类型字段使用 char(1)，便于字典管理
- [ ] varchar 长度合理（不过长也不过短）
- [ ] TEXT 字段仅在确实需要长文本时使用

### 9.3 索引检查

- [ ] 外键字段有索引
- [ ] 常用查询条件有索引
- [ ] 唯一约束有唯一索引
- [ ] 联合索引字段顺序合理（区分度高的在前）
- [ ] 单表索引不超过 5 个
- [ ] 无冗余索引

### 9.4 多租户检查

- [ ] 业务表都有 tenant_id 字段
- [ ] 默认值为 '000000'
- [ ] Entity 继承 TenantEntity
- [ ] 需要跨租户查询的地方有 @TenantIgnore

### 9.5 Entity 映射检查

- [ ] @TableName 注解正确
- [ ] @TableId 注解正确
- [ ] @TableLogic 标注逻辑删除字段
- [ ] 非数据库字段有 @TableField(exist = false)
- [ ] 驼峰命名与下划线列名映射正确

---

## 十、常见设计模式

### 10.1 树形结构

```sql
-- 经典邻接表模式
CREATE TABLE biz_category (
    id          BIGINT       NOT NULL COMMENT '分类ID',
    parent_id   BIGINT       DEFAULT 0 COMMENT '父分类ID',
    name        VARCHAR(100) NOT NULL COMMENT '分类名称',
    sort_order  INT          DEFAULT 0 COMMENT '排序',
    -- 审计字段...
    PRIMARY KEY (id),
    KEY idx_parent_id (parent_id)
) ENGINE=InnoDB COMMENT='分类表';
```

### 10.2 字典值表

```sql
-- 状态/类型字段使用字典管理，不要用枚举硬编码
-- 状态值存 char(1)，通过 sys_dict_type / sys_dict_data 管理
-- 前端通过 @Translation 自动翻译
```

### 10.3 软删除

```sql
-- RuoYi 标准软删除
del_flag CHAR(1) DEFAULT '0' COMMENT '删除标志（0代表存在 2代表删除）'

-- Entity 中使用 @TableLogic
-- MyBatis-Plus 自动处理，查询自动过滤，删除变为更新
```

### 10.4 乐观锁（按需）

```java
// Entity 中添加 version 字段
@Version
private Integer version;
```

```sql
-- 对应 SQL
version INT DEFAULT 0 COMMENT '乐观锁版本号'
```

---

## 十一、经验库写入规范

```markdown
## 数据库设计
- [{日期}] {原则性描述}。{具体场景说明}
```

**示例：**
```markdown
## 数据库设计
- [260430] 金额字段必须用 decimal(10,2)，禁止 float/double。曾因使用 float 导致金额计算精度丢失
- [260430] 关联表建联合唯一索引时，注意 tenant_id 也要纳入唯一约束，否则不同租户可能冲突
```
