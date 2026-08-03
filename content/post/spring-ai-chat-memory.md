---
title: "Spring AI 对话记忆：从上下文窗口到持久化存储"
date: 2026-08-03
draft: false
description: "深入探讨 Spring AI 的对话记忆机制——短期记忆与长期记忆的区别、Message List 和 ChatMemory Advisor 两种实现方案、自动配置与多数据库持久化支持。"
tags: ["Spring AI", "Java", "AI", "对话记忆", "Chat Memory"]
categories: ["后端"]
cover: "/hero/hero-2.png"
---

对话记忆是 AI 应用中最基础也最容易忽略的一环。如果一个模型只能记住当前这句话，无法关联此前的交互，那就谈不上"对话"。Spring AI 在这方面提供了两套风格迥异的方案：直接操作 Message 列表，或者通过 Advisor 让框架代劳。两条路各有优劣，背后对应的是对记忆控制权的不同取舍。

## 短期记忆与长期记忆

理解对话记忆之前，有必要先区分两个概念。

**短期记忆**（上下文记忆）指的是单次会话中模型能"记住"的对话范围，受限于上下文窗口大小。每次请求时，系统会将当前问题连同此前若干轮对话历史一并发送给模型，使模型能在完整的对话上下文中生成回复。

短期记忆有两个天然局限：

- **窗口有限**：上下文窗口存在硬上限。对话轮次累积到一定程度后，最早的内容会从窗口中滑出。
- **会话隔离**：关闭对话或开启新会话后，之前的记忆全部丢失。

**长期记忆**（持久化记忆）则不同——它将重要信息保存在外部存储中（数据库、向量库等），下次对话时按需检索。举个典型场景：之前告诉过 AI "我对花生过敏"，一周后再问"推荐一些健康的零食"，AI 能从长期记忆中检索到这条信息，自动避开含花生的食品。

相比短期记忆，长期记忆的特点是持久、可扩展，但以摘要形式存储难免丢失细节，更适合保存用户偏好和行为记录，而不适用于依赖精确上下文的复杂推理。

长期记忆留到后续再展开，下面聚焦 Spring AI 中短期记忆的两种落地方式。

## 方案一：手动管理 Message 列表

回顾之前关于 Prompt 的介绍，Prompt 支持传入 `List<Message>`。把每一轮对话封装为对应的 Message 类型，逐轮追加到列表中，最终将完整的列表传给 Prompt——这就是最直接的方式。

```java
@GetMapping("/call1")
public String call1(String message) {
    List<Message> messages = new ArrayList<>();
    // 第一轮对话
    messages.add(new SystemMessage("你是一个旅行推荐师"));
    messages.add(new UserMessage("我想去新疆玩"));
    messages.add(new AssistantMessage("好的，我知道了，你要去新疆，请问你准备什么时候去"));
    messages.add(new UserMessage("我准备元旦的时候去玩"));
    messages.add(new AssistantMessage("好的，请问你想玩那些内容？"));
    messages.add(new UserMessage("我喜欢自然风光"));

    Prompt prompt = new Prompt(messages);
    return chatModel.call(prompt).getResult().getOutput().getText();
}
```

模型给出的回复能够关联前两轮对话的内容，说明 Message 列表确实实现了记忆效果。

在多轮交互的场景下，代码演变为"发送 → 收集回复 → 追加到列表 → 再发送"的循环：

```java
@GetMapping("/chat")
public String chat() {
    List<Message> messages = new ArrayList<>();

    // 第一轮对话
    messages.add(new SystemMessage("你是一个游戏设计师"));
    messages.add(new UserMessage("我想设计一个回合制游戏"));
    ChatResponse chatResponse = chatModel.call(new Prompt(messages));
    String content = chatResponse.getResult().getOutput().getText();
    messages.add(new AssistantMessage(content));

    // 第二轮对话
    messages.add(new UserMessage("能帮我结合一些二次元的元素吗?"));
    chatResponse = chatModel.call(new Prompt(messages));
    content = chatResponse.getResult().getOutput().getText();
    messages.add(new AssistantMessage(content));

    // 第三轮对话
    messages.add(new UserMessage("那如果主要是针对女性玩家的游戏呢?有什么需要改进的？"));
    chatResponse = chatModel.call(new Prompt(messages));
    content = chatResponse.getResult().getOutput().getText();

    return content;
}
```

