---
name: backend-coding
description: |
  后端编码规范 — RuoYi-Vue-Plus项目定制版。
  基于项目实际代码（ruoyi-inspection、ruoyi-knowledge、ruoyi-system）总结的编码规范、
  模块创建步骤、完整代码模板（含import路径）、常见模式。
  所有代码模板必须与项目实际代码风格一致，不能凭空编写。
---

# 后端编码规范（RuoYi-Vue-Plus项目定制）

> 本文档基于 `{BACKEND_DIR}` 项目实际代码生成，所有模板、注解、import路径均来自真实代码。
> **铁律：一切以项目代码为准，不凭空编写。**

---

## 一、编码规范（铁律）

### 1.1 项目技术栈

| 层次 | 技术 |
|------|------|
| 框架 | Spring Boot 3.x + MyBatis-Plus |
| 权限 | Sa-Token（`@SaCheckPermission`） |
| ORM | MyBatis-Plus（`BaseMapperPlus`） |
| 对象转换 | MapStruct Plus（`@AutoMapper`，`io.github.linpeilie`） |
| 参数校验 | Jakarta Validation（`@Validated`、`@NotBlank`、`@NotNull`、`@Size`） |
| Excel | EasyExcel（`cn.idev.excel`，`ExcelUtil`） |
| 工具库 | Hutool（`cn.hutool`）、Apache Commons |
| 日志 | Slf4j + Lombok `@Slf4j` |
| 多租户 | 框架自带，Entity继承 `TenantEntity` |
| 逻辑删除 | `@TableLogic`，`del_flag` 字段 |
| 自动填充 | `create_by`、`create_time`、`update_by`、`update_time`、`create_dept` |

### 1.2 包结构规范

新建模块 `ruoyi-{模块名}` 标准包结构：

```
org.dromara.{模块名}/
├── controller/          # 控制器
├── domain/
│   ├── {Entity}.java    # 实体类
│   ├── bo/              # 业务对象（查询/新增/修改入参）
│   └── vo/              # 视图对象（列表Vo + 详情Vo）
├── mapper/              # Mapper接口
├── service/
│   ├── I{Entity}Service.java        # Service接口
│   └── impl/{Entity}ServiceImpl.java  # Service实现
└── job/                 # 定时任务（可选）
```

Mapper XML 位置：`src/main/resources/mapper/{模块名}/`

### 1.3 命名规范

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| Entity | `{业务名}` | `AlertRecord`、`InspectionReport` |
| Bo（查询） | `{业务名}Bo`，继承 `BaseEntity` | `AlertRecordBo` |
| Vo（列表） | `{业务名}Vo`，实现 `Serializable` | `AlertRecordVo` |
| Vo（详情） | `{业务名}DetailVo`，实现 `Serializable` | `AlertRecordDetailVo` |
| Mapper接口 | `{业务名}Mapper`，继承 `BaseMapperPlus<Entity, Vo>` | `AlertRecordMapper` |
| Service接口 | `I{业务名}Service` | `IAlertRecordService` |
| Service实现 | `{业务名}ServiceImpl` | `AlertRecordServiceImpl` |
| Controller | `{业务名}Controller`，继承 `BaseController` | `AlertRecordController` |
| 表名 | 全小写下划线 | `alert_record` |

### 1.4 注解使用规范

**Entity 注解：**
- `@Data` + `@EqualsAndHashCode(callSuper = true)`
- `@TableName("表名")`
- `@TableId(value = "id")` — 主键
- `@TableLogic` — 逻辑删除字段
- 继承 `TenantEntity`（需要多租户时）或 `BaseEntity`（不需要时）

**Bo 注解：**
- `@Data` + `@EqualsAndHashCode(callSuper = true)`
- 继承 `BaseEntity`
- 校验注解：`@NotBlank`、`@NotNull`、`@Size`
- 如需双向转换加 `@AutoMapper(target = Entity.class, reverseConvertGenerate = false)`

**Vo 注解：**
- `@Data`
- 实现 `Serializable`
- `@AutoMapper(target = Entity.class)` — 自动映射
- Excel导出：`@ExcelIgnoreUnannotated` + `@ExcelProperty`
- 字典翻译：`@Translation(type = TransConstant.USER_ID_TO_NICKNAME, mapper = "createBy")`

**Controller 注解：**
- `@Validated` + `@RequiredArgsConstructor`
- `@RestController` + `@RequestMapping("/模块名/业务名")`
- 继承 `BaseController`
- 权限：`@SaCheckPermission("模块名:业务名:操作")`
- 操作日志：`@Log(title = "业务名", businessType = BusinessType.XXX)`
- 防重提交：`@RepeatSubmit`（写操作必须加）

**Service 注解：**
- `@Slf4j` + `@RequiredArgsConstructor`
- `@Service`

### 1.5 HTTP 接口规范

