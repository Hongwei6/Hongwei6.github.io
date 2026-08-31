---
title: "Spring AI 实战 —— RAG 检索增强生成实现"
date: 2026-08-28T10:00:00+08:00
draft: false
description: "基于 Spring AI 实现 RAG 检索增强生成，包括相似度检索、手动 Prompt 拼接和 QuestionAdvisor 三种方式"
tags: ["Spring AI", "RAG", "Java", "向量检索"]
categories: ["AI"]
cover: "/hero/spring-ai-rag-retrieval/cover.png"
---

知识库构建完成后，下一步就是 RAG 的核心环节——**检索增强生成**。通过向量**相似度检索**从知识库中筛选最相关的内容，注入大模型上下文实现**增强生成**。

## 相似度检索

```java
/**
 * 相似度查询
 * @param query 用户的原始问题
 * @return 文档块
 */
public List<Document> similarSearch(String query) {
    return vectorStore.similaritySearch(SearchRequest.builder()
            .query(query)
            .topK(5)
            .similarityThreshold(0.7f)
            .build());
}
```

| 参数 | 含义 | 示例值 | 说明 |
|------|------|--------|------|
| query | 查询语句 | "什么是Mybatis？" | 自动向量化并与向量库对比 |
| topK | 返回条数 | 5 | 数值越大返回越多，一般 3~10 |
| similarityThreshold | 相似度阈值 | 0.7f | 越接近 1 越严格，常用 0.6~0.8 |

合理设置 topK 和 similarityThreshold 能有效提升检索精准度，减少模型幻觉。

## 增强生成（手动拼接）

```java
@Autowired
private ChatModel chatModel;

@GetMapping("/retrieve")
public String retrieve(String query, double threshold) {
    // 1. 相似度检索
    List<Document> documents = embeddingService.similaritySearch(SearchRequest.builder()
            .query(query)
            .similarityThreshold(threshold)
            .build());

    // 2. 拼接文档内容
    String documentContent = documents.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n=========文档分隔线===========\n\n"));

    // 3. 构建提示词
    String promptTemplate = """
            请基于以下提供的参考文档内容，回答用户的问题。
            如果参考文档中没有相关信息，请直接说明"没有找到相关信息"，不要编造内容。
            
            参考文档:
            {documents}
            
            用户问题: {question}
            """;

    PromptTemplate prompt = new PromptTemplate(promptTemplate);
    Prompt realPrompt = prompt.create(Map.of(
            "documents", documentContent,
            "question", query));

    return chatModel.call(realPrompt).getResult().getOutput().getText();
}
```

先根据用户问题在向量库中检索最相关的文档片段，再与提问一起构造 Prompt 交给大模型回答。当检索无命中时，大模型会返回"没有找到相关信息"，确保输出来自文档而非模型本身知识。

## 调用示例

先提问一个与文档无关的问题，触发 Prompt 要求返回"没有找到相关信息"：

**提问："什么是Redis"**

![Redis 查询结果](/hero/spring-ai-rag-retrieval/01-redis-query.png)

再问与文档相关的问题：

**提问："Mybatis是如何将sql执行结果封装为目标对象并返回的？"**

![PGvector 数据](/hero/spring-ai-rag-retrieval/02-pgvector-data.png)

![检索生成结果](/hero/spring-ai-rag-retrieval/03-retrieval-result.png)

大模型根据文本块内容综合分析，成功检索生成出结果。

## QuestionAnswerAdvisor

Spring AI 提供 **Advisor** 机制自动化 RAG 流程，无需手写检索和提示词拼接逻辑。

### 添加依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-advisors-vector-store</artifactId>
    <version>1.1.0</version>
</dependency>
```

### 配置 Advisor

```java
@RestController
@RequestMapping("/rag/retriever")
public class RagRetrieverController implements InitializingBean {

    private ChatClient chatClient;

    @Autowired
    private PgVectorStore vectorStore;

    @GetMapping("/retrieveAdvisor")
    public String retrieveAdvisor(String query) {
        return chatClient.prompt(query).call().content();
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        // 自定义 Prompt 模板
        PromptTemplate promptTemplate = new PromptTemplate("""
                请基于以下提供的参考文档内容，回答用户的问题。
                如果参考文档中没有相关信息，请直接说明"没有找到相关信息"，不要编造内容。
                
                参考文档内容:
                {question_answer_context}
                
                用户问题: {query}
                """);

        QuestionAnswerAdvisor questionAnswerAdvisor = QuestionAnswerAdvisor.builder(vectorStore)
                .searchRequest(SearchRequest.builder()
                        .similarityThreshold(0.5)
                        .topK(5)
                        .build())
                .promptTemplate(promptTemplate)
                .build();

        chatClient = ChatClient.builder(chatModel)
                .defaultAdvisors(questionAnswerAdvisor)
                .build();
    }
}
```

调用时只需简单一行代码：

```java
chatClient.prompt(query).call().content();
```

QuestionAnswerAdvisor 本身也内置了类似的 Prompt 模板：

![Advisor 源码](/hero/spring-ai-rag-retrieval/04-advisor-source.png)

调用效果：

![Advisor 调用结果](/hero/spring-ai-rag-retrieval/05-advisor-result.png)

## 实现原理

从源码层面看 QuestionAnswerAdvisor 的工作原理：

### 初始化阶段

把 RAG 所需的依赖装配好：向量库、检索请求（阈值、topK 等）、提示词模板、调度器（是否异步执行）、执行顺序。提供默认值，也允许自定义。

![初始化阶段](/hero/spring-ai-rag-retrieval/06-init-phase.png)

### 前置处理（before）

调用大模型前的处理操作：先检索相关文档，再把文档上下文拼到用户问题里，最后生成调用模型的请求体。

![前置处理](/hero/spring-ai-rag-retrieval/07-before-process.png)

### 后置处理（after）

模型回答后执行收尾工作，把检索到的文档信息（包括元数据）加到响应元信息里，不修改模型回答。

![后置处理](/hero/spring-ai-rag-retrieval/08-after-process.png)

## 两种方式对比

| 特性 | 手动拼接 | QuestionAnswerAdvisor |
|------|----------|----------------------|
| 灵活性 | 高，可控制每个步骤 | 低，流程固化 |
| 复用性 | 低，需重复编写 | 高，一次配置多处使用 |
| 适用场景 | 需要自定义检索逻辑 | 标准 RAG 流程 |

核心思路都是：**先检索相关文档 → 将文档拼接到提示词 → 调用大模型生成回答**。
