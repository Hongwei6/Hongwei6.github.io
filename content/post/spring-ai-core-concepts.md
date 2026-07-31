---
title: "Spring AI 核心概念：Model、ChatClient 与 Prompt"
date: 2026-07-29
draft: false
description: "梳理 Spring AI 的核心抽象——ChatModel、ChatClient、Prompt、ChatResponse，以及 Advisor 机制，理解框架如何统一对接不同大模型。"
tags: ["Spring AI", "Java", "AI", "大模型"]
categories: ["后端"]
cover: "/hero/tt1.png"
---

Spring AI 的核心目标是为 Java 开发者提供一套统一的大模型对接方案。框架围绕几个关键抽象构建：**Model** 负责与底层模型通信，**ChatClient** 提供更高层的流式 API，**Prompt** 和 **ChatResponse** 分别封装输入输出，**Advisor** 则以拦截器的形式扩展请求处理链路。

## Model 体系

Spring AI 按功能将模型分为 Chat Model、Embedding Model、Image Model、Audio Model 等类型。其中 Chat Model 是对话场景的核心。

### ChatModel 接口

`ChatModel` 定义了与对话模型交互的统一方式，抽象了 OpenAI、Anthropic、Azure OpenAI、百炼、Ollama 等不同厂商的实现差异。无论底层用的是哪个模型，调用方式完全一致。

```java
public interface ChatModel extends Model<Prompt, ChatResponse>, StreamingChatModel {

    default String call(String message) {
        Prompt prompt = new Prompt(new UserMessage(message));
        Generation generation = call(prompt).getResult();
        return (generation != null) ? generation.getOutput().getText() : "";
    }

    @Override
    ChatResponse call(Prompt prompt);

    default Flux<ChatResponse> stream(Prompt prompt) {
        throw new UnsupportedOperationException("streaming is not supported");
    }
}
```

接口继承了 `Model` 和 `StreamingChatModel`：

- `Model` 定义了 `call(Prompt)` 方法，以同步方式调用大模型
- `StreamingChatModel` 定义了 `stream(Prompt)` 方法，返回 `Flux<ChatResponse>`，用于流式输出

### Prompt

`Prompt` 是模型的输入，包含两部分：

```java
public class Prompt implements ModelRequest<List<Message>> {
    // 对话历史 + 当前用户输入
    private final List<Message> messages;
    // 调用 Chat Model 时的额外参数
    @Nullable
    private ChatOptions chatOptions;
}
```

**Message** 有多种实现，对应不同的对话角色：

| 实现类 | 角色 | 用途 |
|--------|------|------|
| `SystemMessage` | SYSTEM | 系统设定，控制模型行为 |
| `UserMessage` | USER | 用户输入 |
| `AssistantMessage` | ASSISTANT | 模型回复 |
| `ToolResponseMessage` | TOOL | 工具返回结果 |

**ChatOptions** 是可选参数，用于指定模型调用时的额外配置：模型名称、温度（temperature）、最大 token 数、Top-k/Top-p 采样策略等。通过 `DashScopeChatOptions.builder()` 可以快速创建：

```java
DashScopeChatOptions.builder()
    .withModel("qwen-plus")
    .temperature(0.7)
    .build()
```

### ChatResponse

`ChatResponse` 是模型的输出。日常使用中最常用的方法：

```java
resp.getResult().getOutput().getText()
```

### 实际使用

以 Spring AI Alibaba 的 `DashScopeChatModel` 为例，框架在自动配置中完成了 Bean 注册，注入即可使用：

```java
@RestController
@RequestMapping("/model")
public class ChatModelController {

    @Autowired
    private DashScopeChatModel dashScopeChatModel;

    @RequestMapping("/call/string")
    public String callString(String message) {
        return dashScopeChatModel.call(message);
    }

    @RequestMapping("/stream/string")
    public Flux<String> callStreamString(String message, HttpServletResponse response) {
        response.setContentType("text/event-stream");
        response.setCharacterEncoding("UTF-8");
        return dashScopeChatModel.stream(message);
    }
}
```

流式接口需要在响应头中设置 `Content-Type: text/event-stream` 和字符编码 `UTF-8`，否则中文会乱码。

## ChatClient

`ChatClient` 是基于 `ChatModel` 封装的更高层 API，提供更简洁的调用方式和更丰富的功能。

基础能力：

- 定制和组装 Prompt
- 格式化解析模型输出（Structured Output）
- 调整 ChatOptions

高级能力：

- 聊天记忆（Chat Memory）
- 工具/函数调用（Function Calling / Tools）
- RAG（检索增强生成）

### 基本用法

```java
@RestController
@RequestMapping("/client")
public class ChatClientController implements InitializingBean {

    @Autowired
    private ChatModel dashScopeChatModel;

    private ChatClient chatClient;

    @GetMapping("/simpleCall")
    public String simpleCall(String message) {
        return chatClient.prompt(message).call().content();
    }

    @GetMapping("/stream")
    public Flux<String> stream(String message) {
        return chatClient.prompt(message).stream().content();
    }

    @Override
    public void afterPropertiesSet() {
        chatClient = ChatClient.builder(dashScopeChatModel)
                .defaultAdvisors(new SimpleLoggerAdvisor())
                .defaultSystem("请用英文回答问题")
                .defaultOptions(
                    DashScopeChatOptions.builder()
                        .temperature(0.7)
                        .build()
                )
                .build();
    }
}
```

### default 配置

`ChatClient` 初始化时可以通过 `default*` 方法预设配置：`defaultSystem`、`defaultUser`、`defaultOptions`、`defaultAdvisors` 等。这些配置在后续调用中自动生效，无需重复指定。

调用时如果重新指定了同一配置（如 `.system("用韩语回答")`），会覆盖对应的 default 值。但有一个例外：如果在 `Prompt` 中通过 `SystemMessage` 设置系统提示词，效果是**追加**而非覆盖。

### Tools

Tools 是大模型能力的关键扩展点——没有工具的大模型只是对话机器人，接入工具后才能成为真正的智能助手。通过 `ChatClient` 的 `.defaultTools()` 或 `.tools()` 方法注册可用工具，模型会在需要时自动调用。具体用法在 Tool Call 篇章展开。

### Advisors

Advisors 是一组拦截器，类似于 Spring AOP 的概念，用于在模型调用前后对 Prompt 或 Response 进行拦截、修改、增强或记录。

```java
public interface CallAroundAdvisor extends AroundAdvisor {
    AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain);
}
```

以 `SimpleLoggerAdvisor` 为例，它在 `aroundCall` 中实现了类似 AOP 的前后拦截：调用前记录请求日志，调用后记录响应日志。

Advisors 是 Spring AI 扩展机制的核心，RAG、聊天记忆等功能都依赖它实现。框架内置了多种 Advisor 实现，也支持自定义扩展。

## 小结

| 概念 | 职责 |
|------|------|
| `ChatModel` | 底层模型抽象，统一同步/流式调用 |
| `ChatClient` | 高层门面 API，支持 default 配置、Tools、Advisors |
| `Prompt` | 模型输入，包含 Message 列表和 ChatOptions |
| `ChatResponse` | 模型输出，通过 `getResult().getOutput().getText()` 获取文本 |
| `Advisor` | 拦截器机制，扩展请求/响应处理链路 |