| 操作 | 方法 | 路径示例 | 权限后缀 |
|------|------|----------|----------|
| 分页列表 | `GET` | `/list` | `:list` 或 `:query` |
| 详情 | `GET` | `/{id}` | `:query` |
| 新增 | `POST` | `/` | `:add` |
| 修改 | `PUT` | `/` | `:edit` |
| 删除 | `DELETE` | `/{ids}` | `:remove` 或 `:delete` |
| 导出 | `POST` | `/export` | `:export` |
| 导入 | `POST` | `/importData` | `:import` |

### 1.6 统一响应体

```java
// 成功（无数据）
R.ok()

// 成功（带数据）
R.ok(data)

// 成功（带消息）
R.ok("操作成功")

// 失败
R.fail("错误信息")

// 警告
R.warn("警告信息")

// CRUD 判断（用于 deleteByIds 等）
toAjax(rows > 0)   // BaseController 提供
```

分页查询直接返回 `TableDataInfo<Vo>`，框架自动序列化为 `{total, rows, code, msg}`。

---

## 二、新建业务模块步骤

### 2.1 创建 Maven 模块

1. 在 `{BACKEND_DIR}/ruoyi-modules/` 下创建 `ruoyi-{模块名}/`
2. 编写 `pom.xml`，parent 指向 `ruoyi-modules`
3. 在 `{BACKEND_DIR}/ruoyi-modules/pom.xml` 的 `<modules>` 中添加子模块

**标准 pom.xml 依赖（参考 ruoyi-inspection）：**

```xml
<dependencies>
    <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>ruoyi-common-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>ruoyi-common-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>ruoyi-common-mybatis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>ruoyi-common-log</artifactId>
    </dependency>
    <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>ruoyi-common-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>ruoyi-common-tenant</artifactId>
    </dependency>
    <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>ruoyi-common-translation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.dromara</groupId>
        <artifactId>ruoyi-common-idempotent</artifactId>
    </dependency>
</dependencies>
```

按需增减（如需要 SSE 加 `ruoyi-common-sse`，需要定时任务加 `ruoyi-common-job`，需要 Excel 加 `ruoyi-common-excel`）。

### 2.2 创建建表 SQL

在 `sql/` 目录下创建 SQL 文件。

### 2.3 按顺序创建代码文件

1. Entity → 2. Bo → 3. Vo → 4. Mapper接口 → 5. Mapper XML → 6. Service接口 → 7. Service实现 → 8. Controller

### 2.4 配置检查

- Mapper 扫描路径：`mybatis-plus.mapperPackage: org.dromara.**.mapper`（已全局配置，无需额外配置）
- 确保 Mapper XML 的 namespace 与 Mapper 接口全限定名一致

---

## 三、代码模板

> 以下模板全部基于项目真实代码（ruoyi-inspection 模块），`{xxx}` 为占位符。

### 3.1 Entity 模板

```java
package org.dromara.{模块名}.domain;

import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import com.baomidou.mybatisplus.annotation.TableLogic;
import lombok.Data;
import lombok.EqualsAndHashCode;
import org.dromara.common.tenant.core.TenantEntity;

import java.io.Serial;
import java.util.Date;

/**
 * {业务中文名}对象 {表名}
 *
 * @author {author}
 */
@Data
@EqualsAndHashCode(callSuper = true)
@TableName("{表名}")
public class {Entity名} extends TenantEntity {

    @Serial
    private static final long serialVersionUID = 1L;

    /**
     * 主键
     */
    @TableId(value = "id")
    private Long id;

    /**
     * {字段中文名}
     */
    private String {字段名};

    // ========== 更多业务字段 ==========

    /**
     * 逻辑删除标志
     */
    @TableLogic
    private String delFlag;
}
```

**说明：**
- 多租户模块继承 `TenantEntity`（自带 `tenantId`、`createBy`、`createTime`、`updateBy`、`updateTime`、`createDept`）
- 不需要多租户的模块继承 `BaseEntity`（无 `tenantId`）
- `@TableLogic` 的 `delFlag` 类型为 `String`（值为 "0"/"1"）
- 日期类型用 `java.util.Date`
- 数值类型：`Integer`、`Long`、`BigDecimal`

### 3.2 Bo 模板

**查询 Bo（仅查询条件，继承 BaseEntity）：**

```java
package org.dromara.{模块名}.domain.bo;

import lombok.Data;
import lombok.EqualsAndHashCode;
import org.dromara.common.mybatis.core.domain.BaseEntity;

import java.io.Serial;

/**
 * {业务中文名}查询业务对象
 *
 * @author {author}
 */
@Data
@EqualsAndHashCode(callSuper = true)
public class {Entity名}Bo extends BaseEntity {

    @Serial
    private static final long serialVersionUID = 1L;

    /**
     * {筛选条件中文名}
     */
    private String {筛选字段名};
}
```

