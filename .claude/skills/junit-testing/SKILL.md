---
name: junit-testing
description: |
  JUnit 5 + Mockito + Spring Boot Test 后端单元/集成测试。给测试工程师Agent使用，当需要编写后端Service、Controller、Mapper层的单元测试或集成测试时触发。覆盖RuoYi-Vue-Plus项目的特有模式（R<T>响应、TableDataInfo分页、Sa-Token权限、多租户隔离等）。
---

# JUnit 5 + Mockito + Spring Boot Test 后端单元/集成测试

> 适配项目：RuoYi-Vue-Plus 5.6.0
> 技术栈：Spring Boot 3.5.12 + Java 17 + MyBatis-Plus 3.5.16 + Sa-Token 1.44.0 + Redis + 多租户

**定位：** 本Skill专注后端测试的**具体写法**（怎么写代码），与 `test-automation` Skill 的流水线编排互补。

---

## 一、测试框架配置

### 1.1 pom.xml 测试依赖

在需要测试的模块 `pom.xml` 中确保有以下依赖（`spring-boot-starter-test` 通常已在父POM中管理）：

```xml
<!-- 单元测试（通常已在 ruoyi-admin 的 pom.xml 中存在） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- H2 内存数据库（用于Mapper层测试，按需添加） -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

> `spring-boot-starter-test` 已包含：JUnit 5、Mockito、AssertJ、Hamcrest、Spring Test、JSONPath 等。

### 1.2 测试配置文件

在 `src/test/resources/` 下创建 `application-test.yml`：

```yaml
# application-test.yml - 测试专用配置
spring:
  datasource:
    # H2 内存数据库（Mapper测试用）
    url: jdbc:h2:mem:testdb;MODE=MYSQL;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  data:
    redis:
      host: localhost
      port: 6379
  sql:
    init:
      mode: never

# MyBatis-Plus 配置
mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl

# Sa-Token 配置
sa-token:
  token-name: Authorization
  timeout: 86400
  is-concurrent: true
  is-share: true
  token-style: uuid
  is-log: false
```

### 1.3 测试目录结构

```
ruoyi-modules/ruoyi-{module}/src/test/java/org/dromara/{module}/
├── service/
│   ├── SysNoticeServiceImplTest.java       # Service层单元测试
│   └── SysPostServiceImplTest.java         # Service层单元测试
├── controller/
│   └── SysNoticeControllerTest.java        # Controller层单元测试
├── mapper/
│   └── SysNoticeMapperTest.java            # Mapper层测试
└── integration/
    └── SysNoticeIntegrationTest.java       # 集成测试
```

---

## 二、Service层单元测试

### 2.1 完整模板（基于 SysNoticeService）

```java
package org.dromara.system.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyList;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.system.domain.SysNotice;
import org.dromara.system.domain.bo.SysNoticeBo;
import org.dromara.system.domain.vo.SysNoticeVo;
import org.dromara.system.domain.vo.SysUserVo;
import org.dromara.system.mapper.SysNoticeMapper;
import org.dromara.system.mapper.SysUserMapper;
import org.dromara.system.service.impl.SysNoticeServiceImpl;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
@DisplayName("SysNoticeService 单元测试")
class SysNoticeServiceImplTest {

    @Mock
    private SysNoticeMapper baseMapper;

    @Mock
    private SysUserMapper userMapper;

    @InjectMocks
    private SysNoticeServiceImpl noticeService;

    private SysNoticeVo testVo;
    private SysNoticeBo testBo;
    private PageQuery pageQuery;

    @BeforeEach
    void setUp() {
        testVo = new SysNoticeVo();
        testVo.setNoticeId(1L);
        testVo.setNoticeTitle("测试公告标题");
        testVo.setNoticeType("1");
        testVo.setNoticeContent("测试公告内容");
        testVo.setStatus("0");
        testVo.setCreateBy(1L);
        testVo.setCreateByName("admin");

        testBo = new SysNoticeBo();
        testBo.setNoticeId(1L);
        testBo.setNoticeTitle("测试公告标题");
        testBo.setNoticeType("1");
        testBo.setNoticeContent("测试公告内容");
        testBo.setStatus("0");

        pageQuery = new PageQuery();
        pageQuery.setPageNum(1);
        pageQuery.setPageSize(10);
    }

    // ==================== 查询测试 ====================

    @Nested
    @DisplayName("查询相关测试")
    class QueryTests {

        @Test
        @DisplayName("根据ID查询公告 - 存在时返回VO")
        void selectNoticeById_shouldReturnVoWhenExists() {
            when(baseMapper.selectVoById(1L)).thenReturn(testVo);
            SysNoticeVo result = noticeService.selectNoticeById(1L);
            assertThat(result).isNotNull();
            assertThat(result.getNoticeId()).isEqualTo(1L);
            assertThat(result.getNoticeTitle()).isEqualTo("测试公告标题");
            verify(baseMapper).selectVoById(1L);
        }

        @Test
        @DisplayName("根据ID查询公告 - 不存在时返回null")
        void selectNoticeById_shouldReturnNullWhenNotExists() {
            when(baseMapper.selectVoById(999L)).thenReturn(null);
            SysNoticeVo result = noticeService.selectNoticeById(999L);
            assertThat(result).isNull();
        }

        @Test
        @DisplayName("分页查询公告列表 - 有数据")
        void selectPageNoticeList_shouldReturnPageWhenHasData() {
            Page<SysNoticeVo> mockPage = new Page<>(1, 10);
            mockPage.setRecords(List.of(testVo));
            mockPage.setTotal(1);
            when(baseMapper.selectVoPage(any(Page.class), any(LambdaQueryWrapper.class)))
                .thenReturn(mockPage);

            TableDataInfo<SysNoticeVo> result = noticeService.selectPageNoticeList(testBo, pageQuery);

            assertThat(result).isNotNull();
            assertThat(result.getTotal()).isEqualTo(1);
            assertThat(result.getRows()).hasSize(1);
            assertThat(result.getRows().get(0).getNoticeTitle()).isEqualTo("测试公告标题");
        }

        @Test
        @DisplayName("分页查询公告列表 - 无数据")
        void selectPageNoticeList_shouldReturnEmptyPageWhenNoData() {
            Page<SysNoticeVo> mockPage = new Page<>(1, 10);
            mockPage.setRecords(Collections.emptyList());
            mockPage.setTotal(0);
            when(baseMapper.selectVoPage(any(Page.class), any(LambdaQueryWrapper.class)))
                .thenReturn(mockPage);

            TableDataInfo<SysNoticeVo> result = noticeService.selectPageNoticeList(testBo, pageQuery);
            assertThat(result.getTotal()).isEqualTo(0);
            assertThat(result.getRows()).isEmpty();
        }

