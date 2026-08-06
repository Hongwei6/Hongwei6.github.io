---
title: "LangChain4j 高层次 API 实战"
date: 2026-08-06T14:00:00+08:00
draft: false
description: "使用 LangChain4j 高层次 API（AiServices / @AiService）优雅地实现对话、流式输出、提示词模板、结构化输出、对话记忆与工具调用。"
tags: ["LangChain4j", "Java", "AI", "LLM"]
categories: ["技术"]
---

低层次 API（见[上一篇](/post/langchain4j-low-level-api/)）能力齐全但样板代码多。LangChain4j 的高层次 API 围绕 `dev.langchain4j.service.AiServices`（构建器）和 `dev.langchain4j.service.spring.AiService`（注解）展开，开发者只需定义接口，由框架代理完成实际的模型调用。

引入 `langchain4j-spring-boot-starter` 才能使用 `@AiService` 注解：

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-spring-boot-starter</artifactId>
    <version>1.8.0-beta15</version>
</dependency>
```

## 对话

定义一个接口，用 `@AiService` 声明，方法入参和返回值都是 `String`：

```java
@AiService
public interface LangChainAiService {
    String chat(String userMessage);
}
```

`@AiService` 会把该接口暴露成一个 Spring Bean，在需要对话的地方直接调用即可：

```java
@Autowired
private LangChainAiService aiService;

@RequestMapping("/chat")
public String chat(HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");
    return aiService.chat("日本都有哪些美食？");
}
```

`chat` 方法底层会自动路由到 `ChatModel#chat(ChatRequest)` 完成对话：

![@AiService 方法的调用路由](/images/posts/langchain4j-high-level-api/aiservice-chat-dispatch.png)

方法名也不必叫 `chat`，框架按入参/出参类型匹配，方法名可以任意：

![方法名可任意命名](/images/posts/langchain4j-high-level-api/aiservice-method-name-flexible.png)

## 流式输出

把方法返回值声明为 `Flux<String>` 即可实现流式输出：

```java
@AiService
public interface LangChainAiService {
    String hollis666(String userMessage);
    Flux<String> chatStream(String userMessage);
}
```

直接使用会报错，提示需要引入 reactor 模块：

```
dev.langchain4j.service.IllegalConfigurationException:
Please import langchain4j-reactor module if you wish to use Flux<String> as a method return type
```

补充 `langchain4j-reactor` 依赖后即可正常流式输出：

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-reactor</artifactId>
    <version>1.8.0-beta15</version>
</dependency>
```

## 提示词模板

LangChain4j 提供内置注解为方法附加默认的系统提示词、用户提示词。用户提示词支持模板，变量用 `{{}}` 表示：

```java
@AiService
public interface LangChainAiService {
    @SystemMessage("你是一个毒舌博主，擅长怼人")
    @UserMessage("针对用户的内容：{{topic}}，先复述一遍他的问题，然后再回答")
    Flux<String> chatStream(String topic);
}
```

提示词模板也支持从资源文件加载：

```java
@UserMessage(fromResource = "your-prompt-template.txt")
String chat(String topic);
```

## 结构化输出

将方法返回值声明为 POJO，框架会自动把模型输出解析为对应对象：

```java
@AiService
public interface LangChainAiService {
    @UserMessage("请帮我推荐1本java相关的书")
    @SystemMessage("你是一个专业的图书推荐人员")
    Book getBooks();
}
```

```java
@RequestMapping("/structure1")
public String structure1(HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");
    Book books = aiService.getBooks();
    return books.toString();
}
```

运行后框架通过提示词约束模型输出为 JSON，再反序列化成 `Book` 对象：

![结构化输出 POJO 结果](/images/posts/langchain4j-high-level-api/structured-output-pojo.png)

不过这种提示词方式**不支持 `List<POJO>`**，返回 List 会报错：

![返回 List 不支持](/images/posts/langchain4j-high-level-api/structured-output-list-unsupported.png)

相关限制可参考 issue：https://github.com/langchain4j/langchain4j/issues/3264

## 对话记忆

高层次 API 实现带记忆隔离（按对话 ID 区分）的对话记忆，需要用 `@MemoryId` 标注记忆标识：

```java
package org.example.cpuai.service;