**增改 Bo（带校验注解，用于新增/修改）：**

```java
package org.dromara.{模块名}.domain.bo;

import io.github.linpeilie.annotations.AutoMapper;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import lombok.Data;
import lombok.EqualsAndHashCode;
import org.dromara.common.mybatis.core.domain.BaseEntity;
import org.dromara.{模块名}.domain.{Entity名};

/**
 * {业务中文名}业务对象 {表名}
 *
 * @author {author}
 */
@Data
@EqualsAndHashCode(callSuper = true)
@AutoMapper(target = {Entity名}.class, reverseConvertGenerate = false)
public class {Entity名}Bo extends BaseEntity {

    @Serial
    private static final long serialVersionUID = 1L;

    /**
     * 主键（修改时必填）
     */
    private Long id;

    /**
     * {字段中文名}
     */
    @NotBlank(message = "{字段中文名}不能为空")
    @Size(min = 0, max = 100, message = "{字段中文名}长度不能超过{max}个字符")
    private String {字段名};

    /**
     * {字段中文名}
     */
    @NotNull(message = "{字段中文名}不能为空")
    private Integer {数字字段名};
}
```

**说明：**
- 查询 Bo 只放查询条件字段，继承 `BaseEntity` 获得 `params` Map（用于 beginTime/endTime 范围查询）
- 增改 Bo 加 `@AutoMapper` 实现双向转换（`reverseConvertGenerate = false` 避免生成反向转换方法）
- 校验注解：`@NotBlank`（字符串）、`@NotNull`（数字/对象）、`@Size`（长度限制）

### 3.3 Vo 模板

**列表 Vo（轻量，用于分页列表展示）：**

```java
package org.dromara.{模块名}.domain.vo;

import lombok.Data;
import org.dromara.{模块名}.domain.{Entity名};
import io.github.linpeilie.annotations.AutoMapper;

import java.io.Serial;
import java.io.Serializable;
import java.util.Date;

/**
 * {业务中文名}列表视图对象
 *
 * @author {author}
 */
@Data
@AutoMapper(target = {Entity名}.class)
public class {Entity名}Vo implements Serializable {

    @Serial
    private static final long serialVersionUID = 1L;

    /**
     * 主键
     */
    private Long id;

    /**
     * {展示字段中文名}
     */
    private String {字段名};

    /**
     * 创建时间
     */
    private Date createTime;
}
```

**详情 Vo（完整，包含所有需要展示的字段）：**

```java
package org.dromara.{模块名}.domain.vo;

import lombok.Data;
import org.dromara.{模块名}.domain.{Entity名};
import io.github.linpeilie.annotations.AutoMapper;

import java.io.Serial;
import java.io.Serializable;
import java.util.Date;

/**
 * {业务中文名}详情视图对象
 *
 * @author {author}
 */
@Data
@AutoMapper(target = {Entity名}.class)
public class {Entity名}DetailVo implements Serializable {

    @Serial
    private static final long serialVersionUID = 1L;

    /**
     * 主键
     */
    private Long id;

    // ========== 所有需要展示的字段 ==========

    /**
     * {字段中文名}
     */
    private String {字段名};

    /**
     * 创建时间
     */
    private Date createTime;

    /**
     * 更新时间
     */
    private Date updateTime;
}
```

**带 Excel 导出注解的 Vo：**

```java
package org.dromara.{模块名}.domain.vo;

import cn.idev.excel.annotation.ExcelIgnoreUnannotated;
import cn.idev.excel.annotation.ExcelProperty;
import io.github.linpeilie.annotations.AutoMapper;
import lombok.Data;
import org.dromara.common.excel.annotation.ExcelDictFormat;
import org.dromara.common.excel.convert.ExcelDictConvert;
import org.dromara.common.translation.annotation.Translation;
import org.dromara.common.translation.constant.TransConstant;
import org.dromara.{模块名}.domain.{Entity名};

import java.io.Serial;
import java.io.Serializable;
import java.util.Date;

/**
 * {业务中文名}视图对象 {表名}
 *
 * @author {author}
 */
@Data
@ExcelIgnoreUnannotated
@AutoMapper(target = {Entity名}.class)
public class {Entity名}Vo implements Serializable {

    @Serial
    private static final long serialVersionUID = 1L;

    /**
     * 主键
     */
    @ExcelProperty(value = "主键")
    private Long id;

    /**
     * {字段中文名}
     */
    @ExcelProperty(value = "{字段中文名}")
    private String {字段名};

    /**
     * 状态
     */
    @ExcelProperty(value = "状态", converter = ExcelDictConvert.class)
    @ExcelDictFormat(dictType = "sys_normal_disable")
    private String status;

    /**
     * 创建时间
     */
    @ExcelProperty(value = "创建时间")
    private Date createTime;

    /**
     * 创建者名称
     */
    @Translation(type = TransConstant.USER_ID_TO_NICKNAME, mapper = "createBy")
    private String createByName;
}
```