        @Test
        @DisplayName("按创建人名称查询 - 关联用户表")
        void selectPageNoticeList_shouldJoinUserWhenCreateByNameProvided() {
            testBo.setCreateByName("admin");
            SysUserVo userVo = new SysUserVo();
            userVo.setUserId(1L);
            when(userMapper.selectVoOne(any(LambdaQueryWrapper.class))).thenReturn(userVo);

            Page<SysNoticeVo> mockPage = new Page<>(1, 10);
            mockPage.setRecords(Collections.emptyList());
            mockPage.setTotal(0);
            when(baseMapper.selectVoPage(any(Page.class), any(LambdaQueryWrapper.class)))
                .thenReturn(mockPage);

            noticeService.selectPageNoticeList(testBo, pageQuery);
            verify(userMapper).selectVoOne(any(LambdaQueryWrapper.class));
        }

        @Test
        @DisplayName("查询公告列表 - 不分页")
        void selectNoticeList_shouldReturnList() {
            when(baseMapper.selectVoList(any(LambdaQueryWrapper.class)))
                .thenReturn(List.of(testVo));
            List<SysNoticeVo> result = noticeService.selectNoticeList(testBo);
            assertThat(result).hasSize(1);
        }
    }

    // ==================== 新增测试 ====================

    @Nested
    @DisplayName("新增相关测试")
    class InsertTests {

        @Test
        @DisplayName("新增公告 - 成功")
        void insertNotice_shouldReturnOneWhenSuccess() {
            when(baseMapper.insert(any(SysNotice.class))).thenReturn(1);
            int result = noticeService.insertNotice(testBo);
            assertThat(result).isEqualTo(1);
            verify(baseMapper).insert(any(SysNotice.class));
        }

        @Test
        @DisplayName("新增公告 - 数据库插入失败返回0")
        void insertNotice_shouldReturnZeroWhenDbError() {
            when(baseMapper.insert(any(SysNotice.class))).thenReturn(0);
            int result = noticeService.insertNotice(testBo);
            assertThat(result).isEqualTo(0);
        }
    }

    // ==================== 更新测试 ====================

    @Nested
    @DisplayName("更新相关测试")
    class UpdateTests {

        @Test
        @DisplayName("更新公告 - 成功")
        void updateNotice_shouldReturnOneWhenSuccess() {
            when(baseMapper.updateById(any(SysNotice.class))).thenReturn(1);
            int result = noticeService.updateNotice(testBo);
            assertThat(result).isEqualTo(1);
        }

        @Test
        @DisplayName("更新公告 - 记录不存在返回0")
        void updateNotice_shouldReturnZeroWhenNotFound() {
            when(baseMapper.updateById(any(SysNotice.class))).thenReturn(0);
            int result = noticeService.updateNotice(testBo);
            assertThat(result).isEqualTo(0);
        }
    }

    // ==================== 删除测试 ====================

    @Nested
    @DisplayName("删除相关测试")
    class DeleteTests {

        @Test
        @DisplayName("单个删除 - 成功")
        void deleteNoticeById_shouldReturnOneWhenSuccess() {
            when(baseMapper.deleteById(1L)).thenReturn(1);
            int result = noticeService.deleteNoticeById(1L);
            assertThat(result).isEqualTo(1);
        }

        @Test
        @DisplayName("批量删除 - 成功")
        void deleteNoticeByIds_shouldReturnCountWhenSuccess() {
            Long[] ids = {1L, 2L, 3L};
            when(baseMapper.deleteByIds(Arrays.asList(ids))).thenReturn(3);
            int result = noticeService.deleteNoticeByIds(ids);
            assertThat(result).isEqualTo(3);
        }
    }
}
```

### 2.2 通用 Service 测试模板（适用于其他模块）

将 `{Feature}` / `{feature}` / `{module}` 替换为实际模块名即可：

```java
package org.dromara.{module}.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

import org.dromara.{module}.domain.{Feature};
import org.dromara.{module}.domain.bo.{Feature}Bo;
import org.dromara.{module}.domain.vo.{Feature}Vo;
import org.dromara.{module}.mapper.{Feature}Mapper;
import org.dromara.{module}.service.impl.{Feature}ServiceImpl;
import org.dromara.common.core.exception.ServiceException;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
@DisplayName("{Feature}Service 单元测试")
class {Feature}ServiceImplTest {

    @Mock
    private {Feature}Mapper baseMapper;

    @InjectMocks
    private {Feature}ServiceImpl {feature}Service;

    private {Feature}Vo testVo;
    private {Feature}Bo testBo;

    @BeforeEach
    void setUp() {
        testVo = new {Feature}Vo();
        testVo.setId(1L);
        testBo = new {Feature}Bo();
        testBo.setName("测试数据");
    }

    @Nested @DisplayName("查询测试") class QueryTests {
        @Test @DisplayName("根据ID查询 - 存在")
        void selectById_shouldReturnVo() {
            when(baseMapper.selectVoById(1L, {Feature}Vo.class)).thenReturn(testVo);
            {Feature}Vo result = {feature}Service.selectDetailById(1L);
            assertThat(result).isNotNull();
            assertThat(result.getId()).isEqualTo(1L);
        }

        @Test @DisplayName("根据ID查询 - 不存在抛异常")
        void selectById_shouldThrowWhenNotFound() {
            when(baseMapper.selectVoById(999L, {Feature}Vo.class)).thenReturn(null);
            assertThatThrownBy(() -> {feature}Service.selectDetailById(999L))
                .isInstanceOf(ServiceException.class);
        }
    }

    @Nested @DisplayName("新增测试") class InsertTests {
        @Test @DisplayName("新增 - 成功")
        void insert_shouldReturnTrue() {
            when(baseMapper.insert(any({Feature}.class))).thenReturn(1);
            Boolean result = {feature}Service.insertByBo(testBo);
            assertThat(result).isTrue();
        }
    }

    @Nested @DisplayName("更新测试") class UpdateTests {
        @Test @DisplayName("更新 - 成功")
        void update_shouldReturnTrue() {
            testBo.setId(1L);
            when(baseMapper.updateById(any({Feature}.class))).thenReturn(1);
            Boolean result = {feature}Service.updateByBo(testBo);
            assertThat(result).isTrue();
        }
    }