import dev.langchain4j.service.MemoryId;
import dev.langchain4j.service.UserMessage;
import dev.langchain4j.service.spring.AiService;

@AiService
public interface LangChainMemoryAiService {
    String chatMemory(@MemoryId String memoryId, @UserMessage String userMessage);
}
```

`@MemoryId` 用于标注会话标识，按其值隔离不同对话的记忆。注意，带 `@MemoryId` 的接口不能直接 `@Autowired` 注入，会报：

```
In order to use @MemoryId, please configure the ChatMemoryProvider on the '...'.
```

错误信息明确：使用 `@MemoryId` 必须提供一个 `ChatMemoryProvider`。用 `AiServices` 构建器配置即可：

```java
langChainMemoryAiService = AiServices.builder(LangChainMemoryAiService.class)
        .chatModel(chatModel)
        .streamingChatModel(streamingChatModel)
        .chatMemoryProvider(memoryId -> MessageWindowChatMemory.withMaxMessages(10))
        .build();
```

这里用 `AiServices` 构造服务实例，并配置了一个 `MessageWindowChatMemory`（最多保留 10 条消息）。也可以直接声明一个 `ChatMemoryProvider` Bean，框架会自动装配，正常 `@Autowired` 注入即可。

### 记忆失效的坑

要让记忆持久生效，`langChainMemoryAiService` 实例必须只初始化一次。下面这种写法——每次请求都新建一个实例——会导致记忆无法保存：

```java
// ❌ 每次请求都新建实例，记忆丢失
@RequestMapping("/memoryChat")
public String memoryChat(HttpServletResponse response, String msg) {
    response.setCharacterEncoding("UTF-8");
    LangChainMemoryAiService langChainMemoryAiService = AiServices.builder(LangChainMemoryAiService.class)
            .chatModel(chatModel)
            .streamingChatModel(streamingChatModel)
            .chatMemoryProvider(memoryId -> MessageWindowChatMemory.withMaxMessages(10))
            .build();
    return langChainMemoryAiService.chatMemory("memoryId", msg);
}
```

原因：每次访问 `memoryChat` 都新建了 `LangChainMemoryAiService`，其 `chatMemoryProvider` 也是新的，自然无法保留之前的记忆。

正确做法是让 `langChainMemoryAiService` 只初始化一次（例如在 `InitializingBean` 的回调中）：

```java
@RestController
@RequestMapping("/langchain")
public class LangChainController implements InitializingBean {

    private LangChainMemoryAiService langChainMemoryAiService;

