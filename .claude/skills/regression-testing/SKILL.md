---
name: regression-testing
description: |
  回归测试 Skill。给测试工程师 Agent 使用，在 Bug 修复或代码变更后需要执行回归测试时触发。覆盖回归策略选择、范围识别、清单模板和自动化方法，适配 RuoYi-Vue-Plus 的模块化架构。
---

# 回归测试 Skill

> 版本：v1.0 | 适配 RuoYi-Vue-Plus 5.6.0
> 后端：Spring Boot 3.5.12 + Java 17 + MyBatis-Plus + Sa-Token + 多租户
> 前端：Vue 3 + Element Plus + TypeScript

---

## 一、回归测试策略

### 1.1 三级策略

| 策略 | 范围 | 耗时 | 触发条件 |
|------|------|------|----------|
| **最小回归** | 只测修复模块 | 分钟级 | P3 Bug 修复、样式调整 |
| **标准回归** | 修复模块 + 关联模块 | 十分钟级 | P1/P2 Bug 修复、功能优化 |
| **完整回归** | 全量测试 | 小时级 | P0 Bug 修复、公共组件修改、架构变更 |

### 1.2 策略选择规则

```
变更类型判断：
  │
  ├─ 仅修改业务模块（如 ruoyi-inspection）→ 最小回归
  │
  ├─ 修改了多个模块的关联逻辑 → 标准回归
  │
  ├─ 修改了 ruoyi-common 下的公共组件 → 完整回归
  │
  ├─ 修改了权限相关代码（Sa-Token配置/权限注解）→ 完整回归
  │
  ├─ 修改了多租户相关代码（TenantEntity/租户插件）→ 完整回归
  │
  └─ 修改了数据库表结构 → 标准回归（关联模块）或 完整回归
```

### 1.3 各策略的测试层级

| 测试层级 | 最小回归 | 标准回归 | 完整回归 |
|----------|---------|---------|---------|
| L1 单元测试 | ✅ 修复模块 | ✅ 修复+关联 | ✅ 全量 |
| L2 集成测试 | ✅ 修复模块 | ✅ 修复+关联 | ✅ 全量 |
| L3 UI测试 | ❌ | ✅ 修复模块 | ✅ 全量 |
| L4 E2E测试 | ❌ | ❌ | ✅ 核心流程 |
| L5 AI测试 | ✅ 修复模块 | ✅ 修复+关联 | ✅ 变更影响范围 |

---

## 二、回归范围识别

### 2.1 代码变更分析

```bash
# 查看变更文件列表
cd {BACKEND_DIR}/
git diff --name-only HEAD~1

# 按模块分类
git diff --name-only HEAD~1 | grep "ruoyi-modules/" | cut -d'/' -f2 | sort | uniq

# 按类型分类
git diff --name-only HEAD~1 | grep "\.java$"   # Java 文件
git diff --name-only HEAD~1 | grep "\.vue$"    # Vue 文件
git diff --name-only HEAD~1 | grep "\.xml$"    # XML 文件
git diff --name-only HEAD~1 | grep "\.sql$"    # SQL 文件
```

### 2.2 依赖关系分析

**模块依赖矩阵：**

```
ruoyi-common-core       ← 几乎所有模块依赖
ruoyi-common-mybatis    ← 所有业务模块依赖
ruoyi-common-redis      ← 需要缓存的模块
ruoyi-common-security   ← 所有需要认证的模块
ruoyi-common-tenant     ← 所有业务模块依赖
ruoyi-common-web        ← 所有模块依赖
```

**影响范围规则：**

| 修改内容 | 影响范围 |
|----------|----------|
| 修改 Entity | 同模块的 Bo、Vo、Mapper、Service、Controller |
| 修改 Service | 同模块的 Controller、其他调用该 Service 的模块 |
| 修改 Mapper XML | 同模块的 Service |
| 修改 ruoyi-common-core | 全部模块（完整回归） |
| 修改 ruoyi-common-mybatis | 所有业务模块（完整回归） |
| 修改 ruoyi-common-tenant | 所有业务模块（完整回归） |
| 修改 ruoyi-common-security | 所有需要认证的模块（完整回归） |
| 修改前端 API 层 | 对应页面组件 |
| 修改前端公共组件 | 所有使用该组件的页面 |