    @Nested @DisplayName("删除测试") class DeleteTests {
        @Test @DisplayName("删除 - 成功")
        void delete_shouldReturnTrue() {
            when(baseMapper.deleteByIds(anyList())).thenReturn(1);
            Boolean result = {feature}Service.deleteWithValidByIds(new Long[]{1L});
            assertThat(result).isTrue();
        }

        @Test @DisplayName("删除 - ID为null不执行")
        void delete_shouldNotExecuteWhenIdNull() {
            {feature}Service.deleteWithValidByIds(new Long[]{null});
            verify(baseMapper, never()).deleteByIds(anyList());
        }
    }
}
```

---

## 三、Controller层单元测试

### 3.1 完整模板（基于 SysNoticeController）

```java
package org.dromara.system.controller.system;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Collections;
import java.util.List;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.system.domain.bo.SysNoticeBo;
import org.dromara.system.domain.vo.SysNoticeVo;
import org.dromara.system.service.ISysNoticeService;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.bean.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

@WebMvcTest(SysNoticeController.class)
@DisplayName("SysNoticeController 单元测试")
class SysNoticeControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private ISysNoticeService noticeService;

    // 如有其他依赖也需 @MockBean：
    // @MockBean private DictService dictService;

    private SysNoticeVo buildTestVo() {
        SysNoticeVo vo = new SysNoticeVo();
        vo.setNoticeId(1L);
        vo.setNoticeTitle("测试公告");
        vo.setNoticeType("1");
        vo.setStatus("0");
        return vo;
    }

    private SysNoticeBo buildTestBo() {
        SysNoticeBo bo = new SysNoticeBo();
        bo.setNoticeTitle("测试公告");
        bo.setNoticeType("1");
        bo.setNoticeContent("测试内容");
        bo.setStatus("0");
        return bo;
    }

    // ==================== 列表查询 ====================

    @Nested
    @DisplayName("GET /system/notice/list - 列表查询")
    class ListTests {

        @Test
        @DisplayName("分页查询 - 返回列表数据")
        void list_shouldReturnPageData() throws Exception {
            Page<SysNoticeVo> mockPage = new Page<>(1, 10);
            mockPage.setRecords(List.of(buildTestVo()));
            mockPage.setTotal(1);
            TableDataInfo<SysNoticeVo> tableData = TableDataInfo.build(mockPage);
            when(noticeService.selectPageNoticeList(any(), any(PageQuery.class)))
                .thenReturn(tableData);

            mockMvc.perform(get("/system/notice/list")
                    .param("pageNum", "1").param("pageSize", "10"))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.total").value(1))
                .andExpect(jsonPath("$.rows").isArray())
                .andExpect(jsonPath("$.rows[0].noticeTitle").value("测试公告"));
        }

        @Test
        @DisplayName("带条件查询 - 按公告类型筛选")
        void list_shouldFilterByType() throws Exception {
            Page<SysNoticeVo> mockPage = new Page<>(1, 10);
            mockPage.setRecords(Collections.emptyList());
            mockPage.setTotal(0);
            when(noticeService.selectPageNoticeList(any(), any(PageQuery.class)))
                .thenReturn(TableDataInfo.build(mockPage));

            mockMvc.perform(get("/system/notice/list")
                    .param("noticeType", "1").param("pageNum", "1").param("pageSize", "10"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.total").value(0));
        }
    }

    // ==================== 详情查询 ====================

    @Nested
    @DisplayName("GET /system/notice/{noticeId} - 详情查询")
    class DetailTests {

        @Test
        @DisplayName("查询存在的公告 - 返回详情")
        void getInfo_shouldReturnDetail() throws Exception {
            when(noticeService.selectNoticeById(1L)).thenReturn(buildTestVo());

            mockMvc.perform(get("/system/notice/1"))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.data.noticeId").value(1))
                .andExpect(jsonPath("$.data.noticeTitle").value("测试公告"));
        }
    }

    // ==================== 新增 ====================

    @Nested
    @DisplayName("POST /system/notice - 新增")
    class AddTests {

        @Test
        @DisplayName("新增公告 - 参数合法返回成功")
        void add_shouldReturnOk() throws Exception {
            when(noticeService.insertNotice(any())).thenReturn(1);

            mockMvc.perform(post("/system/notice")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(buildTestBo())))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200));
        }

        @Test
        @DisplayName("新增公告 - 标题为空返回400")
        void add_shouldReturn400WhenTitleBlank() throws Exception {
            SysNoticeBo bo = new SysNoticeBo();
            bo.setNoticeTitle("");  // 违反 @NotBlank
            bo.setNoticeType("1");

            mockMvc.perform(post("/system/notice")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(bo)))
                .andDo(print())
                .andExpect(status().isBadRequest());
        }

        @Test
        @DisplayName("新增公告 - 标题超过50字符返回400")
        void add_shouldReturn400WhenTitleTooLong() throws Exception {
            SysNoticeBo bo = new SysNoticeBo();
            bo.setNoticeTitle("a".repeat(51));  // 违反 @Size(max=50)
            bo.setNoticeType("1");

            mockMvc.perform(post("/system/notice")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(bo)))
                .andDo(print())
                .andExpect(status().isBadRequest());
        }
    }

    // ==================== 更新 ====================

    @Nested
    @DisplayName("PUT /system/notice - 更新")
    class EditTests {

        @Test
        @DisplayName("更新公告 - 成功")
        void edit_shouldReturnOk() throws Exception {
            SysNoticeBo bo = buildTestBo();
            bo.setNoticeId(1L);
            when(noticeService.updateNotice(any())).thenReturn(1);

            mockMvc.perform(put("/system/notice")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(bo)))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200));
        }
    }

    // ==================== 删除 ====================

    @Nested
    @DisplayName("DELETE /system/notice/{noticeIds} - 删除")
    class RemoveTests {

        @Test
        @DisplayName("删除公告 - 成功")
        void remove_shouldReturnOk() throws Exception {
            when(noticeService.deleteNoticeByIds(any(Long[].class))).thenReturn(1);

            mockMvc.perform(delete("/system/notice/1,2,3"))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200));
        }
    }
}
```

> **注意：** `@WebMvcTest` 只加载 Controller 层的 Spring 上下文。Controller 依赖的其他 Bean 需用 `@MockBean` 声明。

---

## 四、Mapper层测试

### 4.1 完整模板（H2 内存数据库）

```java
package org.dromara.system.mapper;