**说明：**
- 列表 Vo 和详情 Vo 是两个不同的类，列表 Vo 只包含列表展示需要的字段，详情 Vo 包含完整字段
- `@AutoMapper(target = Entity.class)` 自动生成 MapStruct 转换代码
- Excel 导出：`@ExcelIgnoreUnannotated` 忽略未标注字段，`@ExcelProperty` 标注需要导出的字段
- 字典翻译：`@Translation` 注解实现关联查询翻译

### 3.4 Mapper 接口模板

```java
package org.dromara.{模块名}.mapper;

import org.dromara.common.mybatis.core.mapper.BaseMapperPlus;
import org.dromara.{模块名}.domain.{Entity名};
import org.dromara.{模块名}.domain.vo.{Entity名}Vo;

/**
 * {业务中文名}Mapper接口
 *
 * @author {author}
 */
public interface {Entity名}Mapper extends BaseMapperPlus<{Entity名}, {Entity名}Vo> {

}
```

**说明：**
- 继承 `BaseMapperPlus<Entity, DefaultVo>`，第二个泛型是默认的 Vo 类型
- `BaseMapperPlus` 自带 `selectVoById`、`selectVoPage`、`selectVoList`、`insertBatch`、`updateBatchById` 等方法
- 可用 `selectVoById(id, OtherVo.class)` 查询时指定返回不同的 Vo（如详情 Vo）
- 如果有复杂查询，在接口中定义方法并在 XML 中编写 SQL

### 3.5 Mapper XML 模板

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="org.dromara.{模块名}.mapper.{Entity名}Mapper">

</mapper>
```

**说明：**
- 如果使用 MyBatis-Plus 的 `LambdaQueryWrapper` 构建查询，XML 可以为空（项目中的实际做法）
- XML 位置：`src/main/resources/mapper/{模块名}/{Entity名}Mapper.xml`
- namespace 必须与 Mapper 接口全限定名一致

**带自定义查询的 XML 示例：**

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="org.dromara.{模块名}.mapper.{Entity名}Mapper">

    <select id="selectCustomList" resultType="org.dromara.{模块名}.domain.vo.{Entity名}Vo">
        SELECT * FROM {表名}
        WHERE del_flag = '0'
        <if test="name != null and name != ''">
            AND name LIKE CONCAT('%', #{name}, '%')
        </if>
        ORDER BY create_time DESC
    </select>

</mapper>
```

### 3.6 Service 接口模板

```java
package org.dromara.{模块名}.service;

import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.{模块名}.domain.bo.{Entity名}Bo;
import org.dromara.{模块名}.domain.vo.{Entity名}DetailVo;
import org.dromara.{模块名}.domain.vo.{Entity名}Vo;

import java.util.Collection;

/**
 * {业务中文名}Service接口
 *
 * @author {author}
 */
public interface I{Entity名}Service {

    /**
     * 分页查询{业务中文名}列表
     */
    TableDataInfo<{Entity名}Vo> selectPageList({Entity名}Bo bo, PageQuery pageQuery);

    /**
     * 查询{业务中文名}详情
     */
    {Entity名}DetailVo selectDetailById(Long id);

    /**
     * 新增{业务中文名}
     */
    Boolean insertByBo({Entity名}Bo bo);

    /**
     * 修改{业务中文名}
     */
    Boolean updateByBo({Entity名}Bo bo);

    /**
     * 批量删除{业务中文名}
     */
    Boolean deleteByIds(Collection<Long> ids);
}
```

### 3.7 Service 实现模板