### 2.3 自动影响分析脚本

```bash
#!/bin/bash
# analyze-impact.sh - 分析变更影响范围

echo "=== 变更文件 ==="
CHANGED=$(git diff --name-only HEAD~1)
echo "$CHANGED"

echo ""
echo "=== 变更模块 ==="
echo "$CHANGED" | grep "ruoyi-modules/" | cut -d'/' -f2 | sort | uniq

echo ""
echo "=== 影响判断 ==="
if echo "$CHANGED" | grep -q "ruoyi-common"; then
    echo "⚠️  修改了公共组件，建议完整回归"
elif echo "$CHANGED" | grep -q "ruoyi-framework"; then
    echo "⚠️  修改了框架层，建议完整回归"
else
    MODULES=$(echo "$CHANGED" | grep "ruoyi-modules/" | cut -d'/' -f2 | sort | uniq)
    echo "✅ 仅修改了业务模块：$MODULES"
    echo "建议标准回归"
fi
```

---

## 三、回归测试清单模板

### 3.1 Bug 修复回归清单

```markdown
# 回归测试清单

## Bug 信息
- Bug 编号：{编号}
- Bug 标题：{一句话描述}
- 优先级：P0/P1/P2/P3
- 修复模块：{模块名}
- 修复内容：{简述修复了什么}
- 修复人：{开发者}

## 回归策略：{最小/标准/完整}

## 回归范围

| 模块 | 回归原因 | 测试类型 | 用例数 | 负责人 |
|------|----------|----------|--------|--------|
| {修复模块} | 直接修复 | 单元+集成 | N | |
| {关联模块1} | 共用 Service | 单元 | N | |
| {关联模块2} | 数据依赖 | 集成 | N | |

## 回归结果

### 修复验证

| # | 测试项 | 操作步骤 | 预期结果 | 实际结果 | 状态 |
|---|--------|----------|----------|----------|------|
| 1 | Bug 已修复 | {原 Bug 复现步骤} | {不再出现} | | PASS/FAIL |
| 2 | 修复无副作用 | {正常操作流程} | {功能正常} | | PASS/FAIL |

### 关联模块回归

| # | 模块 | 测试项 | 操作步骤 | 预期结果 | 实际结果 | 状态 |
|---|------|--------|----------|----------|----------|------|
| 1 | {模块1} | {功能1} | | | | PASS/FAIL |
| 2 | {模块2} | {功能2} | | | | PASS/FAIL |

## 回归结论
- 回归结果：PASS / FAIL
- 是否可以发布：是 / 否
- 遗留问题：{无 / 描述}
```

### 3.2 功能变更回归清单

```markdown
# 功能变更回归测试清单

## 变更信息
- 变更类型：新增功能 / 功能优化 / 重构 / 配置变更
- 变更模块：{模块名}
- 变更描述：{简述}

## 回归范围

| 模块 | 回归原因 | 测试类型 |
|------|----------|----------|
| | | |

## 回归结果

| # | 测试项 | 测试类型 | 结果 | 备注 |
|---|--------|----------|------|------|
| 1 | | 单元 | PASS/FAIL | |
| 2 | | 集成 | PASS/FAIL | |
| 3 | | UI | PASS/FAIL | |
| 4 | | E2E | PASS/FAIL | |
```

---

## 四、RuoYi 项目回归要点

### 4.1 公共组件回归（必须全量）

| 修改内容 | 回归范围 |
|----------|----------|
| `ruoyi-common-core` 修改 | 全部模块 |
| `ruoyi-common-mybatis` 修改 | 全部业务模块（Mapper/Service/分页） |
| `ruoyi-common-redis` 修改 | 所有使用缓存的模块 |
| `ruoyi-common-security` 修改 | 全部模块（认证/鉴权） |
| `ruoyi-common-tenant` 修改 | 全部业务模块（租户隔离） |
| `ruoyi-common-log` 修改 | 全部模块（操作日志） |
| `ruoyi-common-web` 修改 | 全部模块（全局异常/响应体） |
| `ruoyi-framework` 修改 | 全部模块（启动/配置） |

### 4.2 权限相关回归