import static org.assertj.core.api.Assertions.assertThat;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import java.util.List;
import org.dromara.system.domain.SysNotice;
import org.dromara.system.domain.vo.SysNoticeVo;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.transaction.annotation.Transactional;

@SpringBootTest
@ActiveProfiles("test")
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Transactional  // 每个测试方法执行后自动回滚
@DisplayName("SysNoticeMapper 测试")
class SysNoticeMapperTest {

    @Autowired
    private SysNoticeMapper noticeMapper;

    private SysNotice buildNotice(Long id, String title, String type) {
        SysNotice notice = new SysNotice();
        notice.setNoticeId(id);
        notice.setNoticeTitle(title);
        notice.setNoticeType(type);
        notice.setNoticeContent("内容_" + id);
        notice.setStatus("0");
        notice.setTenantId("000000");
        return notice;
    }

    @Nested
    @DisplayName("CRUD操作")
    class CrudTests {

        @Test
        @DisplayName("插入记录")
        void insert_shouldPersist() {
            SysNotice notice = buildNotice(null, "Mapper测试", "1");
            int rows = noticeMapper.insert(notice);
            assertThat(rows).isEqualTo(1);
            assertThat(notice.getNoticeId()).isNotNull();
        }

        @Test
        @DisplayName("根据ID查询")
        void selectById_shouldReturn() {
            SysNotice notice = buildNotice(null, "查询测试", "1");
            noticeMapper.insert(notice);

            SysNoticeVo vo = noticeMapper.selectVoById(notice.getNoticeId());
            assertThat(vo).isNotNull();
            assertThat(vo.getNoticeTitle()).isEqualTo("查询测试");
        }

        @Test
        @DisplayName("更新记录")
        void updateById_shouldModify() {
            SysNotice notice = buildNotice(null, "原标题", "1");
            noticeMapper.insert(notice);

            notice.setNoticeTitle("更新后标题");
            int rows = noticeMapper.updateById(notice);
            assertThat(rows).isEqualTo(1);

            SysNoticeVo updated = noticeMapper.selectVoById(notice.getNoticeId());
            assertThat(updated.getNoticeTitle()).isEqualTo("更新后标题");
        }

        @Test
        @DisplayName("删除记录")
        void deleteById_shouldRemove() {
            SysNotice notice = buildNotice(null, "待删除", "1");
            noticeMapper.insert(notice);
            Long id = notice.getNoticeId();

            int rows = noticeMapper.deleteById(id);
            assertThat(rows).isEqualTo(1);
            assertThat(noticeMapper.selectVoById(id)).isNull();
        }

        @Test
        @DisplayName("批量删除")
        void deleteByIds_shouldRemoveMultiple() {
            SysNotice n1 = buildNotice(null, "批量1", "1");
            SysNotice n2 = buildNotice(null, "批量2", "2");
            noticeMapper.insert(n1);
            noticeMapper.insert(n2);

            int rows = noticeMapper.deleteByIds(List.of(n1.getNoticeId(), n2.getNoticeId()));
            assertThat(rows).isEqualTo(2);
        }
    }

    @Nested
    @DisplayName("分页查询")
    class PageTests {

        @Test
        @DisplayName("分页查询 - 有数据")
        void selectVoPage_shouldReturnPaged() {
            for (int i = 0; i < 15; i++) {
                noticeMapper.insert(buildNotice(null, "分页" + i, "1"));
            }

            Page<SysNoticeVo> page = new Page<>(1, 10);
            Page<SysNoticeVo> result = noticeMapper.selectVoPage(page, new LambdaQueryWrapper<>());
            assertThat(result.getRecords()).hasSize(10);
            assertThat(result.getTotal()).isGreaterThanOrEqualTo(15);
        }

        @Test
        @DisplayName("分页查询 - 空结果")
        void selectVoPage_shouldReturnEmpty() {
            Page<SysNoticeVo> page = new Page<>(1, 10);
            Page<SysNoticeVo> result = noticeMapper.selectVoPage(page, new LambdaQueryWrapper<>());
            assertThat(result.getRecords()).isEmpty();
        }
    }

    @Nested
    @DisplayName("条件查询")
    class ConditionTests {

        @Test
        @DisplayName("按类型查询")
        void selectVoList_shouldFilterByType() {
            noticeMapper.insert(buildNotice(null, "通知", "1"));
            noticeMapper.insert(buildNotice(null, "公告", "2"));

            LambdaQueryWrapper<SysNotice> wrapper = new LambdaQueryWrapper<>();
            wrapper.eq(SysNotice::getNoticeType, "1");
            List<SysNoticeVo> result = noticeMapper.selectVoList(wrapper);
            assertThat(result).hasSize(1);
            assertThat(result.get(0).getNoticeType()).isEqualTo("1");
        }

        @Test
        @DisplayName("模糊查询标题")
        void selectVoList_shouldSearchByTitle() {
            noticeMapper.insert(buildNotice(null, "系统维护通知", "1"));
            noticeMapper.insert(buildNotice(null, "版本发布公告", "2"));

            LambdaQueryWrapper<SysNotice> wrapper = new LambdaQueryWrapper<>();
            wrapper.like(SysNotice::getNoticeTitle, "通知");
            List<SysNoticeVo> result = noticeMapper.selectVoList(wrapper);
            assertThat(result).hasSize(1);
        }
    }
}
```

> **注意：** Mapper层测试使用真实数据库连接。如果项目没有启动Redis等基础设施，建议用 `@MockBean` 替换或使用 H2 内存数据库配合 `@AutoConfigureTestDatabase`。

---

## 五、集成测试

### 5.1 完整模板（SpringBootTest + Sa-Token登录态模拟）

```java
package org.dromara.system.integration;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

import cn.dev33.satoken.stp.StpUtil;
import org.dromara.common.core.domain.R;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.system.domain.bo.SysNoticeBo;
import org.dromara.system.domain.vo.SysNoticeVo;
import org.dromara.system.service.ISysNoticeService;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import com.fasterxml.jackson.databind.ObjectMapper;

