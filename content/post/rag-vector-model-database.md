---
title: "RAG 向量模型、向量数据库与向量存储"
date: 2026-08-24T11:30:00+08:00
draft: false
description: "Spring AI 向量化与存储实践，包括 EmbeddingModel、PGvector 接入与使用"
tags: ["Spring AI", "RAG", "Java", "PGvector", "向量数据库"]
categories: ["AI"]
cover: "/hero/rag-vector-store/spring-ai-embedding-models.png"
---

经过文档清洗与分片处理后，获得结构化干净的 `List<Document>` 数据。接下来进行向量化处理并存储到向量数据库，完成索引构建。

## 向量模型

Spring AI 提供 `EmbeddingModel` 接口，支持各厂商的离线向量模型快速接入，也可使用百炼平台的在线向量模型。

![Spring AI 支持的向量模型](/hero/rag-vector-store/spring-ai-embedding-models.png)

### DashScopeEmbeddingModel

百炼提供的 EmbeddingModel 实现，Spring 中默认有这个 Bean，可直接使用。

配置：
```yaml
spring:
  ai:
    dashscope:
      embedding:
        options:
          model: text-embedding-v4
          dimensions: 768
```

- `model`：使用的 embedding 模型
- `dimensions`：向量维度

### OpenAiEmbeddingModel

兼容市面上大部分 OpenAI 规范的向量模型，包括通义千问。

依赖：
```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

配置：
```yaml
spring:
  ai:
    openai:
      embedding:
        base-url: https://dashscope.aliyuncs.com/compatible-mode/
        api-key: xxxxxxxxxxxxxxxxxxxxxxxxxxxx
        options:
          dimensions: 768
          model: text-embedding-v4
```

### 向量化实现

向量化本质是对 `List<Document>` 遍历并执行 embed 操作，得到 `List<float[]>`，每个文档表示为一个浮点数组（多维向量）。

```java
public List<float[]> embed(List<Document> documents) {
    if (CollectionUtils.isEmpty(documents)) {
        return new ArrayList<>();
    }
    return documents.stream()
            .map(doc -> openAiEmbeddingModel.embed(doc))
            .collect(Collectors.toList());
}
```

## 向量数据库

常见向量数据库选型：

| 数据库 | 特点 | 适用场景 |
|--------|------|----------|
| **Milvus** | 开源分布式，高性能 | 海量数据、高并发 |
| **Elasticsearch** | 支持向量检索 | 混合检索场景 |
| **Chroma** | 轻量级，易用 | 小规模应用 |
| **PGvector** | PostgreSQL 扩展，部署简单 | Java 后端开发友好 |

选用 **PGvector** 进行演示，方便操作，易于理解向量存储和检索流程。

### 安装 PGvector

```bash
docker run --name pgvector \
  -e POSTGRES_USER=pgvector \
  -e POSTGRES_PASSWORD=pgvector \
  -e POSTGRES_DB=rag_test \
  -p 5433:5432 \
  -v /home/docker_pgvector:/var/lib/postgresql/data \
  -d ankane/pgvector:v0.5.0
```

数据库名称 `rag_test`，用户密码均为 `pgvector`，端口映射 5433。

远程连接需修改 `pg_hba.conf`，添加：
```
host    all             all        all                     scram-sha-256
```

重启容器后可用 Navicat 远程连接：

![Navicat 连接](/hero/rag-vector-store/navicat-connection.png)

### 引入 PGvector

依赖：
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
    <version>1.1.0</version>
</dependency>
```

会自动注入 `VectorStore` Bean，依赖两个关键 Bean：
- **JdbcTemplate**：与向量数据库交互，做 CRUD
- **EmbeddingModel**：做 embed，把文本转换成向量

![VectorStore Bean](/hero/rag-vector-store/vectorstore-bean.png)

配置：
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5433/rag_test
    username: pgvector
    password: pgvector
  ai:
    vectorstore:
      pgvector:
        index-type: HNSW           # 向量索引类型
        distance-type: COSINE_DISTANCE  # 距离度量
        dimensions: 768            # 向量维度
        max-document-batch-size: 9 # 批量处理大小
        initialize-schema: true    # 启动时自动创建表
        table-name: vector_st      # 表名
```

### EmbeddingService 实现

```java
@Service
public class EmbeddingService {
    @Autowired
    private EmbeddingModel embeddingModel;
    @Autowired
    private VectorStore vectorStore;

    public List<float[]> embed(List<Document> documents) {
        return documents.stream()
                .map(document -> embeddingModel.embed(document.getText()))
                .collect(Collectors.toList());
    }

    public void embedAndStore(List<Document> documents) {
        if (CollectionUtils.isEmpty(documents)) {
            return;
        }
        int batchSize = 9;
        for (int i = 0; i < documents.size(); i += batchSize) {
            List<Document> subList = documents.subList(i, Math.min(i + batchSize, documents.size()));
            vectorStore.add(subList);
        }
    }
}
```

### PgVectorStore 实现原理

`VectorStore.add()` 方法包含两个步骤：向量化和存储。

```java
public void doAdd(List<Document> documents) {
    // 使用 embeddingModel 做向量化
    List<float[]> embeddings = this.embeddingModel.embed(documents, EmbeddingOptions.builder().build(), this.batchingStrategy);
    // 使用 JdbcTemplate 做存储
    List<List<Document>> batchedDocuments = this.batchDocuments(documents);
    batchedDocuments.forEach((batchDocument) -> this.insertOrUpdateBatch(batchDocument, documents, embeddings));
}
```

### 批次大小限制

不同的 embedding 模型有批次大小限制：

![批次大小限制](/hero/rag-vector-store/batch-size-limit.png)

`max-document-batch-size` 参数控制一次发送多少个 document 给向量数据库存储，但**不控制一次给向量模型多少个批次的文档做向量化**。

Debug 可以看到，embed 方法在 batchDocuments 之前调用：

![Debug 批次](/hero/rag-vector-store/debug-batch.png)

![doAdd 方法](/hero/rag-vector-store/doadd-method.png)

因此需要在 `embedAndStore` 中手动分批，一次只处理 9 个 document。

## 向量存储

完整流程：
```java
@RequestMapping("embed")
public String embed(String filePath) {
    List<Document> documents;
    try {
        documents = documentReaderFactory.read(new File(filePath));
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
    List<Document> allChunkedDocuments = documents.stream()
            .flatMap(document -> {
                RecursiveCharacterTextSplitter splitter = new RecursiveCharacterTextSplitter(300, new String[]{"\n\n", "\n"});
                return splitter.split(document).stream();
            })
            .collect(Collectors.toList());
    embeddingService.embedAndStore(allChunkedDocuments);
    return "success";
}
```

经过 VectorStore 向量化+存储后，可在向量数据库中查看结果：

![向量数据库结果](/hero/rag-vector-store/vector-db-result.png)

可清晰查看文本块的**向量、原始文本、以及元数据**。至此完成向量数据库构建，后续可进行检索增强。