| 修改内容 | 回归范围 |
|----------|----------|
| 新增菜单/权限 | 该菜单关联的所有页面和接口 |
| 修改角色权限 | 该角色的所有功能 |
| 修改 @SaCheckPermission | 该接口的前端+后端权限 |
| 修改数据权限配置 | 涉及数据隔离的所有查询 |

### 4.3 多租户相关回归

| 修改内容 | 回归范围 |
|----------|----------|
| 新增业务表 | 该表的 CRUD + 租户隔离 |
| 修改租户插件配置 | 全部业务模块的租户隔离 |
| 修改 @TenantIgnore 使用 | 该方法的数据可见性 |
| 修改租户切换逻辑 | 全部业务模块 |

### 4.4 前端公共组件回归

| 修改内容 | 回归范围 |
|----------|----------|
| 修改 `src/components/` 下的公共组件 | 所有引用该组件的页面 |
| 修改 `src/utils/request.ts` | 全部接口请求 |
| 修改 `src/directive/` 下的指令 | 使用该指令的页面 |
| 修改 `src/store/` 全局 Store | 依赖该 Store 的页面 |
| 修改路由配置 | 菜单和页面访问 |

---

## 五、回归测试自动化

### 5.1 基于 git diff 自动识别

```bash
#!/bin/bash
# auto-regression.sh - 自动回归测试

cd {BACKEND_DIR}/

# 1. 获取变更模块
CHANGED_MODULES=$(git diff --name-only HEAD~1 | grep "ruoyi-modules/" | cut -d'/' -f2 | sort | uniq)

# 2. 判断回归策略
if git diff --name-only HEAD~1 | grep -q "ruoyi-common"; then
    echo "🔄 检测到公共组件修改，执行完整回归"
    mvn test
else
    echo "🔄 仅业务模块修改：$CHANGED_MODULES"
    for module in $CHANGED_MODULES; do
        echo "  📦 测试模块: $module"
        mvn test -pl ruoyi-modules/$module -am
    done
fi
```

### 5.2 自动回归执行流程

```
代码提交
  │
  ▼
git diff 分析变更
  │
  ├─ 公共组件 → 完整回归
  │   └─ mvn test（全量） + npx vitest run（全量）
  │
  ├─ 单模块 → 最小回归
  │   └─ mvn test -pl ruoyi-modules/{module} -am
  │
  └─ 多模块 → 标准回归
      └─ mvn test -pl ruoyi-modules/{module1},ruoyi-modules/{module2} -am
  │
  ▼
生成回归报告
```

### 5.3 前端自动回归

```bash
cd {FRONTEND_DIR}/

# 获取变更的 Vue 文件
CHANGED_VUE=$(git diff --name-only HEAD~1 | grep "\.vue$" | grep "views/")

if [ -n "$CHANGED_VUE" ]; then
    echo "🔄 检测到前端页面变更"
    # 运行前端单元测试
    npx vitest run
fi
```

---

## 六、回归测试检查清单

### 6.1 回归前检查

- [ ] 已获取变更文件列表（git diff）
- [ ] 已确定回归策略（最小/标准/完整）
- [ ] 已识别影响范围（关联模块）
- [ ] 回归用例已准备

### 6.2 回归执行检查

- [ ] Bug 修复已验证（原 Bug 不再复现）
- [ ] 修复模块功能正常
- [ ] 关联模块功能正常
- [ ] 公共功能未受影响（登录/权限/租户）
- [ ] 无新增 Bug

### 6.3 回归后检查

- [ ] 回归报告已生成
- [ ] PASS → 可以发布
- [ ] FAIL → 问题已反馈开发修复
- [ ] 经验库已更新（如有值得记录的问题）

---

## 七、经验库写入规范

```markdown
## 回归测试
- [{日期}] {原则性描述}。{具体场景说明}
```

**示例：**
```markdown
## 回归测试
- [260430] 修改 ruoyi-common-mybatis 的 BaseMapperPlus 方法后必须全量回归。曾因修改 selectVoPage 默认行为导致全部分页查询异常
- [260430] 修改前端 request.ts 的响应拦截器后，需回归所有模块的接口调用。曾因修改错误处理逻辑导致部分模块的错误提示丢失
```
