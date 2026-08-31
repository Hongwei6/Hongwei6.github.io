---
title: "Spring AI 实战 —— RAG 查询路由"
date: 2026-08-31T10:00:00+08:00
draft: false
description: "RAG 查询路由：根据问题意图将请求分发到不同数据源或提示词，包括数据源路由和 Prompt 路由"
tags: ["Spring AI", "RAG", "查询路由", "LangChain4j"]
categories: ["AI"]
cover: "/hero/rag-query-routing/cover.png"
---

RAG 从外部知识源检索相关信息并注入 LLM。但**外部知识源不一定是向量数据库，检索也不一定是向量检索**。企业内部知识除了文档，还有组织架构、服务器部署等结构化数据，这些并不在文档中，单靠向量数据库无法覆盖。因此需要**查询路由（Query Routing）**，把用户的请求转发到不同的数据源或提示词上去处理。

## 数据源路由

数据源路由根据问题意图，把请求分发到向量数据库、图数据库或关系型数据库等不同数据源上查询。

![查询路由流程](/hero/rag-query-routing/01-routing-flow.png)

### 数据库服务

三个模拟的数据库服务，分别对应不同类型的数据源：

```java
@Service
public class GraphDatabaseService {
    public String searchGraphDatabase(String query) {
        return "图数据库搜索结果: 基于关系图谱，找到与'" + query + "'相关的实体关系和路径。" +
                "这里模拟返回了知识图谱的实体关联结果，实际应用中会连接到Neo4j、ArangoDB或Amazon Neptune等图数据库。";
    }
}
```

```java
@Service
public class RelationalDatabaseService {
    public String searchRelationalDatabase(String query) {
        return "关系型数据库搜索结果: 基于结构化查询，找到与'" + query + "'匹配的数据记录。" +
                "这里模拟返回了SQL查询结果，实际应用中会连接到MySQL、PostgreSQL或Oracle等关系型数据库进行精确查询和统计分析。";
    }
}
```

```java
@Service
public class VectorDatabaseService {
    public String searchVectorDatabase(String query) {
        return "向量数据库搜索结果: 基于语义相似性，找到与'" + query + "'相关的文档片段。" +
                "这里模拟返回了相关的嵌入向量匹配结果，实际应用中会连接到真实的向量数据库如Chroma、Milvus、Faiss等。";
    }
}
```

### 意图识别

通过 LLM 判断用户问题适合哪种数据库：

```java
@Service
public class QueryRouteService {
    private static final String DATASOURCE_ROUTE_PROMPT = """
            你需要判断用户的查询问题适合使用哪种数据库进行检索。
            如果是语义相似性搜索、文档检索、内容推荐类问题，回答'VECTOR'
            如果是关系查询、知识图谱、实体关联类问题，回答'GRAPH'
            如果是结构化数据查询、统计分析、精确匹配类问题，回答'RELATIONAL'
            如果无法确定，请回答'VECTOR'
            只回答VECTOR、GRAPH或RELATIONAL，不要其他内容。

            用户问题：
            {QUESTION}
            """;

    @Autowired
    private ChatModel chatModel;

    public String routeDataSource(String query) {
        PromptTemplate promptTemplate = new PromptTemplate(DATASOURCE_ROUTE_PROMPT);
        promptTemplate.add("QUESTION", query);
        return chatModel.call(promptTemplate.create()).getResult().getOutput().getText();
    }
}
```

### 路由串联

根据识别结果调用对应的数据库服务：

```java
@RestController
@RequestMapping("/rag/routing")
public class RagQueryRoutingController {
    @Autowired
    private QueryRouteService ragRoutingService;
    @Autowired
    private VectorDatabaseService vectorDatabaseService;
    @Autowired
    private GraphDatabaseService graphDatabaseService;
    @Autowired
    private RelationalDatabaseService relationalDatabaseService;

    @RequestMapping("/query")
    public String ragQuery(HttpServletResponse response, @RequestParam String question) {
        response.setCharacterEncoding("UTF-8");
        String databaseType = ragRoutingService.routeDataSource(question);
        String result;
        switch (databaseType.trim()) {
            case "VECTOR":
                result = vectorDatabaseService.searchVectorDatabase(question);
                break;
            case "GRAPH":
                result = graphDatabaseService.searchGraphDatabase(question);
                break;
            case "RELATIONAL":
                result = relationalDatabaseService.searchRelationalDatabase(question);
                break;
            default:
                result = "无法确定合适的数据库类型，默认使用向量数据库: " +
                        vectorDatabaseService.searchVectorDatabase(question);
        }
        return String.format("路由到: %s 数据库\n\n查询结果:\n%s", databaseType, result);
    }
}
```

## Prompt 路由

另一种常见路由是**提示词路由**。固定提示词效果有限，可提前定义多个提示词，针对用户问题先通过 LLM 分析适合用哪个提示词。

以医疗 AI 助手为例，定义两个系统提示词：

- Prompt 1：专业的医生，从医疗角度给出患者建议
- Prompt 2：专业的药学专家，在用药方面给出建议

根据用户是询问病情还是用药建议，路由到不同提示词。

### 定义提示词服务

以下代码使用 langchain4j 的 API：

```java
import dev.langchain4j.service.SystemMessage;
import dev.langchain4j.service.spring.AiService;
import reactor.core.publisher.Flux;

@AiService
public interface MedicalPromptRoutingService {
    @SystemMessage("你是一个专业的医生，可以从专业的医疗角度给出患者建议。")
    Flux<String> doctorConsultation(String userMessage);

    @SystemMessage("你是一个专业的药学专家，掌握丰富的药品知识，能够在用药方面给出更好的建议。")
    Flux<String> pharmacistConsultation(String userMessage);

    @SystemMessage("你需要判断用户的询问是关于病情咨询还是用药建议。如果是询问病情、症状、诊断相关的问题，回答'DOCTOR'。如果是询问药物、用药方法、药物副作用相关的问题，回答'PHARMACIST'。只回答DOCTOR或PHARMACIST，不要其他内容。")
    String determineConsultationType(String userMessage);
}
```

### 路由调用

先用 `determineConsultationType` 做意图识别，再根据结果调用对应方法：

```java
@RestController
@RequestMapping("/medical")
public class MedicalAssistantController {
    @Autowired
    private MedicalPromptRoutingService medicalRoutingService;

    @RequestMapping("/consultation")
    public Flux<String> medicalConsultation(HttpServletResponse response, @RequestParam String question) {
        response.setCharacterEncoding("UTF-8");
        String consultationType = medicalRoutingService.determineConsultationType(question);
        if ("DOCTOR".equals(consultationType.trim())) {
            return medicalRoutingService.doctorConsultation(question);
        } else if ("PHARMACIST".equals(consultationType.trim())) {
            return medicalRoutingService.pharmacistConsultation(question);
        } else {
            return medicalRoutingService.doctorConsultation(question);
        }
    }
}
```

## 总结

查询路由的本质是**先意图识别，再分发处理**：数据源路由解决"查哪里"（向量库 / 图库 / 关系库），Prompt 路由解决"用什么角色回答"。两者都是通过 LLM 对用户问题进行分类，再路由到对应的处理逻辑，是生产级 RAG 提升回答质量与覆盖面的常用手段。
