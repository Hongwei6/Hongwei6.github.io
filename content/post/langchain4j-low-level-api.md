---
title: "LangChain4j 低层次 API 实战"
date: 2026-08-06T13:00:00+08:00
draft: false
description: "使用 LangChain4j 低层次 API 实现 ChatModel 调用、流式输出、对话记忆、结构化输出与工具调用，并理解 function call 的底层原理。"
tags: ["LangChain4j", "Java", "AI", "LLM"]
categories: ["技术"]
---

LangChain4j 的低层次 API 提供了对大模型交互的直接控制能力，涵盖 ChatModel、流式输出、对话记忆、结构化输出和工具调用。相比高层次抽象（AI Services），它更底层、可定制性更强，也更能帮助开发者理解 LLM 应用的工作原理。

## ChatModel

ChatModel 和 LanguageModel 是 LangChain4j 中与模型交互的核心入口，对应 Spring AI 中的 ChatModel。除这两者外，LangChain4j 还支持以下模型类型：

- `EmbeddingModel`——将文本转换为向量（Embedding）。
- `ImageModel`——生成和编辑图像。
- `ModerationModel`——检查文本是否包含有害内容。
- `ScoringModel`——对多个文本片段针对查询进行评分排序，衡量与查询的相关性，是 RAG 场景的关键组件。

ChatModel 是首选，因为 LanguageModel 的功能过于简单。引入 `langchain4j-open-ai` 后，会自动提供一个 `ChatModel` 的实现 `OpenAiChatModel`，可以直接注入到 Bean 中调用。它提供了几个重载的 chat 方法：

```java
ChatResponse chat(ChatRequest chatRequest);
String chat(String userMessage);
ChatResponse chat(ChatMessage... messages);
ChatResponse chat(List<ChatMessage> messages);
ChatResponse doChat(ChatRequest chatRequest);
```

各方法主要在入参上有差别：最简单的 `String` 入参直接传入用户提示词即可；`ChatMessage` 和 `ChatRequest` 则提供更细粒度的控制。

### ChatMessage

ChatMessage 用于承载对话上下文，对应 Spring AI 中的 Message，是实现对话记忆的基础。它按消息来源区分了多种类型：

![ChatMessage 类型](/images/posts/langchain4j-low-level-api/chatmessage-types.png)

- `UserMessage`——来自用户的消息（最终用户或应用程序本身）。
- `AiMessage`——AI 生成的消息，作为对已发送消息的回应。
- `ToolExecutionResultMessage`——`ToolExecutionRequest` 的执行结果。
- `SystemMessage`——来自系统的消息。
- `CustomMessage`——包含任意属性的自定义消息。

### ChatRequest

当需要在模型交互时设置参数（如 `temperature`、`topP`、`topK` 等），使用 ChatRequest：

```java
ChatRequest chatRequest = ChatRequest.builder()
        .messages(...)
        .modelName(...)
        .temperature(...)
        .topP(...)
        .topK(...)
        .frequencyPenalty(...)
        .presencePenalty(...)
        .maxOutputTokens(...)
        .stopSequences(...)
        .toolSpecifications(...)
        .toolChoice(...)
        .responseFormat(...)
        .parameters(...)
        .build();
```

## 流式输出

低层次 API 做流式输出相对繁琐。前面提到的 ChatModel 没有流式方法（与 Spring AI 不同），流式输出需要使用专门的 `StreamingChatModel`，例如 `OpenAiStreamingChatModel`。

它不仅需要单独的模型类，还需要单独的环境变量配置：

```properties
langchain4j.open-ai.streaming-chat-model.api-key={YOUR_KEY}
langchain4j.open-ai.streaming-chat-model.model-name=qwen-max-latest
langchain4j.open-ai.streaming-chat-model.base-url=https://dashscope.aliyuncs.com/compatible-mode/v1
```

之前配置的 `langchain4j.open-ai.chat-model.*` 对流式输出不生效，必须单独配置 `langchain4j.open-ai.streaming-chat-model.*`。

