---
title: "Spring AI 实战 —— RAG 问题改写优化"
date: 2026-08-28T11:00:00+08:00
draft: false
description: "通过问题改写提升 RAG 检索效果，包括分解、富化、多样化、回溯提示四种策略"
tags: ["Spring AI", "RAG", "问题改写", "Prompt Engineering"]
categories: ["AI"]
cover: "/hero/rag-query-rewrite/cover.png"
---

在 RAG 的相似度检索中，检索效果很大程度取决于用户问题的表达方式。用户输入往往存在**模糊、信息缺失、上下文依赖**等问题，难以直接高效匹配到高质量文档块。问题改写的目标是：**将自然语言提问转化为更适合检索的查询表达**。

## 四种改写策略

### 1. 分解（子问题）

将复杂、多步骤或包含多个子问题的查询，拆解成若干相互独立、更具体的子问题。

**示例：** "iPhone 15 发布的时候，苹果的 CEO 是谁？"

分解为：
- Q1：iPhone 15 是什么时候发布的？
- Q2：2023 年 9 月时，苹果公司的 CEO 是谁？

**Prompt 模板：**

```
# 角色
你是一名专业的查询逻辑分析专家。

# 任务
将给定的"用户原始问题"分解为一系列相互独立、逻辑清晰，
且可单独用于检索的子查询列表。输出必须是标准 JSON 数组格式。

# 用户原始问题
{QUESTION}

# 输出格式要求 (JSON Array)
["子查询1", "子查询2", "..."]
```

### 2. 富化

在原始查询中添加上下文信息、背景知识或限制条件，消除歧义、补充缺失信息。

**适用场景：**
- 存在指代关系（如"他有什么特点？"中的"他"）
- 省略主语或关键信息
- 缺少时间、地点等限制条件

**Prompt 模板：**

```
# 角色
你是一个专业的问题重写优化器。

# 任务
根据提供的"对话历史"和"用户原始问题"，重写为一个独立、完整、
且包含所有必要背景信息的新查询，用于 RAG 检索。

## 对话历史：
{CHAT_HISTORY}

## 原始问题：
{QUESTION}

# 输出
输出富化过后的新问题，不要包含多余的解释性内容
```

### 3. 多样化

对同一查询生成多个语义相近但措辞不同的变体，提升召回率。

**适用场景：**
- 文档表述风格不一致（如"接口性能优化" vs "提升接口响应速度"）
- 不同用户表达习惯不同

**Prompt 模板：**

```
# 角色
你是一名专业的语义扩展专家。

# 任务
为给定的"原始问题"生成 3 个语义相同但措辞完全不同、且利于检索的查询变体。
输出必须是标准 JSON 数组格式。

# 原始问题
{QUESTION}

# 输出格式要求 (JSON Array)
["变体1", "变体2", "变体3"]
```

### 4. 回溯提示（Step-Back Prompting）

引导模型"后退一步"，从具体问题中抽象出更一般的原理或概念，再基于这些信息进行检索。

**示例：** "我老舅结婚，我可以请几天假？"

回溯为："探亲假的请假政策是怎么样的？"

**Prompt 模板：**

```
# 角色
你是一个擅长抽象思维和原理推理的专家。

# 任务
请根据用户提出的具体问题，先"后退一步"，将其转化为一个更通用、更本质的问题，
聚焦于背后的原理、规律、概念或一般性知识，而不是具体细节。

# 原始问题
{QUESTION}

# 输出
请只输出改写后的"后退问题"，不要解释，不要包含原始问题，也不要回答它。
```

## 实战实现

### 问题改写服务

```java
@Service
@Slf4j
public class QuestionRewriteService {

    @Autowired
    private ChatModel chatModel;

    /**
     * 问题分解
     */
    public List<String> decompose(String question) {
        PromptTemplate promptTemplate = new PromptTemplate(DECOMPOSE_PROMPT);
        promptTemplate.add("QUESTION", question);
        String result = chatModel.call(promptTemplate.create())
                .getResult().getOutput().getText();
        return JSON.parseArray(result, String.class);
    }

    /**
     * 问题富化
     */
    public String enrich(String chatHistory, String question) {
        PromptTemplate promptTemplate = new PromptTemplate(ENRICH_PROMPT);
        promptTemplate.add("CHAT_HISTORY", chatHistory);
        promptTemplate.add("QUESTION", question);
        return chatModel.call(promptTemplate.create())
                .getResult().getOutput().getText();
    }

    /**
     * 问题多样化
     */
    public List<String> diversify(String question) {
        PromptTemplate promptTemplate = new PromptTemplate(DIVERSIFY_PROMPT);
        promptTemplate.add("QUESTION", question);
        String result = chatModel.call(promptTemplate.create())
                .getResult().getOutput().getText();
        return JSON.parseArray(result, String.class);
    }

    /**
     * 问题回退
     */
    public String stepBack(String question) {
        PromptTemplate promptTemplate = new PromptTemplate(STEP_BACK_PROMPT);
        promptTemplate.add("QUESTION", question);
        return chatModel.call(promptTemplate.create())
                .getResult().getOutput().getText();
    }

    /**
     * 组合策略：回退 → 分解 → 多样化
     */
    public List<String> rewriteQuery(String query) {
        // 1. 回退
        String stepBackQuery = this.stepBack(query);
        // 2. 分解
        List<String> decomposedQueries = this.decompose(stepBackQuery);
        // 3. 多样化
        List<String> finalQueries = new ArrayList<>();
        for (String subQuery : decomposedQueries) {
            List<String> variations = this.diversify(subQuery);
            finalQueries.addAll(variations);
        }
        if (finalQueries.isEmpty()) {
            finalQueries.add(query);
        }
        return finalQueries;
    }
}
```

### 结合 RAG 检索

```java
@GetMapping("/chatWithQueryRewrite")
public String chatWithQueryRewrite(@RequestParam("query") String query) {
    // 1. 问题改写
    List<String> rewriteQuery = queryRewriteService.rewriteQuery(query);

    // 2. 多路检索并去重
    Set<Document> similarDocs = new LinkedHashSet<>();
    for (String q : rewriteQuery) {
        List<Document> docs = embeddingService.similarSearch(q);
        if (docs != null && !docs.isEmpty()) {
            similarDocs.addAll(docs);
        }
    }

    // 3. 构建提示词并生成回答
    String documentContent = similarDocs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));

    PromptTemplate prompt = new PromptTemplate("""
            请基于以下提供的参考文档内容，回答用户的问题。
            参考文档: {documents}
            用户问题: {question}
            """);

    Prompt realPrompt = prompt.create(Map.of(
            "documents", documentContent,
            "question", query));

    return chatClient.prompt(realPrompt).call()
            .chatResponse().getResult().getOutput().getText();
}
```

## 效果对比

**不含问题改写：** 检索到 0 个相关文档块，回答来自模型本身知识而非文档。

![无改写日志](/hero/rag-query-rewrite/01-no-rewrite-log.png)

**含问题改写：** 通过回退、分解、多样化组合处理，检索到 5 个相关文档块。

![有改写日志](/hero/rag-query-rewrite/04-with-rewrite-log.png)

## 注意事项

问题重写是生产级 RAG 的必备操作，但并非策略越多效果越好。核心在于**在性能与质量之间找到平衡点**：

- 过多冗余流程会显著增加响应时间
- 需结合业务场景灵活组合策略
- 分析用户提问习惯，选择最合适的改写方式