```java
package org.dromara.{模块名}.service.impl;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.dromara.common.core.exception.ServiceException;
import org.dromara.common.core.utils.StringUtils;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.{模块名}.domain.{Entity名};
import org.dromara.{模块名}.domain.bo.{Entity名}Bo;
import org.dromara.{模块名}.domain.vo.{Entity名}DetailVo;
import org.dromara.{模块名}.domain.vo.{Entity名}Vo;
import org.dromara.{模块名}.mapper.{Entity名}Mapper;
import org.dromara.{模块名}.service.I{Entity名}Service;
import org.springframework.stereotype.Service;

import java.util.Collection;
import java.util.Map;

/**
 * {业务中文名}Service业务层处理
 *
 * @author {author}
 */
@Slf4j
@RequiredArgsConstructor
@Service
public class {Entity名}ServiceImpl implements I{Entity名}Service {

    private final {Entity名}Mapper {entity名}Mapper;

    // ========== 查询 ==========

    @Override
    public TableDataInfo<{Entity名}Vo> selectPageList({Entity名}Bo bo, PageQuery pageQuery) {
        LambdaQueryWrapper<{Entity名}> lqw = buildQueryWrapper(bo);
        Page<{Entity名}Vo> result = {entity名}Mapper.selectVoPage(pageQuery.build(), lqw);
        return TableDataInfo.build(result);
    }

    @Override
    public {Entity名}DetailVo selectDetailById(Long id) {
        return {entity名}Mapper.selectVoById(id, {Entity名}DetailVo.class);
    }

    // ========== 增改 ==========

    @Override
    public Boolean insertByBo({Entity名}Bo bo) {
        {Entity名} entity = new {Entity名}();
        // TODO: 设置字段值（Bo → Entity 映射）
        // 如果 Bo 上有 @AutoMapper(reverseConvertGenerate = true)，可以用 MapstructUtils.convert
        boolean flag = {entity名}Mapper.insert(entity) > 0;
        if (flag) {
            // bo.setId(entity.getId());  // 回填主键
        }
        return flag;
    }

    @Override
    public Boolean updateByBo({Entity名}Bo bo) {
        {Entity名} entity = {entity名}Mapper.selectById(bo.getId());
        if (entity == null) {
            throw new ServiceException("{业务中文名}不存在");
        }
        // TODO: 更新字段值
        return {entity名}Mapper.updateById(entity) > 0;
    }

    // ========== 删除 ==========

    @Override
    public Boolean deleteByIds(Collection<Long> ids) {
        return {entity名}Mapper.deleteByIds(ids) > 0;
    }

    // ========== 私有方法 ==========

    private LambdaQueryWrapper<{Entity名}> buildQueryWrapper({Entity名}Bo bo) {
        Map<String, Object> params = bo.getParams();
        LambdaQueryWrapper<{Entity名}> lqw = Wrappers.lambdaQuery();
        lqw.eq(StringUtils.isNotBlank(bo.getStatus()), {Entity名}::getStatus, bo.getStatus());
        lqw.like(StringUtils.isNotBlank(bo.getName()), {Entity名}::getName, bo.getName());
        // 时间范围查询（通过 params 传递 beginTime/endTime）
        lqw.between(params.get("beginTime") != null && params.get("endTime") != null,
            {Entity名}::getCreateTime, params.get("beginTime"), params.get("endTime"));
        lqw.orderByDesc({Entity名}::getCreateTime);
        return lqw;
    }
}
```

**说明：**
- 构造函数注入：`@RequiredArgsConstructor` + `private final` 字段
- 分页查询：`pageQuery.build()` 构建 `Page` 对象，`selectVoPage` 自动分页 + Vo 转换
- 查询详情时指定 Vo 类型：`selectVoById(id, DetailVo.class)`
- LambdaQueryWrapper 条件构造：`eq`（等于）、`like`（模糊）、`between`（范围）、`orderByDesc`（降序）
- `StringUtils.isNotBlank()` 用于条件判断，为空时跳过该条件
- 异常抛出：`throw new ServiceException("错误信息")`

### 3.8 Controller 模板

**标准 CRUD Controller：**

```java
package org.dromara.{模块名}.controller;

import cn.dev33.satoken.annotation.SaCheckPermission;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.validation.constraints.NotNull;
import lombok.RequiredArgsConstructor;
import org.dromara.common.core.domain.R;
import org.dromara.common.excel.utils.ExcelUtil;
import org.dromara.common.idempotent.annotation.RepeatSubmit;
import org.dromara.common.log.annotation.Log;
import org.dromara.common.log.enums.BusinessType;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.common.web.core.BaseController;
import org.dromara.{模块名}.domain.bo.{Entity名}Bo;
import org.dromara.{模块名}.domain.vo.{Entity名}DetailVo;
import org.dromara.{模块名}.domain.vo.{Entity名}Vo;
import org.dromara.{模块名}.service.I{Entity名}Service;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * {业务中文名}Controller
 *
 * @author {author}
 */
@Validated
@RequiredArgsConstructor
@RestController
@RequestMapping("/{模块名}/{业务名}")
public class {Entity名}Controller extends BaseController {

    private final I{Entity名}Service {entity名}Service;

    /**
     * 分页查询{业务中文名}列表
     */
    @SaCheckPermission("{模块名}:{业务名}:list")
    @GetMapping("/list")
    public TableDataInfo<{Entity名}Vo> list({Entity名}Bo bo, PageQuery pageQuery) {
        return {entity名}Service.selectPageList(bo, pageQuery);
    }

    /**
     * 获取{业务中文名}详情
     */
    @SaCheckPermission("{模块名}:{业务名}:query")
    @GetMapping("/{id}")
    public R<{Entity名}DetailVo> detail(@NotNull(message = "ID不能为空") @PathVariable Long id) {
        return R.ok({entity名}Service.selectDetailById(id));
    }

    /**
     * 新增{业务中文名}
     */
    @SaCheckPermission("{模块名}:{业务名}:add")
    @Log(title = "{业务中文名}", businessType = BusinessType.INSERT)
    @RepeatSubmit
    @PostMapping
    public R<Void> add(@Validated @RequestBody {Entity名}Bo bo) {
        return toAjax({entity名}Service.insertByBo(bo));
    }

    /**
     * 修改{业务中文名}
     */
    @SaCheckPermission("{模块名}:{业务名}:edit")
    @Log(title = "{业务中文名}", businessType = BusinessType.UPDATE)
    @RepeatSubmit
    @PutMapping
    public R<Void> edit(@Validated @RequestBody {Entity名}Bo bo) {
        return toAjax({entity名}Service.updateByBo(bo));
    }

    /**
     * 删除{业务中文名}
     */
    @SaCheckPermission("{模块名}:{业务名}:remove")
    @Log(title = "{业务中文名}", businessType = BusinessType.DELETE)
    @DeleteMapping("/{ids}")
    public R<Void> delete(@PathVariable List<Long> ids) {
        return toAjax({entity名}Service.deleteByIds(ids));
    }

    /**
     * 导出{业务中文名}列表
     */
    @Log(title = "{业务中文名}", businessType = BusinessType.EXPORT)
    @SaCheckPermission("{模块名}:{业务名}:export")
    @PostMapping("/export")
    public void export({Entity名}Bo bo, HttpServletResponse response) {
        List<{Entity名}Vo> list = {entity名}Service.selectList(bo);
        ExcelUtil.exportExcel(list, "{业务中文名}数据", {Entity名}Vo.class, response);
    }
}
```

