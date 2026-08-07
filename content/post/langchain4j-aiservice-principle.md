---
title: "LangChain4j @AiService 实现原理"
date: 2026-08-07T09:00:00+08:00
draft: false
description: "剖析 LangChain4j @AiService 的实现原理：Bean 定义替换、JDK 动态代理、方法调用拦截与流式输出分发。"
tags: ["LangChain4j", "Java", "AI", "LLM", "Spring"]
categories: ["技术"]
---

[上一篇](/post/langchain4j-high-level-api/)介绍了 `@AiService` 的高层次用法——只需定义接口，框架就会代理完成所有模型调用。这篇深入源码，拆解它背后的实现机制。整个过程分为三步：**注册阶段**替换 Bean 定义、**实例化阶段**创建代理对象、**调用阶段**拦截方法并分发到具体的模型执行器。

## 注册阶段：替换 Bean 定义

当接口被 `@AiService` 标注后，应用启动时会在 `AiServicesAutoConfiguration#aiServicesRegisteringBeanFactoryPostProcessor` 这个 Bean 的生命周期中被扫描到：

![扫描 @AiService 的 BeanFactoryPostProcessor](/images/posts/langchain4j-aiservice-principle/register-bean-postprocessor.png)

对每个标注了 `@AiService` 的接口，框架会**动态替换其 Bean 定义**，使其由 `AiServiceFactory` 创建实例。

首先是创建一个 `BeanDefinition`，将 Bean 的类设置为 `AiServiceFactory`，并把原始接口类型作为构造参数传入：

```java
GenericBeanDefinition aiServiceBeanDefinition = new GenericBeanDefinition();
aiServiceBeanDefinition.setBeanClass(AiServiceFactory.class);
aiServiceBeanDefinition.getConstructorArgumentValues()
        .addGenericArgumentValue(aiServiceClass);
```

接着通过一系列 `addBeanReference` 方法，为每个 AI 服务配置依赖组件（如 `ChatModel`、`ChatMemory`）：

![配置依赖组件](/images/posts/langchain4j-aiservice-principle/add-bean-reference.png)

最后将原始的 `@AiService` 接口的 Bean 定义替换为新的 `AiServiceFactory` 定义：

```java
BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
registry.removeBeanDefinition(aiService);
registry.registerBeanDefinition(lowercaseFirstLetter(aiService), aiServiceBeanDefinition);
```

这一步确保 Spring 容器在初始化时，统一用 `AiServiceFactory` 来生成所有 `@AiService` 接口的具体实现，而非走默认的接口实例化路径。

## 实例化阶段：创建代理对象

当应用中真正需要用到某个 `@AiService` 服务的 Bean（例如 `LangChainAiService`）时，Spring 会回调 `AiServiceFactory#getObject`。该方法会进一步调用 `DefaultAiServices#build` 来构造一个代理对象：

![AiServiceFactory#getObject 构造代理](/images/posts/langchain4j-aiservice-principle/factory-get-object-build.png)

`DefaultAiServices#build` 主要做两件事：

1. 创建一个代理对象
2. 返回该代理对象

其中创建代理时使用了 JDK 动态代理 `Proxy#newProxyInstance`，并传入一个 `InvocationHandler`。这个处理器会拦截对代理对象的所有方法调用，转而进入自定义的 `invoke` 方法。

## 调用阶段：代理方法分发

当业务代码调用 `LangChainAiService#hollis666` 这样的方法时，代理对象的 `invoke` 方法被触发。其调用栈如下：

![invoke 调用栈](/images/posts/langchain4j-aiservice-principle/invoke-call-stack.png)

核心是通过一个 `ChatExecutor` 来执行具体调用。对于同步方法，最终会路由到 `SynchronousChatExecutor#execute`：

![SynchronousChatExecutor#execute](/images/posts/langchain4j-aiservice-principle/synchronous-chat-executor.png)

这里的执行逻辑本质上就是在调用 `chatModel`——也就是说，`@AiService` 接口方法的底层，最终仍由 `ChatModel` 完成实际的大模型交互，与[低层次 API](/post/langchain4j-low-level-api/)用的是同一套核心组件。

### 流式输出的分发

流式输出的实现与同步调用同源。区别在于 `InvocationHandler#invoke` 中会先判断方法的返回值类型：如果是流式返回（如 `Flux<String>`），则走流式调用链：

![流式调用链](/images/posts/langchain4j-aiservice-principle/streaming-invoke-chain.png)

最终同样落到 `streamingChatModel` 的 chat 方法：

![流式调用落到 streamingChatModel](/images/posts/langchain4j-aiservice-principle/streaming-chat-model-call.png)

## 小结

`@AiService` 的整套机制可以归纳为一条链路：

> `@AiService` 接口 → 注册期替换为 `AiServiceFactory` 的 Bean 定义 → 实例化期由 `FactoryBean#getObject` 构建代理 → 调用期 `InvocationHandler` 拦截方法 → 按返回值类型分发到 `SynchronousChatExecutor`（同步）或流式执行器 → 最终调用 `ChatModel` / `StreamingChatModel`

可见，所谓「高层次 API」并非新造了一套调用能力，而是用 **Spring 的 Bean 定义替换 + JDK 动态代理** 两个标准机制，把低层次的 `ChatModel` 调用封装成了声明式的接口调用。理解这条链路，既能帮助调试接口方法未按预期工作的问题，也能解释为什么不同返回值类型（`String` / `POJO` / `Flux`）会走到不同的执行器。
