---
title: "初识 LangChain4j"
date: 2026-08-06T12:00:00+08:00
draft: false
description: "LangChain4j 是 LangChain 的 Java 实现，通过统一接口连接不同 LLM 与外部工具，实现模型、数据、业务的解耦。"
tags: ["LangChain4j", "Java", "AI", "LLM"]
categories: ["技术"]
---

## LangChain 与 LangChain4j

LangChain 是一个开源框架，用于开发基于大语言模型（LLM）的应用程序，提供了一套完整的工具和组件，将提示词、记忆、外部存储（RAG）、链式调用、Agent 等能力串联起来，简化 LLM 应用的开发流程。

它在 Java 生态中的定位类似于数据库领域的 JDBC——通过统一接口连接不同的 LLM（如 GPT-4、ChatGLM、Qwen）与外部工具（数据库、API），实现「模型-数据-业务」的解耦。

![LangChain 与 JDBC 的类比](/images/posts/langchain4j-intro/langchain-jdbc.png)

LangChain 官方推出了 Python 版和 JS 版，在 Python 中使用便捷，但对 Java 开发者并不友好。LangChain4j 正是为 Java 量身打造的 LangChain 实现。

## 与 Spring AI 的关系

LangChain4j 的能力与 Spring AI 基本对齐，同样覆盖提示词工程、提示词模板、对话记忆、结构化输出，以及 RAG、Agent、MCP 等场景。Spring AI 提供的能力，LangChain4j 几乎都有。

LangChain4j 为开发者提供了两个层次的抽象：

**低层次抽象**——提供 Basics（大模型、提示词模板、模型记忆等）和 RAG（向量模型、向量数据库、文本载入分割工具）两类接口，开发者可以灵活实现并按需组合，定制自己的大模型应用。

**高层次抽象**——为了让 Java 开发者更聚焦业务逻辑而非底层实现，LangChain4j 提供了两个高层次 API：

- **Chains**：源于 LangChain，将低层次模块组合成固定的处理流程，协调它们之间的交互。
- **AI Services**：为 Java 量身定制的解决方案，类似于 Spring Data JPA——只需显式定义接口，即可按需注入 Memory、Tools 或 RAG，具体的调用逻辑由 LangChain4j 代理完成。

![LangChain4j 架构层次](/images/posts/langchain4j-intro/langchain4j-architecture.png)

## 接入 LangChain4j

引入两个依赖：`langchain4j` 核心包，以及 `langchain4j-open-ai-spring-boot-starter`，后者是一个 Spring Boot Starter，内部初始化了相关 Bean 的定义。

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j</artifactId>
    <version>1.8.0</version>
</dependency>

<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
    <version>1.8.0-beta15</version>
</dependency>
```

在 `application.properties` 中配置模型相关变量：

```properties
langchain4j.open-ai.chat-model.api-key={YOUR_KEY}
langchain4j.open-ai.chat-model.model-name=qwen-max-latest
langchain4j.open-ai.chat-model.base-url=https://dashscope.aliyuncs.com/compatible-mode/v1
langchain4j.open-ai.chat-model.log-requests=true
langchain4j.open-ai.chat-model.log-responses=true
```

`log-requests` 和 `log-responses` 用于开启日志打印，方便查看与模型交互的提示词和响应。

完成配置后即可在代码中调用：

```java
@RestController
@RequestMapping("/langchain")
public class LangChainController {

    @Autowired
    OpenAiChatModel chatModel;

    @RequestMapping("/hello")
    public String hello() {
        return chatModel.chat("你好,你是谁？");
    }
}
```

注入 `OpenAiChatModel`，调用其 `chat` 方法并传入用户提示词即可完成对话。大模型的回复如下：

![LangChain4j 调用结果](/images/posts/langchain4j-intro/langchain4j-result.png)

`OpenAiChatModel` 由 `langchain4j-open-ai-spring-boot-starter` 自动装配，会读取 `langchain4j.open-ai` 前缀下的相关配置。