@SpringBootTest
@ActiveProfiles("test")
@AutoConfigureMockMvc
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@DisplayName("SysNotice 集成测试")
class SysNoticeIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private ISysNoticeService noticeService;

    private static Long createdNoticeId;

    /**
     * 模拟管理员登录，获取 Sa-Token
     */
    private String loginAsAdmin() {
        // 方式1：直接通过 StpUtil 创建登录态（绕过密码验证）
        StpUtil.login(1L);
        return StpUtil.getTokenValue();

        // 方式2：通过 MockMvc 调用登录接口（测试完整登录流程）
        // return mockMvc.perform(post("/auth/login")
        //         .contentType(MediaType.APPLICATION_JSON)
        //         .content("{\"username\":\"admin\",\"password\":\"admin123\"}"))
        //     .andExpect(status().isOk())
        //     .andReturn().getResponse().getHeader("Authorization");
    }

    /**
     * 构建带登录态的请求
     */
    private MockHttpServletRequestBuilder withAuth(MockHttpServletRequestBuilder req) {
        return req.header("Authorization", "Bearer " + loginAsAdmin());
    }

    // ==================== CRUD 全流程 ====================

    @Test
    @Order(1)
    @DisplayName("1. 新增公告")
    void createNotice_shouldReturnSuccess() throws Exception {
        SysNoticeBo bo = new SysNoticeBo();
        bo.setNoticeTitle("集成测试公告");
        bo.setNoticeType("1");
        bo.setNoticeContent("集成测试内容");
        bo.setStatus("0");

        String response = mockMvc.perform(withAuth(post("/system/notice"))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(bo)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(200))
            .andReturn().getResponse().getContentAsString();

        // 解析响应，记录创建的ID（如果接口返回）
        // createdNoticeId = JsonPath.parse(response).read("$.data", Long.class);
    }

    @Test
    @Order(2)
    @DisplayName("2. 查询公告列表")
    void listNotice_shouldReturnData() throws Exception {
        mockMvc.perform(withAuth(get("/system/notice/list")
                .param("pageNum", "1").param("pageSize", "10")))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(200))
            .andExpect(jsonPath("$.rows").isArray());
    }

    @Test
    @Order(3)
    @DisplayName("3. 查询公告详情")
    void getNotice_shouldReturnDetail() throws Exception {
        mockMvc.perform(withAuth(get("/system/notice/1")))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(200));
    }

    @Test
    @Order(4)
    @DisplayName("4. 更新公告")
    void updateNotice_shouldReturnSuccess() throws Exception {
        SysNoticeBo bo = new SysNoticeBo();
        bo.setNoticeId(1L);
        bo.setNoticeTitle("更新后的集成测试公告");
        bo.setNoticeType("1");
        bo.setNoticeContent("更新后内容");
        bo.setStatus("0");

        mockMvc.perform(withAuth(put("/system/notice"))
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(bo)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(200));
    }

    @Test
    @Order(5)
    @DisplayName("5. 删除公告")
    void deleteNotice_shouldReturnSuccess() throws Exception {
        mockMvc.perform(withAuth(delete("/system/notice/1")))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(200));
    }

    // ==================== 权限测试 ====================

    @Test
    @Order(10)
    @DisplayName("未登录访问 - 应返回401")
    void accessWithoutLogin_shouldReturn401() throws Exception {
        mockMvc.perform(get("/system/notice/list")
                .param("pageNum", "1").param("pageSize", "10"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @Order(11)
    @DisplayName("无权限访问 - 应返回403")
    void accessWithoutPermission_shouldReturn403() throws Exception {
        // 模拟一个没有 notice:list 权限的用户
        StpUtil.login(999L);
        String token = StpUtil.getTokenValue();

        mockMvc.perform(get("/system/notice/list")
                .param("pageNum", "1").param("pageSize", "10")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isForbidden());
    }
}
```

### 5.2 集成测试基类（复用登录态）

```java
package org.dromara.system.integration;

import cn.dev33.satoken.stp.StpUtil;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;

@SpringBootTest
@ActiveProfiles("test")
@AutoConfigureMockMvc
public abstract class BaseIntegrationTest {

    @Autowired
    protected MockMvc mockMvc;

    protected String adminToken;
    protected String userToken;

    @BeforeEach
    void setUpAuth() {
        // 管理员登录
        StpUtil.login(1L);
        adminToken = StpUtil.getTokenValue();
        StpUtil.logout();

        // 普通用户登录
        StpUtil.login(2L);
        userToken = StpUtil.getTokenValue();
        StpUtil.logout();
    }

    protected String authHeader(String token) {
        return "Bearer " + token;
    }
}
```

---

## 六、测试数据构建

### 6.1 测试数据工厂类

```java
package org.dromara.system.test;

import org.dromara.system.domain.SysNotice;
import org.dromara.system.domain.bo.SysNoticeBo;
import org.dromara.system.domain.vo.SysNoticeVo;

/**
 * 测试数据工厂 - 集中管理测试数据的创建
 */
public final class NoticeTestDataFactory {

    private NoticeTestDataFactory() {}

    /** 构建默认的Vo */
    public static SysNoticeVo buildDefaultVo() {
        SysNoticeVo vo = new SysNoticeVo();
        vo.setNoticeId(1L);
        vo.setNoticeTitle("默认测试公告");
        vo.setNoticeType("1");
        vo.setNoticeContent("默认测试内容");
        vo.setStatus("0");
        return vo;
    }

    /** 构建默认的Bo */
    public static SysNoticeBo buildDefaultBo() {
        SysNoticeBo bo = new SysNoticeBo();
        bo.setNoticeTitle("默认测试公告");
        bo.setNoticeType("1");
        bo.setNoticeContent("默认测试内容");
        bo.setStatus("0");
        return bo;
    }

    /** 构建指定ID的实体 */
    public static SysNotice buildEntity(Long id, String title, String type) {
        SysNotice entity = new SysNotice();
        entity.setNoticeId(id);
        entity.setNoticeTitle(title);
        entity.setNoticeType(type);
        entity.setNoticeContent("内容_" + title);
        entity.setStatus("0");
        entity.setTenantId("000000");
        return entity;
    }

    /** 构建指定数量的Vo列表 */
    public static java.util.List<SysNoticeVo> buildVoList(int count) {
        java.util.ArrayList<SysNoticeVo> list = new java.util.ArrayList<>();
        for (int i = 1; i <= count; i++) {
            SysNoticeVo vo = buildDefaultVo();
            vo.setNoticeId((long) i);
            vo.setNoticeTitle("测试公告_" + i);
            list.add(vo);
        }
        return list;
    }
}
```

### 6.2 参数化测试

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;
import java.util.stream.Stream;

@Nested
@DisplayName("参数化测试")
class ParametrizedTests {

    @ParameterizedTest(name = "公告类型={0} 应该有效")
    @ValueSource(strings = {"1", "2"})
    @DisplayName("有效公告类型验证")
    void noticeType_shouldBeValid(String type) {
        SysNoticeBo bo = NoticeTestDataFactory.buildDefaultBo();
        bo.setNoticeType(type);
        // 执行测试...
    }

    @ParameterizedTest(name = "空标题 [{0}] 应该抛出异常")
    @NullAndEmptySource
    @DisplayName("空标题校验")
    void noticeTitle_shouldRejectBlank(String title) {
        assertThatThrownBy(() -> {
            SysNoticeBo bo = NoticeTestDataFactory.buildDefaultBo();
            bo.setNoticeTitle(title);
            noticeService.insertNotice(bo);
        }).isInstanceOf(ServiceException.class);
    }

    @ParameterizedTest(name = "标题长度={0} 应该{1}")
    @CsvSource({
        "5,   通过",
        "50,  通过",
        "51,  失败",
        "100, 失败"
    })
    @DisplayName("标题长度校验")
    void noticeTitleLength_shouldValidate(int length, String expected) {
        SysNoticeBo bo = NoticeTestDataFactory.buildDefaultBo();
        bo.setNoticeTitle("a".repeat(length));
        if ("失败".equals(expected)) {
            assertThatThrownBy(() -> noticeService.insertNotice(bo))
                .isInstanceOf(ServiceException.class);
        }
    }

    @ParameterizedTest(name = "{0}")
    @MethodSource("invalidNoticeProvider")
    @DisplayName("无效公告数据校验")
    void invalidNotice_shouldReject(String desc, String title, String type) {
        SysNoticeBo bo = NoticeTestDataFactory.buildDefaultBo();
        bo.setNoticeTitle(title);
        bo.setNoticeType(type);
        assertThatThrownBy(() -> noticeService.insertNotice(bo))
            .isInstanceOf(ServiceException.class);
    }

    static Stream<Arguments> invalidNoticeProvider() {
        return Stream.of(
            Arguments.of("标题为null", null, "1"),
            Arguments.of("类型为null", "标题", null),
            Arguments.of("类型无效", "标题", "99")
        );
    }
}
```

### 6.3 通用测试数据Builder

```java
package org.dromara.common.test;

/**
 * 通用测试数据Builder - 链式构建测试数据
 */
public class TestEntityBuilder<T> {

    private final T entity;
    private final java.util.function.Consumer<T> modifier;

    private TestEntityBuilder(T entity, java.util.function.Consumer<T> modifier) {
        this.entity = entity;
        this.modifier = modifier;
    }

    public static <T> TestEntityBuilder<T> of(Class<T> clazz) {
        try {
            return new TestEntityBuilder<>(clazz.getDeclaredConstructor().newInstance(), e -> {});
        } catch (Exception e) {
            throw new RuntimeException("无法创建实例: " + clazz.getName(), e);
        }
    }

    public TestEntityBuilder<T> with(java.util.function.Consumer<T> setter) {
        setter.accept(entity);
        return this;
    }

    public T build() {
        return entity;
    }
}

// 使用示例：
// SysNoticeVo vo = TestEntityBuilder.of(SysNoticeVo.class)
//     .with(v -> v.setNoticeId(1L))
//     .with(v -> v.setNoticeTitle("测试"))
//     .build();
```

---

## 七、断言最佳实践（AssertJ）

### 7.1 基础断言

```java
import static org.assertj.core.api.Assertions.*;

// 对象断言
assertThat(result).isNotNull();
assertThat(result).isNull();
assertThat(result).isEqualTo(expected);
assertThat(result).isNotEqualTo(other);
assertThat(result).isSameAs(expected);

// 字符串断言
assertThat(name).isNotBlank();
assertThat(name).isEqualTo("expected");
assertThat(name).startsWith("prefix");
assertThat(name).endsWith("suffix");
assertThat(name).contains("substring");
assertThat(name).hasSize(10);
assertThat(name).matches("regex.*");

// 数字断言
assertThat(count).isGreaterThan(0);
assertThat(count).isGreaterThanOrEqualTo(1);
assertThat(count).isLessThan(100);
assertThat(count).isBetween(1, 100);

// 布尔断言
assertThat(flag).isTrue();
assertThat(flag).isFalse();

// 集合断言
assertThat(list).isNotNull();
assertThat(list).isEmpty();
assertThat(list).isNotEmpty();
assertThat(list).hasSize(3);
assertThat(list).contains(element);
assertThat(list).containsExactly(a, b, c);  // 顺序一致
assertThat(list).containsOnly(a, b, c);     // 无序
assertThat(list).doesNotContain(x);
assertThat(list).hasAtLeastOneElementOfType(String.class);
assertThat(list).allMatch(s -> s.length() > 0);
assertThat(list).anyMatch(s -> s.equals("target"));
```

### 7.2 RuoYi 响应对象断言

```java
// R<T> 响应断言
R<SysNoticeVo> r = noticeService.selectNoticeById(1L);
assertThat(r).isNotNull();
assertThat(r.getCode()).isEqualTo(200);
assertThat(r.getData()).isNotNull();
assertThat(r.getMsg()).isEqualTo("操作成功");

// TableDataInfo 分页断言
TableDataInfo<SysNoticeVo> page = noticeService.selectPageNoticeList(bo, pageQuery);
assertThat(page).isNotNull();
assertThat(page.getTotal()).isGreaterThan(0);
assertThat(page.getRows()).isNotEmpty();
assertThat(page.getRows().get(0).getNoticeTitle()).isEqualTo("预期标题");

// Map 响应断言（字典等）
Map<String, String> dictMap = dictService.selectDictDataByType("sys_notice_type");
assertThat(dictMap).isNotEmpty();
assertThat(dictMap).containsKey("1");
assertThat(dictMap.get("1")).isEqualTo("通知");
```

### 7.3 异常断言

```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.assertThatCode;

// 断言抛出指定异常
assertThatThrownBy(() -> noticeService.insertNotice(invalidBo))
    .isInstanceOf(ServiceException.class)
    .hasMessageContaining("公告标题不能为空");

// 断言不抛出异常
assertThatCode(() -> noticeService.insertNotice(validBo))
    .doesNotThrowAnyException();

// 断言抛出任何异常
assertThatThrownBy(() -> someService.riskyOperation())
    .isInstanceOf(Exception.class);
```

### 7.4 软断言（多个断言一起报告）

```java
import org.assertj.core.api.SoftAssertions;

SoftAssertions softly = new SoftAssertions();
softly.assertThat(vo.getNoticeId()).isEqualTo(1L);
softly.assertThat(vo.getNoticeTitle()).isEqualTo("测试公告");
softly.assertThat(vo.getNoticeType()).isEqualTo("1");
softly.assertThat(vo.getStatus()).isEqualTo("0");
softly.assertAll();  // 一次性报告所有失败

// 或者使用 assertSoftly（自动调用 assertAll）
assertSoftly(softly -> {
    softly.assertThat(vo.getNoticeId()).isEqualTo(1L);
    softly.assertThat(vo.getNoticeTitle()).isEqualTo("测试公告");
});
```

### 7.5 MockMvc 断言

```java
// JSON 路径断言
.andExpect(jsonPath("$.code").value(200))
.andExpect(jsonPath("$.data.noticeId").value(1))
.andExpect(jsonPath("$.data.noticeTitle").value("测试公告"))
.andExpect(jsonPath("$.rows", hasSize(10)))
.andExpect(jsonPath("$.rows[0].noticeType").exists())

// 响应状态断言
.andExpect(status().isOk())              // 200
.andExpect(status().isBadRequest())        // 400
.andExpect(status().isUnauthorized())      // 401
.andExpect(status().isForbidden())         // 403
.andExpect(status().isNotFound())          // 404

// 响应体断言
.andExpect(content().contentType(MediaType.APPLICATION_JSON))
.andExpect(content().string(containsString("操作成功")))

// 打印请求/响应（调试用）
.andDo(print())
```

---

## 八、RuoYi-Vue-Plus 特有测试要点

### 8.1 R<T> 统一响应

RuoYi 后端所有接口返回 `R<T>` 包装对象。测试时需要检查 `code`、`msg`、`data` 三个字段：

```java
// 成功响应
mockMvc.perform(get("/system/notice/1"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.code").value(200))
    .andExpect(jsonPath("$.msg").value("操作成功"))
    .andExpect(jsonPath("$.data").isNotEmpty());

// 失败响应
mockMvc.perform(get("/system/notice/999"))
    .andExpect(status().isOk())  // HTTP 200，但业务code非200
    .andExpect(jsonPath("$.code").value(500))
    .andExpect(jsonPath("$.msg").value(containsString("不存在")));
```

### 8.2 TableDataInfo 分页响应

分页接口返回特殊的 `TableDataInfo<T>` 结构：

```java
// 分页接口测试
mockMvc.perform(get("/system/notice/list")
        .param("pageNum", "1").param("pageSize", "10"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.total").isNumber())           // 总记录数
    .andExpect(jsonPath("$.rows").isArray())              // 数据列表
    .andExpect(jsonPath("$.code").value(200));

// Service 层断言
TableDataInfo<SysNoticeVo> page = noticeService.selectPageNoticeList(bo, pageQuery);
assertThat(page.getTotal()).isGreaterThanOrEqualTo(0);
assertThat(page.getRows()).isNotNull();
assertThat(page.getRows().size()).isLessThanOrEqualTo(pageQuery.getPageSize());
```

### 8.3 Bo 校验测试

RuoYi 的 Bo 对象使用 JSR-303 注解校验（`@NotBlank`、`@Size`、`@NotNull` 等）。Controller 层使用 `@Validated` 触发校验：

```java
@Nested
@DisplayName("Bo 校验测试")
class ValidationTests {

    @Test
    @DisplayName("标题为空 - 返回400")
    void titleBlank_shouldReturn400() throws Exception {
        SysNoticeBo bo = NoticeTestDataFactory.buildDefaultBo();
        bo.setNoticeTitle(null);

        mockMvc.perform(post("/system/notice")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(bo)))
            .andExpect(status().isBadRequest());
    }

    @Test
    @DisplayName("标题超长 - 返回400")
    void titleTooLong_shouldReturn400() throws Exception {
        SysNoticeBo bo = NoticeTestDataFactory.buildDefaultBo();
        bo.setNoticeTitle("x".repeat(51));  // @Size(max = 50)

        mockMvc.perform(post("/system/notice")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(bo)))
            .andExpect(status().isBadRequest());
    }

    @Test
    @DisplayName("状态值无效 - 返回400")
    void invalidStatus_shouldReturn400() throws Exception {
        SysNoticeBo bo = NoticeTestDataFactory.buildDefaultBo();
        bo.setStatus("99");  // 不在有效范围内

        mockMvc.perform(post("/system/notice")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(bo)))
            .andExpect(status().isBadRequest());
    }
}
```

### 8.4 MyBatis-Plus 条件构造器测试

Service 层通常使用 `LambdaQueryWrapper` 构建查询条件，需要验证条件是否正确传递给 Mapper：

```java
@Nested
@DisplayName("条件构造器测试")
class WrapperTests {

    @Test
    @DisplayName("按类型+状态查询 - 条件正确传递")
    void query_shouldPassCorrectWrapper() {
        testBo.setNoticeType("1");
        testBo.setStatus("0");

        when(baseMapper.selectVoPage(any(Page.class), any(LambdaQueryWrapper.class)))
            .thenReturn(new Page<>());

        noticeService.selectPageNoticeList(testBo, pageQuery);

        // 验证 Mapper 被调用
        verify(baseMapper).selectVoPage(any(Page.class), any(LambdaQueryWrapper.class));
    }

    @Test
    @DisplayName("按创建人名称查询 - 关联查询")
    void query_shouldJoinUserTable() {
        testBo.setCreateByName("admin");
        SysUserVo userVo = new SysUserVo();
        userVo.setUserId(1L);
        when(userMapper.selectVoOne(any())).thenReturn(userVo);
        when(baseMapper.selectVoPage(any(), any())).thenReturn(new Page<>());

        noticeService.selectPageNoticeList(testBo, pageQuery);

        // 验证先查用户，再查公告
        verify(userMapper).selectVoOne(any());
        verify(baseMapper).selectVoPage(any(), any());
    }
}
```

### 8.5 字典翻译测试

RuoYi 使用字典表实现数据翻译。测试时需确保字典数据已加载或被 Mock：

```java
@Nested
@DisplayName("字典翻译测试")
class DictTranslateTests {

    @Mock
    private ISysDictTypeService dictTypeService;

    @Test
    @DisplayName("公告类型字典翻译")
    void noticeType_shouldTranslateFromDict() {
        // 模拟字典查询返回
        when(dictTypeService.selectDictDataByType("sys_notice_type"))
            .thenReturn(Map.of("1", "通知", "2", "公告"));

        Map<String, String> dictMap = dictTypeService.selectDictDataByType("sys_notice_type");
        assertThat(dictMap).containsEntry("1", "通知");
        assertThat(dictMap).containsEntry("2", "公告");
    }
}
```

### 8.6 多租户隔离测试

RuoYi-Vue-Plus 支持多租户，测试时需验证租户数据隔离：

```java
@Nested
@DisplayName("多租户隔离测试")
class TenantTests {

    @Test
    @DisplayName("不同租户数据隔离")
    void differentTenant_shouldNotSeeOthersData() throws Exception {
        // 租户A的数据
        StpUtil.login(1L);
        StpUtil.getSession().set("tenantId", "000001");
        String tokenA = StpUtil.getTokenValue();

        // 创建公告
        SysNoticeBo bo = NoticeTestDataFactory.buildDefaultBo();
        mockMvc.perform(post("/system/notice")
                .header("Authorization", "Bearer " + tokenA)
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(bo)))
            .andExpect(status().isOk());

        // 租户B不应看到租户A的数据
        StpUtil.login(2L);
        StpUtil.getSession().set("tenantId", "000002");
        String tokenB = StpUtil.getTokenValue();

        mockMvc.perform(get("/system/notice/list")
                .param("pageNum", "1").param("pageSize", "10")
                .header("Authorization", "Bearer " + tokenB))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.total").value(0));
    }
}
```

### 8.7 数据权限测试

RuoYi 支持数据权限注解 `@DataScope`，测试时需验证数据范围：

```java
@Nested
@DisplayName("数据权限测试")
class DataScopeTests {

    @Test
    @DisplayName("本部门数据权限")
    void deptScope_shouldOnlySeeOwnDeptData() throws Exception {
        // 使用有部门数据权限的用户登录
        StpUtil.login(2L);
        String token = StpUtil.getTokenValue();

        mockMvc.perform(get("/system/user/list")
                .param("pageNum", "1").param("pageSize", "10")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.rows").isArray());
    }
}
```

---

## 九、测试运行与CI集成

### 9.1 运行测试命令

```bash
# 运行所有测试
mvn test

# 运行指定模块测试
mvn test -pl ruoyi-modules/ruoyi-system

# 运行指定测试类
mvn test -Dtest=SysNoticeServiceImplTest

# 运行指定测试方法
mvn test -Dtest=SysNoticeServiceImplTest#selectNoticeById_shouldReturnVoWhenExists

# 跳过测试
mvn package -DskipTests

# 运行指定Tag的测试
mvn test -Dgroups="integration"
```

### 9.2 测试Tag分组

```java
import org.junit.jupiter.api.Tag;

@Tag("unit")
class SysNoticeServiceImplTest { ... }

@Tag("integration")
class SysNoticeIntegrationTest { ... }

@Tag("slow")
class FullDataImportTest { ... }
```

### 9.3 GitLab CI 配置示例

```yaml
# .gitlab-ci.yml
 test:
  stage: test
  script:
    - mvn test -pl ruoyi-modules/ruoyi-system
  artifacts:
    reports:
      junit: "**/target/surefire-reports/TEST-*.xml"
  rules:
    - changes:
        - ruoyi-modules/ruoyi-system/**
```

---

## 十、常见问题与解决方案

### Q1: @WebMvcTest 启动报 Bean 冲突

**原因：** `@WebMvcTest` 只加载 MVC 层，但 Controller 依赖的全局异常处理器、拦截器等需要额外配置。

**解决：**
```java
@WebMvcTest(value = SysNoticeController.class, excludeFilters = {
    @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE,
        classes = {SaTokenConfigure.class})  // 排除不需要的配置
})
class SysNoticeControllerTest { ... }
```

### Q2: Mapper 测试连不上数据库

**解决：** 使用 H2 内存数据库 + `@ActiveProfiles("test")`：
```java
@SpringBootTest
@ActiveProfiles("test")
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class SysNoticeMapperTest { ... }
```

### Q3: Sa-Token 拦截导致测试 401

**解决：** 使用 `@MockBean` 替换 Sa-Token 拦截器，或直接用 `StpUtil.login()` 创建登录态：
```java
@BeforeEach
void login() {
    StpUtil.login(1L);
}
```

### Q4: 测试中事务不回滚

**原因：** 方法使用了 `@Transactional` 但测试类没有。

**解决：** 在测试类上添加 `@Transactional`：
```java
@SpringBootTest
@Transactional  // 自动回滚
@Rollback  // 显式声明回滚（默认就是true）
class MyTest { ... }
```

### Q5: 多租户插件导致 SQL 错误

**原因：** 测试时未设置租户ID，MyBatis-Plus 租户插件自动拼接 `tenant_id` 条件。

**解决：** 在测试中模拟租户上下文：
```java
@BeforeEach
void setupTenant() {
    // 设置当前租户（根据项目实际的租户上下文实现）
    TenantHelper.setDynamic("000000");
}

@AfterEach
void clearTenant() {
    TenantHelper.clearDynamic();
}
```

---

## 附录：测试检查清单

编写测试时，逐项确认：

- [ ] 单元测试使用 `@ExtendWith(MockitoExtension.class)`，不依赖 Spring 容器
- [ ] Service 层 Mock 了所有 Mapper 依赖
- [ ] Controller 层使用 `@WebMvcTest` + `@MockBean`
- [ ] 集成测试使用 `@SpringBootTest` + `@Transactional` 回滚
- [ ] 每个测试方法有 `@DisplayName` 描述
- [ ] 使用 `@Nested` 组织相关测试
- [ ] 断言使用 AssertJ（`assertThat`），不使用 JUnit 4 的 `assertEquals`
- [ ] 异常测试使用 `assertThatThrownBy`
- [ ] 边界值有覆盖（空值、超长、负数、零）
- [ ] RuoYi 响应格式 `R<T>` 和 `TableDataInfo` 有正确断言
- [ ] Sa-Token 权限有测试（登录/未登录/无权限）
- [ ] 多租户场景有隔离测试
- [ ] 测试数据通过工厂类或 Builder 构建，不硬编码