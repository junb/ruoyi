# Quickstart: 项目知识助手

**Branch**: `005-knowledge-assistant` | **Date**: 2026-04-18

## 前置条件

1. **Chroma Server** 运行中（默认端口 8000）
   ```bash
   docker run -d -p 8000:8000 chromadb/chroma
   ```

2. **百炼 API Key** 已配置（在字典管理中配置 `knowledge_assistant` 字典类型的 `BAILIAN_API_KEY`）

3. **MySQL** 和 **Redis** 正常运行

## 后端启动

```bash
export JAVA_HOME=/Users/jun/Library/Java/JavaVirtualMachines/azul-17.0.15/Contents/Home
MVN=/Users/jun/Documents/tools/maven/apache-maven-3.9.14/bin/mvn

cd RuoYi-Vue-Plus
$MVN clean package -P dev -pl ruoyi-admin -am
$MVN spring-boot:run -pl ruoyi-admin -P dev
```

## 前端启动

```bash
cd plus-ui
npm run dev
```

## 配置项

在 `application-dev.yml` 中新增以下配置：

```yaml
# 知识助手配置
knowledge:
  chroma:
    url: http://localhost:8000
  embedding:
    model: text-embedding-v3
    batch-size: 25
  chat:
    model: qwen-plus
    max-history-rounds: 10
    top-k: 5
  document:
    chunk-size: 500
    chunk-overlap: 50
    max-file-size: 20MB
```

## 使用流程

1. 登录系统，进入 **知识助手 → 知识管理**
2. 上传 PDF/Word/文本文件，等待解析完成
3. 进入 **知识助手 → 知识问答**
4. 新建对话，输入问题，AI 基于文档内容流式回答

## 数据库初始化

SQL 脚本位于 `ruoyi-modules/ruoyi-knowledge/src/main/resources/sql/`，包含：
- 建表语句
- 菜单和权限初始化数据