即便配置完成，`OpenAiStreamingChatModel` 用起来仍不方便——它的 chat 方法没有 `Flux<String>` 返回类型，返回值都是 `void`：

![StreamingChatModel 的 chat 方法](/images/posts/langchain4j-low-level-api/streaming-chat-methods.png)

要实现流式返回给前端，需要自行转换成 Flux。这些 chat 方法支持传入一个 `StreamingChatResponseHandler`：

```java
public interface StreamingChatResponseHandler {
    void onPartialResponse(String partialResponse);
    void onCompleteResponse(ChatResponse completeResponse);
    void onError(Throwable error);
}
```

通过实现该接口，可以为不同事件定义回调：

- `onPartialResponse(String)`——生成下一个部分响应时触发。部分响应可由单个或多个 token 组成，可以在 token 可用时立即推送到前端。
- `onCompleteResponse(ChatResponse)`——LLM 完成生成时触发，`ChatResponse` 包含完整响应（`AiMessage`）及元数据 `ChatResponseMetadata`。
- `onError(Throwable)`——发生错误时触发。

借助 Flux，将每次 LLM 返回的内容作为流的一部分推送给前端：

```java
@Autowired
OpenAiStreamingChatModel streamingChatModel;

@RequestMapping("/streamHello")
public Flux<String> streamHello(HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");
    Flux<String> flux = Flux.create(fluxSink -> {
        streamingChatModel.chat("你好,你是谁？", new StreamingChatResponseHandler() {
            @Override
            public void onPartialResponse(String partialResponse) {
                fluxSink.next(partialResponse);
            }

            @Override
            public void onCompleteResponse(ChatResponse completeResponse) {
                fluxSink.complete();
            }

            @Override
            public void onError(Throwable error) {
                fluxSink.error(error);
            }
        });
    });
    return flux;
}
```

## 对话记忆

前面提到，调用 ChatModel 的 chat 方法时可以传入 `List<ChatMessage>` 作为参数之一，这本身就构成了一种记忆机制——维护整个对话过程，每次调用时把历史消息带给大模型：

```java
@RequestMapping("/memory")
public String memory(HttpServletResponse response) {
    List<ChatMessage> messages = new ArrayList<>();

    // 第一轮对话
    messages.add(systemMessage("你是一个AI助手"));
    messages.add(userMessage("我叫Hong，是一个程序员"));
    AiMessage answer = chatModel.chat(messages).aiMessage();
    messages.add(answer);

    // 第二轮对话
    messages.add(userMessage("Hong是干什么的?"));
    AiMessage answer1 = chatModel.chat(messages).aiMessage();
    messages.add(answer1);

    // 第三轮对话
    messages.add(userMessage("我是谁？"));
    AiMessage answer2 = chatModel.chat(messages).aiMessage();
    return answer2.text();
}
```

模型能正确记住前两轮的上下文，第三轮被问「我是谁」时仍能回答出「Hong，程序员」，说明已经具备了记忆能力。

手动维护 `List<ChatMessage>` 可行，但如果需要按固定轮次或 token 数限制记忆，更合适的方式是用 `ChatMemory`。LangChain4j 提供了两种实现：一种按对话轮次限制（`MessageWindowChatMemory`），一种按 token 限制（`TokenWindowChatMemory`）。

![ChatMemory 类型](/images/posts/langchain4j-low-level-api/chatmemory-types.png)

用 ChatMemory 维护记忆的写法：

```java
@RequestMapping("/memory1")
public String memory1(HttpServletResponse response) {
    ChatMemory chatMemory = MessageWindowChatMemory.withMaxMessages(10);

    // 第一轮对话
    chatMemory.add(systemMessage("你是一个AI助手"));
    chatMemory.add(userMessage("我叫Hollis，是一个程序员"));
    AiMessage answer = chatModel.chat(chatMemory.messages()).aiMessage();
    chatMemory.add(answer);

    // 第二轮对话
    chatMemory.add(userMessage("Hollis是干什么的?"));
    AiMessage answer1 = chatModel.chat(chatMemory.messages()).aiMessage();
    chatMemory.add(answer1);

    // 第三轮对话
    chatMemory.add(userMessage("我是谁？"));
    AiMessage answer2 = chatModel.chat(chatMemory.messages()).aiMessage();
    return answer2.text();
}
```

