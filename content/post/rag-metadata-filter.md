---
title: "Spring AI 实战 —— RAG 元数据过滤优化"
date: 2026-08-28T10:30:00+08:00
draft: false
description: "通过元数据过滤提升 RAG 检索精度，实现精确过滤、参考源追溯和访问权限控制"
tags: ["Spring AI", "RAG", "元数据", "向量检索"]
categories: ["AI"]
cover: "/hero/rag-metadata/filter-flow.png"
---

在 RAG 系统中，向量检索的本质是**相似度查询**，但有时我们需要类似关系型数据库的**精确检索**。元数据过滤可以将两者结合，显著提升检索精度。

## 什么是元数据

元数据是附加到文本块上的结构化信息，描述该文本块的属性，如文件名、页码、用户 ID 等。在向量数据库中，每个文本块都有一个 `metadata` 字段用于存储这些信息。

![metadata 字段](/hero/rag-metadata/metadata-field.png)

## 使用场景

### 精确过滤

假设有一个汽车用户手册文档集：

- 《汽车用户手册（2023年版）》— 钥匙启动
- 《汽车用户手册（2024年版）》— 旋钮启动
- 《汽车用户手册（2025年版）》— 手机启动

用户提问"根据《2023年版》，汽车如何启动？"时，由于三个版本文本块仅年份数字不同，相似度都非常高，普通检索可能返回三个版本的内容。

通过元数据过滤：先筛选 `fileName == '2023年版'`，再执行相似度检索，确保只返回正确版本的内容。

### 提供参考源

为每个文本块存储元数据：

- 文档名称（来源文件）
- 页码或章节位置
- 文档类型或版本号

生成答案时展示引用来源：

> **参考来源：**《汽车用户手册（2024年版）》第5页

作用：
- **增加可信度**：用户知道回答来自文档而非模型编造
- **便于追溯**：有疑问时可快速定位原文核对
- **提升体验**：文档量大时帮助快速定位信息

### 访问权限控制

企业级应用中，文档涉及不同安全级别。将权限信息写入元数据：

- 部门 ID / 角色 ID / 用户 ID
- 保密等级
- 生效时间或版本状态

查询时先根据用户身份过滤，确保 RAG 只基于用户能访问的内容生成答案。

## 实现示例

### 存储元数据

修改索引构建方法，在文档切片和向量化之间添加元数据：

![导入向量库](/hero/rag-metadata/import-vector-db.png)

```java
@GetMapping("/embedding")
public String embedding(String filePath, String fileName) {
    List<Document> documents;
    try {
        documents = documentReaderFactory.read(new File(filePath));
    } catch (IOException e) {
        throw new RuntimeException(e);
    }

    for (Document document : documents) {
        document.getMetadata().put("fileName", fileName);
    }

    embeddingService.embedAndStore(documents);
    return "success";
}
```

### 检索过滤

在相似度检索中添加 `filterExpression`：

```java
List<Document> similarDocs = vectorStore.similaritySearch(SearchRequest.builder()
        .query(query)
        .topK(5)
        .similarityThreshold(0.5f)
        .filterExpression("fileName == '" + fileName + "'")
        .build());
```

`filterExpression` 支持精确匹配表达式。阈值 `similarityThreshold` 需要根据实际场景反复调试，找到最佳值。

### 带元数据的检索生成

```java
@GetMapping("/retrieveMetadata")
public String retrieveMetadata(String query, String fileName) {
    SearchRequest searchRequest = SearchRequest.builder()
            .query(query)
            .filterExpression("fileName == '" + fileName + "'")
            .build();
    return embeddingService.similaritySearch(searchRequest).toString();
}
```

> 实际环境中，文件名的提取工作需要大模型进行参数抽取。

### 结合 QuestionAnswerAdvisor

使用 Advisor 时设置过滤参数：

```java
@GetMapping("/retrieveAdvisorWithMetadata")
public String retrieveAdvisorWithMetadata(String query, String fileName) {
    return chatClient.prompt(query)
            .advisors(advisorSpec -> advisorSpec.param(
                    "qa_filter_expression",
                    "fileName == '" + fileName + "'"))
            .call().content();
}
```

`qa_filter_expression` 是 `QuestionAnswerAdvisor` 规定的过滤参数名。

## 总结

元数据过滤的核心价值：

| 场景 | 作用 |
|------|------|
| 精确过滤 | 结合相似度检索与精确匹配，避免返回无关版本 |
| 参考源追溯 | 提供引用来源，增强回答可信度 |
| 权限控制 | 按用户身份过滤文档，防止数据泄露 |

元数据是文本块的"身份证"，让检索更精准，也让用户看到可靠的参考依据。