**说明：**
- 类级别 `@Validated` + 方法级别 `@Validated @RequestBody` 双重校验
- `@SaCheckPermission` 权限字符串格式：`模块名:业务名:操作`
- `@Log` 记录操作日志，`businessType` 对应操作类型：INSERT/UPDATE/DELETE/EXPORT/IMPORT
- `@RepeatSubmit` 防止重复提交（写操作必须加）
- 分页列表返回 `TableDataInfo<Vo>`（无需包 `R`）
- 详情/增/改/删返回 `R<Void>` 或 `R<Data>`
- `toAjax()` 方法来自 `BaseController`，将 `boolean/int` 转为 `R<Void>`
- 删除接口支持批量：`@PathVariable List<Long> ids`

### 3.9 建表 SQL 模板

```sql
-- ----------------------------
-- {业务中文名}表
-- ----------------------------
DROP TABLE IF EXISTS `{表名}`;
CREATE TABLE `{表名}` (
    `id`          BIGINT(20)   NOT NULL COMMENT '主键',
    `tenant_id`   VARCHAR(20)  DEFAULT '000000' COMMENT '租户编号',
    `{字段名}`     VARCHAR(100) DEFAULT NULL COMMENT '{字段中文名}',
    `status`      CHAR(1)      DEFAULT '0' COMMENT '状态（0正常 1停用）',
    `del_flag`    CHAR(1)      DEFAULT '0' COMMENT '删除标志（0存在 2删除）',
    `create_dept` BIGINT(20)   DEFAULT NULL COMMENT '创建部门',
    `create_by`   BIGINT(20)   DEFAULT NULL COMMENT '创建者',
    `create_time` DATETIME     DEFAULT NULL COMMENT '创建时间',
    `update_by`   BIGINT(20)   DEFAULT NULL COMMENT '更新者',
    `update_time` DATETIME     DEFAULT NULL COMMENT '更新时间',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='{业务中文名}表';
```

**说明：**
- 主键：`BIGINT(20)`，雪花算法生成
- 多租户字段：`tenant_id` VARCHAR(20)，默认 '000000'
- 逻辑删除：`del_flag` CHAR(1)，'0' 存在 / '2' 删除
- 审计字段：`create_dept`、`create_by`、`create_time`、`update_by`、`update_time`
- 字符集：`utf8mb4`
- 自动填充字段（create_dept、create_by、create_time、update_by、update_time）由 MyBatis-Plus `MetaObjectHandler` 自动处理

---

## 四、常见模式

### 4.1 分页查询（项目标准模式）

```java
// Controller
@GetMapping("/list")
public TableDataInfo<{Vo}> list({Bo} bo, PageQuery pageQuery) {
    return service.selectPageList(bo, pageQuery);
}

// Service 实现
@Override
public TableDataInfo<{Vo}> selectPageList({Bo} bo, PageQuery pageQuery) {
    LambdaQueryWrapper<{Entity}> lqw = buildQueryWrapper(bo);
    Page<{Vo}> result = mapper.selectVoPage(pageQuery.build(), lqw);
    return TableDataInfo.build(result);
}

// 构建查询条件
private LambdaQueryWrapper<{Entity}> buildQueryWrapper({Bo} bo) {
    Map<String, Object> params = bo.getParams();
    LambdaQueryWrapper<{Entity}> lqw = Wrappers.lambdaQuery();
    // 精确匹配
    lqw.eq(StringUtils.isNotBlank(bo.getStatus()), {Entity}::getStatus, bo.getStatus());
    // 模糊匹配
    lqw.like(StringUtils.isNotBlank(bo.getName()), {Entity}::getName, bo.getName());
    // 时间范围（通过 params 传递 beginTime/endTime）
    lqw.between(params.get("beginTime") != null && params.get("endTime") != null,
        {Entity}::getCreateTime, params.get("beginTime"), params.get("endTime"));
    // 排序
    lqw.orderByDesc({Entity}::getCreateTime);
    return lqw;
}
```