## 结构化输出

低层次 API 也能实现结构化输出，但实际效果并不理想。官方示例通过 `ResponseFormat` 配合 `JsonSchema` 定义期望的输出结构：

```java
@RequestMapping("/structure")
public String structure() {
    ResponseFormat responseFormat = ResponseFormat.builder()
            .type(JSON) // 类型可以是 TEXT（默认）或 JSON
            .jsonSchema(JsonSchema.builder()
                    .name("Person") // OpenAI 要求为 schema 指定名称
                    .rootElement(JsonObjectSchema.builder()
                            .addStringProperty("name")
                            .addIntegerProperty("age")
                            .addNumberProperty("height")
                            .addBooleanProperty("married")
                            .required("name", "age", "height", "married")
                            .build())
                    .build())
            .build();

    ChatRequest chatRequest = ChatRequest.builder()
            .responseFormat(responseFormat)
            .messages(UserMessage.from("""
                John is 42 years old and lives an independent life.
                He stands 1.75 meters tall and carries himself with confidence.
                Currently unmarried, he enjoys the freedom to focus on his personal goals and interests.
                """))
            .build();

    return chatModel.chat(chatRequest).aiMessage().text();
}
```

直接运行会报 `400 Bad Request`：

```
'messages' must contain the word 'json' in some form, to use 'response_format' of type 'json_object'.
```

问题不在 LangChain4j，而在阿里云百炼的限制——调用模型时必须在提示词中明确出现 `JSON` 字样，否则拒绝 `json_object` 类型的 `response_format`：

![百炼的 JSON 提示词限制](/images/posts/langchain4j-low-level-api/bailian-json-limit.png)

在用户提示词末尾加上 `output in json format` 即可绕过：

```java
ChatRequest chatRequest = ChatRequest.builder()
        .responseFormat(responseFormat)
        .messages(UserMessage.from("""
            John is 42 years old and lives an independent life.
            He stands 1.75 meters tall and carries himself with confidence.
            Currently unmarried, he enjoys the freedom to focus on his personal goals and interests.
            output in json format
            """))
        .build();
```

但即便如此，模型实际输出的 JSON 与 schema 并不完全一致——要求的 `married` 字段并未出现，多出了 `lifestyle`、`marital_status` 等字段。值得注意的是，Spring AI 在调用时没有遇到这个限制，说明它在底层已将 JSON 相关的提示词预置到了 prompt 中。

## 工具调用

工具调用同样借助 ChatRequest，通过 `List<ToolSpecification> toolSpecifications` 告知大模型有哪些可用工具。`ToolSpecifications.toolSpecificationsFrom` 接受一个定义了工具的类，从中提取工具描述：

![ToolSpecifications.toolSpecificationsFrom](/images/posts/langchain4j-low-level-api/toolspec-from.png)

定义工具时使用 `@Tool` 描述工具作用、`@P` 描述入参，与 Spring AI 的工具定义方式类似——LLM 需要这些说明才能判断工具的用途和调用方式：

```java
public class TemperatureTools {
    @Tool(value = "Get temperature by city and date", name = "getTemperatureByCityAndDate")
    public String getTemperatureByCityAndDate(
            @P("city for get Temperature") String city,
            @P("date for get Temperature") String date) {
        System.out.println("getTemperatureByCityAndDate invoke...");
        return "23摄氏度";
    }
}
```

完整的工具调用流程：

