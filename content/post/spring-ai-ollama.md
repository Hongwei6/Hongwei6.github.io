---
title: "Spring AI 集成 Ollama 调用本地模型"
date: 2026-08-05T12:00:00+08:00
draft: false
description: "介绍如何在 Spring AI 中集成 Ollama，实现本地模型调用。"
tags: ["Spring AI", "Java", "AI", "Ollama", "本地模型"]
categories: ["技术"]
---

## 背景

Ollama 是一个流行的本地模型部署工具。Spring AI 通过 `spring-ai-ollama` 模块提供了对 Ollama 的原生支持，其中定义了 `OllamaChatModel` 作为 `ChatModel` 的实现。

## 集成步骤

### 添加依赖

首先引入 Ollama 核心依赖：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-ollama</artifactId>
    <version>1.1.0</version>
</dependency>
```

### 版本兼容问题

早期版本的 `spring-ai-ollama-spring-boot-starter`（最高到 1.0.0-M6）与新版本 Spring AI 存在兼容性问题。

原因是 Spring AI 新版本将 `spring-ai-spring-boot-autoconfigure` 拆分成了多个独立的 starter：

![自动配置拆分](/images/posts/spring-ai-ollama/autoconfigure-split.png)

这会导致同一个 Bean 在多个地方初始化，应用启动失败。

### 解决方案

使用新版本提供的独立自动配置模块：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-autoconfigure-model-ollama</artifactId>
    <version>1.1.0</version>
</dependency>
```

### 配置

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        model: deepseek-r1:7b
```

## 使用示例

注入 `OllamaChatModel`：

```java
@Autowired
@Qualifier("ollamaChatModel")
private ChatModel ollamaChatModel;
```

创建流式调用接口：

```java
@RestController
@RequestMapping("/ai/ollama")
public class OllamaChatController {

    @Autowired
    @Qualifier("ollamaChatModel")
    private ChatModel ollamaChatModel;

    @GetMapping("/stream/chat")
    public Flux<String> streamChat(HttpServletResponse response) {
        response.setCharacterEncoding("UTF-8");
        Flux<ChatResponse> stream = ollamaChatModel.stream(new Prompt("你是谁？"));
        return stream.map(resp -> resp.getResult().getOutput().getText());
    }
}
```

启动应用后访问接口，即可调用本地 DeepSeek 模型：

![调用结果](/images/posts/spring-ai-ollama/ollama-result.png)

## 小结

Spring AI 对 Ollama 的集成比较简单，核心是引入正确的依赖版本。新版本的模块拆分虽然增加了配置复杂度，但也提供了更灵活的依赖管理。