**前端传参方式：**
```
GET /模块名/业务名/list?pageNum=1&pageSize=10&status=active&name=关键字&params[beginTime]=2024-01-01&params[endTime]=2024-12-31
```

### 4.2 Excel 导出

```java
// Controller
@Log(title = "{业务名}", businessType = BusinessType.EXPORT)
@SaCheckPermission("{模块名}:{业务名}:export")
@PostMapping("/export")
public void export({Bo} bo, HttpServletResponse response) {
    List<{Vo}> list = service.selectList(bo);  // 不分页，查全部
    ExcelUtil.exportExcel(list, "{业务中文名}数据", {Vo}.class, response);
}
```

**Vo 中的 Excel 注解：**
```java
@Data
@ExcelIgnoreUnannotated  // 只导出标注了 @ExcelProperty 的字段
@AutoMapper(target = {Entity}.class)
public class {Vo} implements Serializable {
    @ExcelProperty(value = "字段名")
    private String name;

    @ExcelProperty(value = "状态", converter = ExcelDictConvert.class)
    @ExcelDictFormat(dictType = "sys_normal_disable")
    private String status;
}
```

### 4.3 Excel 导入

```java
// Controller
@Log(title = "{业务名}", businessType = BusinessType.IMPORT)
@SaCheckPermission("{模块名}:{业务名}:import")
@PostMapping(value = "/importData", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public R<Void> importData(@RequestPart("file") MultipartFile file, boolean updateSupport) throws Exception {
    ExcelResult<{ImportVo}> result = ExcelUtil.importExcel(
        file.getInputStream(), {ImportVo}.class, new {ImportListener}(updateSupport));
    return R.ok(result.getAnalysis());
}

// 导入模板下载
@PostMapping("/importTemplate")
public void importTemplate(HttpServletResponse response) {
    ExcelUtil.exportExcel(new ArrayList<>(), "{业务中文名}数据", {ImportVo}.class, response);
}
```

### 4.4 文件下载

```java
// Controller
@SaCheckPermission("{模块名}:{业务名}:download")
@Log(title = "{业务名}", businessType = BusinessType.EXPORT)
@GetMapping("/download/{id}")
public void download(@NotNull(message = "ID不能为空") @PathVariable Long id, HttpServletResponse response) {
    {DetailVo} detail = service.downloadById(id);
    try {
        String fileName = detail.getName() + ".md";
        FileUtils.setAttachmentResponseHeader(response, fileName);
        response.setContentType("application/octet-stream");
        response.getWriter().write(detail.getContent());
        response.getWriter().flush();
    } catch (Exception e) {
        throw new RuntimeException("下载失败: " + e.getMessage());
    }
}
```

### 4.5 异步任务 + SSE 推送

```java
// Service
@Lazy
@Autowired
private I{Entity名}Service self;  // 自注入代理，确保 @Async 生效

public void executeTask(Long id) {
    self.executeAsync(id);  // 通过代理调用
}

@Async
@Override
public void executeAsync(Long id) {
    // 执行耗时操作...
    // 推送 SSE 通知
    SseMessageDto dto = new SseMessageDto();
    dto.setUserIds(List.of(userId));
    dto.setMessage("{\"type\":\"status\",\"status\":\"success\"}");
    SseMessageUtils.publishMessage(dto);
}
```

**关键点：**
- `@Async` 方法必须是 `public`，且通过代理调用（不能在同类中直接调用）
- 使用 `@Lazy @Autowired` 自注入解决同类调用问题

### 4.6 字典服务

```java
@RequiredArgsConstructor
@Service
public class {ServiceImpl} implements I{Service} {

    private final DictService dictService;

    public void someMethod() {
        // 获取字典类型的所有键值对
        Map<String, String> config = dictService.getAllDictByDictType("dict_type");
        String value = config.get("KEY");
    }
}
```

### 4.7 获取当前登录用户

```java
import org.dromara.common.satoken.utils.LoginHelper;

// 获取用户 ID
Long userId = LoginHelper.getUserId();

// 获取用户名
String username = LoginHelper.getUsername();

// 获取租户 ID
String tenantId = LoginHelper.getTenantId();
```

### 4.8 逻辑删除