1. 构造 `toolSpecifications`，连同 userMessage 一起放入 `ChatRequest` 发起请求。
2. 将模型返回的 `aiMessage` 加入对话消息列表。
3. 遍历 `toolExecutionRequests`，用 `ToolExecutor` 执行每个工具，把结果封装成 `ToolExecutionResultMessage` 加入消息列表。
4. 基于汇总后的消息列表再次发起请求，得到最终文本结果。

```java
@RequestMapping("/tool")
public String tool() {
    // 1、定义工具列表
    List<ToolSpecification> toolSpecifications =
            ToolSpecifications.toolSpecificationsFrom(TemperatureTools.class);

    // 2、构造用户提示词
    UserMessage userMessage = UserMessage.from("2025年11月11日，杭州的气温怎样？");
    List<ChatMessage> chatMessages = new ArrayList<>();
    chatMessages.add(userMessage);

    // 3、创建 ChatRequest，并指定工具列表
    ChatRequest request = ChatRequest.builder()
            .messages(userMessage)
            .toolSpecifications(toolSpecifications)
            .toolChoice(ToolChoice.AUTO)
            .build();

    // 4、调用模型
    ChatResponse response = chatModel.chat(request);
    AiMessage aiMessage = response.aiMessage();

    // 5、把模型结果添加到 chatMessages 中
    chatMessages.add(aiMessage);

    // 6、执行工具
    List<ToolExecutionRequest> toolExecutionRequests = response.aiMessage().toolExecutionRequests();
    toolExecutionRequests.forEach(toolExecutionRequest -> {
        ToolExecutor toolExecutor = new DefaultToolExecutor(new TemperatureTools(), toolExecutionRequest);
        System.out.println("execute tool " + toolExecutionRequest.name());
        String result = toolExecutor.execute(toolExecutionRequest, UUID.randomUUID().toString());
        ToolExecutionResultMessage toolExecutionResultMessages =
                ToolExecutionResultMessage.from(toolExecutionRequest, result);
        // 7、把工具执行结果添加到 chatMessages 中
        chatMessages.add(toolExecutionResultMessages);
    });

    // 8、重新构造 ChatRequest，带上完整对话和工具列表
    ChatRequest finalRequest = ChatRequest.builder()
            .messages(chatMessages)
            .toolSpecifications(toolSpecifications)
            .build();

    // 9、再次调用模型，返回最终结果
    ChatResponse finalChatResponse = chatModel.chat(finalRequest);
    return finalChatResponse.aiMessage().text();
}
```

### 一个容易踩的坑

如果第 8 步直接用 `chatModel.chat(chatMessages)` 而不带 `toolSpecifications`，工具虽然被调用了（控制台能看到 `getTemperatureByCityAndDate invoke...`），但模型返回的最终文本并不会用到工具结果：

![工具调用结果未生效](/images/posts/langchain4j-low-level-api/tool-bug-result.png)

控制台输出：

![控制台日志](/images/posts/langchain4j-low-level-api/tool-console.png)

原因在于：最后一次与模型对话时，模型没有拿到工具定义信息，所以无法理解工具执行结果的含义。修复方法是重新构造 `ChatRequest`，既带上完整对话 `chatMessages`，也带上 `toolSpecifications`（即上面代码的第 8 步）。修正后，模型才能正确结合工具返回的温度给出最终回答：

![工具调用正确结果](/images/posts/langchain4j-low-level-api/tool-success-result.png)

## 小结

这套低层次 API 的工具调用流程确实繁琐，但它能帮助开发者真正理解 LLM function call 的原理：LLM 只负责告诉调用方「该用哪个工具、传什么参数」，真正的执行需要由调用方完成；执行后若需要模型再做文本输出，还必须把工具结果连同工具定义一起重新喂给模型，让它再做一次对话。（Spring AI 封装程度高，把这些步骤都隐藏了。）

低层次 API 的价值在于提供基础能力，允许更直接地使用和深度定制。当高层次 API 无法满足需求时，可以回到低层次 API 中实现。日常开发中更推荐使用高层次 API（AI Services），它们在封装这些细节的同时保留了足够的灵活性。
