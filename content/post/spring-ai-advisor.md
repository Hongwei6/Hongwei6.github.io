---
title: "Spring AI 核心特性：Advisor 机制详解"
date: 2026-08-05T11:00:00+08:00
draft: false
description: "深入解析 Spring AI 中的 Advisor 机制，包括接口设计、执行流程和内置实现。"
tags: ["Spring AI", "Java", "AI", "Advisor"]
categories: ["技术"]
---

## 概述

Spring AI 提供了一个灵活且强大的拦截机制——Advisor。它类似于插件，可以拦截、修改和增强 AI 交互功能。

例如，实现记忆功能可以使用 Memory 相关的 Advisor：

![Advisor 概览](/images/posts/spring-ai-advisor/advisor-overview.png)

实现日志记录可以添加 Logger Advisor：

![Logger Advisor](/images/posts/spring-ai-advisor/logger-advisor.png)

Spring AI 内置了多种 Advisor：

![内置 Advisor](/images/posts/spring-ai-advisor/builtin-advisors.png)

## 接口设计

Advisor 接口继承自 `Ordered`，主要用于控制多个 Advisor 的执行顺序：

```java
public interface Advisor extends Ordered {
    int DEFAULT_CHAT_MEMORY_PRECEDENCE_ORDER = -2147482648;
    String getName();
}
```

接口体系包含三个核心子接口：

![Advisor 接口体系](/images/posts/spring-ai-advisor/advisor-interface.png)

- **CallAdvisor**：同步调用场景
- **StreamAdvisor**：流式调用场景
- **BaseAdvisor**：基础抽象类，提供 before/after 钩子

```java
public interface CallAdvisor extends Advisor {
    ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain);
}

public interface StreamAdvisor extends Advisor {
    Flux<ChatClientResponse> adviseStream(ChatClientRequest request, StreamAdvisorChain chain);
}
```

## 执行流程

Advisor 的执行机制类似于 AOP。在 `DefaultChatClient` 中维护了一个 `advisorChain`，按顺序依次调用每个 Advisor：

![Advisor 链](/images/posts/spring-ai-advisor/advisor-chain.png)

调用流程：

![调用流程](/images/posts/spring-ai-advisor/call-flow.png)

## 内置 Advisor 实现

`adviseCall` 方法的实现主要分为四类：

![Advisor 类型](/images/posts/spring-ai-advisor/advisor-types.png)

### ChatModelCallAdvisor

直接调用 `chatModel.call()` 与大模型交互。由于是最终执行者，优先级设置为最低：

![ChatModelCallAdvisor](/images/posts/spring-ai-advisor/chatmodel-advisor.png)

```java
@Override
public int getOrder() {
    return Ordered.LOWEST_PRECEDENCE;
}
```

### SimpleLoggerAdvisor

内置的日志 Advisor，在调用前后分别记录 request 和 response：

![SimpleLoggerAdvisor 实现](/images/posts/spring-ai-advisor/logger-impl.png)

### SafeGuardAdvisor

内置的安全审查 Advisor，主要实现敏感词拦截功能：

![SafeGuardAdvisor](/images/posts/spring-ai-advisor/safeguard-advisor.png)

### BaseAdvisor

除上述 Advisor 外，其他自定义 Advisor 的基础接口。充分体现了 AOP 机制——在调用下一个 Advisor 前执行 `before`，获取结果后执行 `after`：

![BaseAdvisor](/images/posts/spring-ai-advisor/base-advisor.png)

例如之前常用的 `MessageChatMemoryAdvisor`，就是通过 before/after 实现记忆记录：

![MemoryAdvisor 实现](/images/posts/spring-ai-advisor/memory-advisor.png)

## 调用流程图

Spring AI 官方提供的 Advisor 调用流程：

![Advisor 调用流程图](/images/posts/spring-ai-advisor/flow-diagram.png)

## 小结

Advisor 机制是 Spring AI 的核心扩展点。通过它，可以以声明式的方式添加记忆、日志、安全审查等功能，而无需修改业务代码。理解 Advisor 的执行顺序和钩子机制，是掌握 Spring AI 的关键。