Entity 中 `@TableLogic` 标注的字段（`del_flag`），MyBatis-Plus 会自动处理：
- 查询时自动加 `WHERE del_flag = '0'`
- 删除时自动执行 `UPDATE SET del_flag = '2'`（逻辑删除）
- `deleteByIds` 也是逻辑删除

### 4.9 字段自动填充

以下字段由 `BaseEntity` 定义，MyBatis-Plus 自动填充：
- `create_dept`：`@TableField(fill = FieldFill.INSERT)`
- `create_by`：`@TableField(fill = FieldFill.INSERT)`
- `create_time`：`@TableField(fill = FieldFill.INSERT)`
- `update_by`：`@TableField(fill = FieldFill.INSERT_UPDATE)`
- `update_time`：`@TableField(fill = FieldFill.INSERT_UPDATE)`

代码中无需手动设置这些字段。

---

## 五、自验清单

完成编码后，逐项检查：

### 5.1 代码结构
- [ ] Entity 继承 `TenantEntity`（多租户）或 `BaseEntity`（非多租户）
- [ ] Bo 继承 `BaseEntity`，增改 Bo 有校验注解
- [ ] Vo 实现 `Serializable`，有 `@AutoMapper` 注解
- [ ] Mapper 继承 `BaseMapperPlus<Entity, DefaultVo>`
- [ ] Mapper XML namespace 与 Mapper 接口一致
- [ ] Service 接口以 `I` 开头
- [ ] Controller 继承 `BaseController`

### 5.2 注解完整性
- [ ] Controller 有 `@Validated` + `@RequiredArgsConstructor`
- [ ] 每个接口有 `@SaCheckPermission`
- [ ] 写操作有 `@Log` + `@RepeatSubmit`
- [ ] Entity 有 `@TableName` + `@TableId` + `@TableLogic`

### 5.3 功能正确性
- [ ] 分页查询参数正确传递（Bo + PageQuery）
- [ ] 查询条件构建正确（eq/like/between）
- [ ] 详情查询指定正确的 Vo 类型
- [ ] 新增/修改有唯一性校验（如需要）
- [ ] 删除支持批量
- [ ] 时间范围查询通过 `params` 传递 `beginTime`/`endTime`

### 5.4 安全性
- [ ] 无硬编码密钥
- [ ] SQL 无注入风险（使用 LambdaQueryWrapper）
- [ ] 接口权限配置正确
- [ ] 敏感数据不返回前端

### 5.5 编码规范
- [ ] import 路径正确（`org.dromara.xxx`）
- [ ] 注释完整（类注释 + 方法注释 + 字段注释）
- [ ] 无未使用的 import
- [ ] 日期类型统一用 `java.util.Date`
- [ ] 集合类型用 `Collection<Long>`（Service 接口）/ `List<Long>`（Controller）

---

## 六、经验库写入规范

在开发过程中遇到问题时，将解决方案写入经验库，格式如下：

```markdown
## [日期] 问题描述

### 问题
{简述问题现象}

### 原因
{根本原因分析}

### 解决方案
{具体解决步骤}

### 预防
{如何避免再次发生}
```

**写入位置：** `{DOC_DIR}/经验库/{模块名}.md`

---

## 附录：关键 import 速查

```java
// ===== Entity =====
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import com.baomidou.mybatisplus.annotation.TableLogic;
import org.dromara.common.tenant.core.TenantEntity;
import org.dromara.common.mybatis.core.domain.BaseEntity;

// ===== Bo =====
import io.github.linpeilie.annotations.AutoMapper;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

// ===== Vo =====
import io.github.linpeilie.annotations.AutoMapper;
import cn.idev.excel.annotation.ExcelIgnoreUnannotated;
import cn.idev.excel.annotation.ExcelProperty;
import org.dromara.common.excel.annotation.ExcelDictFormat;
import org.dromara.common.excel.convert.ExcelDictConvert;
import org.dromara.common.translation.annotation.Translation;
import org.dromara.common.translation.constant.TransConstant;

// ===== Mapper =====
import org.dromara.common.mybatis.core.mapper.BaseMapperPlus;

// ===== Service =====
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import org.dromara.common.core.exception.ServiceException;
import org.dromara.common.core.utils.StringUtils;

// ===== Controller =====
import cn.dev33.satoken.annotation.SaCheckPermission;
import org.dromara.common.core.domain.R;
import org.dromara.common.log.annotation.Log;
import org.dromara.common.log.enums.BusinessType;
import org.dromara.common.idempotent.annotation.RepeatSubmit;
import org.dromara.common.excel.utils.ExcelUtil;
import org.dromara.common.web.core.BaseController;

// ===== 其他工具 =====
import cn.hutool.core.util.StrUtil;
import org.dromara.common.satoken.utils.LoginHelper;
import org.dromara.common.core.service.DictService;
```
