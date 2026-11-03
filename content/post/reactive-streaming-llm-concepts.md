---
title: "拆一个大模型流式对话系统：Reactor 六个核心概念的落地位置"
date: 2026-11-03T13:00:00+08:00
draft: false
description: "用 Mono、Flux、Sinks、Disposable、flatMap、Backpressure 六个 Reactor 概念，还原一个支持流式输出、任务管理、停止生成的智能对话系统的数据链路。"
tags: ["Java", "Reactor", "Spring AI", "大模型"]
categories: ["后端"]
---

写完一个带流式输出、任务管理、停止生成的对话系统后回头看代码，反复出现的其实就是六个 Reactor 概念。逐个对着代码讲一下它们各自站在数据链路的哪个位置。

先给全景：

| 概念 | 含义 | 在系统中的位置 |
| --- | --- | --- |
| Mono | 单值异步结果 | 非流式调用 / 启动后台任务 |
| Flux | 连续数据流 | token 流式输出 |
| Sinks | 手动向流写数据 | 推 token 到 SSE |
| Disposable | 订阅句柄 | 停止生成 |
| flatMap | 元素到异步流的转换 | 并发调用工具/模型 |
| Backpressure | 流速控制 | 防 token 堆积 |

## Mono：两种用法

第一种是替代阻塞返回。传统 MVC 写法：

```java
public String chat(String prompt) {
    return llmClient.call(prompt);
}
```

响应式项目里返回 `Mono<String>`——"未来某个时间点会有一个 String"。调非流式接口（一次性返回完整回答）时就是这样。

第二种更常被忽视：**用 Mono 启动异步任务**。智能体的执行逻辑（工具调用、搜索、往 sink 写 token）不是一个"返回值"的活，需要在后台线程跑起来：

```java
Mono.fromRunnable(() -> {
    agent.execute(conversationId, question)
         .subscribe(token -> sink.tryEmitNext(token));
})
.subscribeOn(Schedulers.boundedElastic())
.subscribe();
```

`fromRunnable` 包装任务，`subscribeOn` 指定线程池，`subscribe` 触发执行。这里 Mono 的价值只是"以响应式的方式点火"，任务本身的结果走的是 sink。

## Flux：token 流的载体

模型不是一次吐全文，而是一个个 token 往外蹦。这个持续产生数据的过程就是 `Flux<String>`：

```java
chatModel.stream(prompt)
        .subscribe(token -> sink.tryEmitNext(token));
```

整条链路：`大模型 → Flux → SSE → 前端`。Mono 对应单次返回，Flux 对应流式输出，选择依据就这么简单。

## Sinks：数据不是只能来自 API

很多场景下流的数据源不是现成的，需要自己往里写。`Sinks.Many` 就是一个可主动写入的流通道：

```java
Sinks.Many<String> sink = Sinks.many().unicast().onBackpressureBuffer();
```

生产侧 `sink.tryEmitNext(token)` 写入，消费侧 `sink.asFlux()` 转成 Flux 返回给前端。链路变成：

```
LLM token → sink.tryEmitNext() → Flux → SSE → 前端
```

Sink 负责生产、Flux 是流本身、Subscriber 消费。Agent 后台线程和 SSE 响应线程之间，靠的就是这个 sink 解耦。

## Disposable：停止生成的关键

`subscribe()` 的返回值不是摆设，它是当前流式任务的控制开关。把它注册到任务管理器：

```java
Disposable disposable = chatModel.stream(prompt)
        .subscribe(token -> sink.tryEmitNext(token));
taskManager.registerTask(conversationId, disposable);
```

用户点"停止生成"时调 `disposable.dispose()`，会取消订阅、停掉 token 生成、终止后台推理。不保存这个句柄，前端停止按钮就没有真正的后端语义——这是流式系统里最容易漏的一环。

## flatMap：流里的异步展开

`flatMap` 把一个元素转换成一个新的异步流并合并结果：

```java
Flux<String> result = prompts.flatMap(prompt -> llmClient.call(prompt));
```

典型场景：一个问题拆成多个子任务，flatMap 并发执行（查多个工具、调多个 API），最后汇总。如果这里用 `map`，返回的会是 `Mono<String>` 的流而不是结果本身——这是初写响应式代码最常犯的错误。

## Backpressure：消费不过来怎么办

模型每秒出 100 个 token，前端只能消化 20 个，不做控制就是内存持续增长直到积压崩溃。响应式框架的解法是背压策略：

| 策略 | 行为 |
| --- | --- |
| Buffer | 先进缓冲区 |
| Drop | 丢弃 |
| Latest | 只留最新 |

创建 sink 时的 `onBackpressureBuffer()` 选的就是 Buffer——token 不能丢，大模型系统里这几乎总是正确选择。

## 小结

这六个概念拼起来就是完整的流式对话系统骨架：Mono 点火后台任务，Flux/Sinks 搭 token 通道，Disposable 管生命周期，flatMap 做并发展开，Backpressure 兜底流速。响应式在大模型应用里成为主流选择，正是因为这套原语刚好覆盖了 token 级输出、异步工具调用和任务管理这三件事。