每一轮先把用户消息放入列表，等模型返回后把 AssistantMessage 也追加上去，循环往复。

这种方式最大的优势是**完全可控**。想忽略某句话？直接从列表里移除。想在对话过长时做摘要压缩？在调用前对列表做一轮处理即可。记忆的每一个细节都在代码中显式呈现，不存在黑盒。

代价也很明显：每次请求都要把完整历史重新发送，上下文越长，Token 消耗越大。而且需要手动维护列表，代码量不小。

## 方案二：ChatMemory + Advisor

Spring AI 提供了一套更简洁的封装——通过 `ChatMemory` 和 Advisor 机制，让框架自动管理对话历史的存储与组装。

核心思路是：为每组对话分配一个 `chat_memory_conversation_id`，同一 ID 下的消息由框架自动聚合，通过 Advisor 在调用前注入到 Prompt 中。

```java
@RestController
@RequestMapping("/ai/memory")
public class ChatMemoryController implements InitializingBean {

    @Autowired
    private DashScopeChatModel chatModel;
    private ChatClient chatClient;

    @GetMapping("/chat1")
    public Flux<String> chat1(String message, String chatId, HttpServletResponse response) {
        response.setCharacterEncoding("UTF-8");
        return chatClient
                .prompt()
                .user(message)
                .advisors(spec -> spec.param(ChatMemory.CONVERSATION_ID, chatId))
                .stream().content();
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        ChatMemory chatMemory = MessageWindowChatMemory.builder()
                .maxMessages(10)
                .build();
        this.chatClient = ChatClient.builder(chatModel)
                .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
                .defaultOptions(
                        DashScopeChatOptions.builder()
                                .withTopP(0.7)
                                .build()
                )
                .build();
    }
}
```

这里用到了 `MessageWindowChatMemory`，它是 Spring AI 框架内置的基于窗口的记忆实现。核心逻辑很直白：只保留最近 N 条交互消息，超出窗口上限的消息自动丢弃。需要注意，`maxMessages` 统计的是所有类型的 Message（UserMessage、AssistantMessage 都计入），而非仅用户消息。

Advisor 有两个具体实现：

- **MessageChatMemoryAdvisor**：将用户问题和模型回答追加到历史记录中，在每次调用时自动注入，实现上下文记忆。这是最常用的选择。
- **PromptChatMemoryAdvisor**：MessageChatMemoryAdvisor 的增强版。当模型不支持 SystemMessage 之外的 messages 参数时，它会改写 SystemPrompt，把每轮对话的输入和输出嵌入其中。适用场景较窄，但填补了兼容性缺口。

## 方案对比

两种方案代表了两种设计哲学。

方案二（ChatMemory + Advisor）胜在简洁：代码量少，借助框架自动管理，每次请求只携带窗口内的最近消息，Token 消耗可控。适合大多数标准场景。

方案一（Message 列表）的优势在于**控制力**。因为每一轮的消息列表完全由业务代码构造，可以在任意节点介入——选择性遗忘某段对话、在发送前对历史做摘要压缩、甚至根据业务规则动态调整窗口策略。这些在 Advisor 方案中很难做到，因为记忆的维护过程对业务代码是不透明的。

简单场景用 Advisor，需要精细控制时用 Message 列表。

## 自动配置与持久化

`ChatMemory` 也无需手动 `new`。Spring AI 提供了自动配置 Starter：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-autoconfigure-model-chat-memory</artifactId>
    <version>1.1.0</version>
</dependency>
```

引入后，框架会自动注册一个 `ChatMemory` Bean，默认是基于内存的实现。代码中可以直接 `@Autowired` 注入。

内存记忆的局限在于应用重启后全部丢失。对于需要跨会话持久化的场景，Spring AI 支持多种数据库后端。

关系型数据库：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
```

支持 PostgreSQL、MySQL/MariaDB、SQL Server、HSQLDB。

此外还有 Cassandra 和 Neo4j 的适配：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-cassandra</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-neo4j</artifactId>
</dependency>
```

这就是对话记忆的另一条路径——从内存到持久化，从单一会话到跨会话检索。后续再展开长期记忆的具体实现。
