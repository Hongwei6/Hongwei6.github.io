---
title: "LangChain4j 实现持久化记忆"
date: 2026-08-07T10:00:00+08:00
draft: false
description: "通过自定义 ChatMemoryStore 实现，将 LangChain4j 的对话记忆持久化到 Redis 等外部存储，并理解 AiServices / ChatMemory / ChatMemoryStore 的三层抽象。"
tags: ["LangChain4j", "Java", "AI", "LLM", "Redis"]
categories: ["技术"]
---

之前[介绍过](/post/langchain4j-high-level-api/)LangChain4j 基于内存的短期记忆（`MessageWindowChatMemory`）。但内存记忆在应用重启后会丢失，要做真正的持久化记忆，官方没有提供现成实现，需要开发者自行扩展。

好在扩展点已经预留好了——LangChain4j 内置了 `ChatMemoryStore` 接口，用于把消息的增删查交给外部存储。

## ChatMemoryStore 接口

该接口定义了消息的查、改、删三个方法：

```java
public interface ChatMemoryStore {
    List<ChatMessage> getMessages(Object memoryId);
    void updateMessages(Object memoryId, List<ChatMessage> messages);
    void deleteMessages(Object memoryId);
}
```

实现这个接口，就能用 MySQL、Redis 等任意存储来保存对话历史。下面是一个基于 Redis 的实现示例：

```java
@Component
public class RedisChatMemoryStore implements ChatMemoryStore {

    private final RedisTemplate<String, String> redisTemplate;

    public RedisChatMemoryStore(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @Override
    public List<ChatMessage> getMessages(Object memoryId) {
        String key = buildKey(memoryId);
        String json = redisTemplate.opsForValue().get(key);
        if (json == null || json.isEmpty()) {
            return Collections.emptyList();
        }
        return ChatMessageDeserializer.messagesFromJson(json);
    }

    @Override
    public void updateMessages(Object memoryId, List<ChatMessage> messages) {
        String key = buildKey(memoryId);
        String json = ChatMessageSerializer.messagesToJson(messages);
        redisTemplate.opsForValue().set(key, json);
    }

    @Override
    public void deleteMessages(Object memoryId) {
        redisTemplate.delete(buildKey(memoryId));
    }

    private String buildKey(Object memoryId) {
        return "langchain4j:chat-memory:" + memoryId;
    }
}
```

其中 `ChatMessageSerializer` / `ChatMessageDeserializer` 是 LangChain4j 提供的工具类，负责消息列表与 JSON 之间的互转。

使用时，构造一个 `MessageWindowChatMemory` 并把自定义 Store 设置进去即可：

```java
langChainMemoryAiService = AiServices.builder(LangChainMemoryAiService.class)
        .chatModel(chatModel)
        .chatMemoryProvider(memoryId -> MessageWindowChatMemory.builder()
                .id(memoryId)
                .maxMessages(10)
                .chatMemoryStore(redisChatMemoryStore)
                .build())
        .build();
```

## 三层抽象

LangChain4j 通过三层抽象实现多轮对话记忆，从上到下各司其职：

```
┌───────────────────────────────────────────────────────┐
│                    AiServices                         │
│  (代理层：自动管理消息的 add / messages 生命周期)        │
└─────────────────────────┬─────────────────────────────┘
                          │ 使用
┌─────────────────────────▼─────────────────────────────┐
│              ChatMemory (接口)                         │
│  实现类：MessageWindowChatMemory                       │
│  职责：控制记忆窗口大小、消息淘汰策略                      │
│  核心方法：add(message) / messages()                   │
└─────────────────────────┬─────────────────────────────┘
                          │ 委托持久化
┌─────────────────────────▼─────────────────────────────┐
│            ChatMemoryStore (接口)                      │
│  职责：消息的存储和读取（持久化层）                       │
│  核心方法：getMessages() / updateMessages()            │
└───────────────────────────────────────────────────────┘
```

- **AiServices**——代理层，自动管理消息的 `add` / `messages` 生命周期，开发者无需手动维护。
- **ChatMemory**——记忆层，控制记忆窗口大小和消息淘汰策略（如按消息条数、按 token 数）。
- **ChatMemoryStore**——持久化层，负责消息的存储与读取，是自定义实现的扩展点。

## 持久化的工作流程

LangChain4j 的 `AiServices` 在每次 LLM 调用时，内部对 `ChatMemoryStore` 的操作流程如下（伪代码）：

```java
// 1. 加载历史
List<ChatMessage> history = chatMemoryStore.getMessages(memoryId);

// 2. 加入 SystemMessage 与当前用户消息
history.add(systemMessage);
history.add(userMessage);

// 3. 持久化（含新用户消息）
chatMemoryStore.updateMessages(memoryId, history);

// 4. 调用 LLM
AiMessage response = llm.chat(history);

// 5. 加入 AI 回复
history.add(response);

// 6. 再次持久化（含 AI 回复）
chatMemoryStore.updateMessages(memoryId, history);
```

一次调用会对 Store 做两次 `updateMessages`：先存入用户消息，再在拿到回复后存入 AI 回复。这就要求 `ChatMemoryStore` 的实现必须保证：

- `getMessages()` 返回的列表在同一次调用内可变且一致。
- `updateMessages()` 必须真正落盘，否则下一次请求会丢失历史。

## 多会话隔离

通过 `chatMemoryProvider` 实现多会话隔离：

```java
.chatMemoryProvider(memoryId -> MessageWindowChatMemory.builder()
        .id(memoryId)                              // memoryId = conversationId
        .maxMessages(10)                           // 滑动窗口：最多保留10条
        .chatMemoryStore(databaseChatMemoryStore)  // 持久化委托
        .build())
```

配合 AiService 接口上的 `@MemoryId` 注解：

```java
@AiService
public interface LangChainMemoryAiService {
    String chatMemory(@MemoryId String memoryId, @UserMessage String userMessage);
}
```

LangChain4j 会自动把 `memoryId`（即 conversationId）传递给 `ChatMemoryStore`，作为存储的 Key 维度，从而实现每个会话拥有独立的持久化记忆。