    @RequestMapping("/memoryChat")
    public String memoryChat(HttpServletResponse response, String msg, String memoryId) {
        response.setCharacterEncoding("UTF-8");
        return langChainMemoryAiService.chatMemory(memoryId, msg);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        langChainMemoryAiService = AiServices.builder(LangChainMemoryAiService.class)
                .chatModel(chatModel)
                .streamingChatModel(streamingChatModel)
                .chatMemoryProvider(memoryId -> MessageWindowChatMemory.withMaxMessages(10))
                .build();
    }
}
```

### 记忆隔离的验证

第一次对话「我叫Hollis」，指定 `memoryId=1234`，模型正常回应：

![同一 memoryId 第一次对话](/images/posts/langchain4j-high-level-api/memory-same-id-1.png)

此时发送的请求体只包含这一条 user 消息：

```json
{
  "model": "deepseek-v3",
  "messages": [
    { "role": "user", "content": "我叫Hollis" }
  ]
}
```

第二次对话「我叫什么」，仍指定 `memoryId=1234`，模型正确回答出「Hollis」：

![同一 memoryId 第二次对话](/images/posts/langchain4j-high-level-api/memory-same-id-2.png)

此时请求体已带上完整的历史上下文——LangChain4j 把之前的提问和响应都加入了 messages：

```json
{
  "model": "deepseek-v3",
  "messages": [
    { "role": "user", "content": "我叫Hollis" },
    { "role": "assistant", "content": "你好，Hollis！很高兴认识你～ 😊 ..." },
    { "role": "user", "content": "我叫什么" }
  ]
}
```

换成不同的 `memoryId` 再问「我叫什么」，由于记忆隔离，该会话没有历史上下文，模型回答「你还没有告诉我你的名字」：

![不同 memoryId 隔离生效](/images/posts/langchain4j-high-level-api/memory-diff-id.png)

从对话日志可以看出：记忆在内存中维护，`memoryId` 相同时，LangChain4j 会把之前的对话内容（提问 + 响应）一并加入上下文提交给 LLM；`memoryId` 不同则各自独立。

## 工具调用

上一篇用低层次 API 做工具调用代码相当繁琐，高层次 API 把这些全部封装了。用 `AiServices` 指定工具列表，直接调用 chat 即可：

```java
@RequestMapping("/toolCalling")
public String toolCalling(HttpServletResponse response, String msg) {
    response.setCharacterEncoding("UTF-8");
    LangChainAiService langChainAiService1 = AiServices.builder(LangChainAiService.class)
            .tools(new TemperatureTools())
            .chatModel(chatModel)
            .build();
    return langChainAiService1.chat("2025年11月11日，杭州的气温怎样？");
}
```

框架自动完成了工具调用，无需手动编排，与 Spring AI 体验一致：

![高层次 API 自动工具调用](/images/posts/langchain4j-high-level-api/tool-calling-auto.png)

从底层日志可以看到完整的两次交互：

**第一次请求**——把工具定义随消息一起发给 LLM，LLM 返回要调用的工具名和参数：

```json
{
  "model": "deepseek-v3",
  "messages": [
    { "role": "user", "content": "2025年11月11日，杭州的气温怎样？" }
  ],
  "tools": [{
    "type": "function",
    "function": {
      "name": "getTemperatureByCityAndDate",
      "description": "Get temperature by city and date",
      "parameters": { "type": "object", "properties": { "city": {"type":"string"}, "date": {"type":"string"} }, "required": ["city", "date"] }
    }
  }]
}
```

LLM 响应中给出 `tool_calls`，要求调用 `getTemperatureByCityAndDate`，参数为 `{"city":"杭州","date":"2025-11-11"}`。

**框架执行工具**（控制台输出 `getTemperatureByCityAndDate invoke...`），拿到结果「23摄氏度」后发起**第二次请求**——带上 user、assistant(tool_calls)、tool(结果) 三类消息：

```json
{
  "messages": [
    { "role": "user", "content": "2025年11月11日，杭州的气温怎样？" },
    { "role": "assistant", "tool_calls": [{ "id": "call_445b...", "function": { "name": "getTemperatureByCityAndDate", "arguments": "{\"city\":\"杭州\",\"date\":\"2025-11-11\"}" } }] },
    { "role": "tool", "tool_call_id": "call_445b...", "content": "23摄氏度" }
  ]
}
```

LLM 最终返回文本：「2025年11月11日，杭州的气温预计为23摄氏度。」

整个流程与上一篇手动实现的一致，只是 LangChain4j 把「发起请求 → 解析工具调用 → 执行工具 → 回传结果 → 再次请求」全部封装好了。

### 按需加载工具

工具过多时一次性全部传给 LLM 会占用上下文。`AiServices` 支持配置一个 `ToolProvider`，按条件动态组装工具。以下是官方示例——仅在用户消息包含 "booking" 时才注入 `get_booking_details` 工具：

```java
ToolProvider toolProvider = (toolProviderRequest) -> {
    if (toolProviderRequest.userMessage().singleText().contains("booking")) {
        ToolSpecification toolSpecification = ToolSpecification.builder()
                .name("get_booking_details")
                .description("返回预订详情")
                .parameters(JsonObjectSchema.builder()
                        .addStringProperty("bookingNumber")
                        .build())
                .build();
        return ToolProviderResult.builder()
                .add(toolSpecification, toolExecutor)
                .build();
    } else {
        return null;
    }
};

Assistant assistant = AiServices.builder(Assistant.class)
        .chatLanguageModel(model)
        .toolProvider(toolProvider)
        .build();
```

`ToolProvider` 根据当前请求的消息内容决定返回哪些工具，从而避免无关工具消耗上下文 token。
