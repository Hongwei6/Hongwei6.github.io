---
title: "Spring AI MCP 调用实现原理"
date: 2026-08-21T13:30:00+08:00
draft: false
description: "深入分析 Spring AI 中 MCP Client 的注入机制与 ChatClient 调用 MCP Server 的完整流程"
tags: ["Spring AI", "Java", "MCP", "源码分析"]
categories: ["后端"]
cover: "/hero/spring-ai-mcp-source/01-tool-callback.png"
---

本文分析 Spring AI 中 MCP Client 的注入机制以及 ChatClient 调用 MCP Server 的完整流程。

## MCP 注入机制

### ToolCallback 转换

从 `getToolCallbacks` 方法入手，Spring AI 将 `McpSyncClient` 转换为 `SyncMcpToolCallback`。这个转换将工具定义、元数据、执行逻辑封装在一起。

![ToolCallback 转换](/hero/spring-ai-mcp-source/01-tool-callback.png)

### SyncMcpToolCallback 执行逻辑

核心 `call` 方法接收大模型传来的工具参数，最终调用 `mcpClient.callTool`：

```java
@Override
public String call(String toolCallInput, @Nullable ToolContext toolContext) {
    if (!StringUtils.hasText(toolCallInput)) {
        toolCallInput = "{}";
    }
    
    Map<String, Object> arguments = ModelOptionsUtils.jsonToMap(toolCallInput);
    CallToolResult response;
    
    try {
        var mcpMeta = toolContext != null ? this.toolContextToMcpMetaConverter.convert(toolContext) : null;
        var request = CallToolRequest.builder()
                .name(this.tool.name())
                .arguments(arguments)
                .meta(mcpMeta)
                .build();
        response = this.mcpClient.callTool(request);
    } catch (Exception ex) {
        throw new ToolExecutionException(this.getToolDefinition(), ex);
    }
    
    if (response.isError() != null && response.isError()) {
        throw new ToolExecutionException(this.getToolDefinition(),
                new IllegalStateException("Error calling tool: " + response.content()));
    }
    
    return ModelOptionsUtils.toJsonString(response.content());
}
```

### 自动配置（Spring AI 1.1.0-M4+）

`SyncMcpToolCallbackProvider` 的初始化在 `McpToolCallbackAutoConfiguration` 中完成，当 `spring.ai.mcp.client.type=SYNC` 时触发。

![自动配置入口](/hero/spring-ai-mcp-source/02-builder-caller.png)

![AutoConfiguration](/hero/spring-ai-mcp-source/03-auto-config.png)

### 传输层构建

核心在于 `NamedClientMcpTransport`，包含 Stdio、SSE、Streamable 三种实现：

![NamedClientMcpTransport](/hero/spring-ai-mcp-source/04-named-transport.png)

![传输实现类](/hero/spring-ai-mcp-source/05-transport-implementations.png)

以 Stdio 为例，通过读取配置文件中的 properties 来构建 transport 通道：

![Stdio 配置](/hero/spring-ai-mcp-source/06-stdio-config.png)

### Client 初始化

在 `McpClientAutoConfiguration` 中，完成 transport 构建后，对 McpClient 进行初始化并加入 `List<McpSyncClient>`，供 `SyncMcpToolCallbackProvider` 使用。

![Client 初始化](/hero/spring-ai-mcp-source/07-client-initialize.png)

### 旧版本事件机制（Spring AI 1.1.0 之前）

早期版本通过事件机制触发初始化，监听 `McpToolsChangedEvent`：

![事件监听器](/hero/spring-ai-mcp-source/08-event-listener.png)

事件在 `McpSyncToolsChangeEventEmmiter` 中发布：

![事件发射器](/hero/spring-ai-mcp-source/09-event-emitter.png)

![事件触发](/hero/spring-ai-mcp-source/10-event-trigger.png)

同样在 `McpClientAutoConfiguration` 中创建：

![事件源码](/hero/spring-ai-mcp-source/11-event-source.png)

## ChatClient 调用 MCP Server 流程

### 调用入口

从 ChatClient 的 `call` 方法进入：

![ChatClient 调用](/hero/spring-ai-mcp-source/12-chatclient-call.png)

### DefaultChatClient 实现

`call` 方法中的 `buildAdvisorChain` 将 advisor 构建成链，`ChatModelCallAdvisor` 和 `ChatModelStreamAdvisor` 位于链末端，负责最终调用大模型。

![DefaultChatClient](/hero/spring-ai-mcp-source/13-default-chatclient.png)

![Advisor 链](/hero/spring-ai-mcp-source/14-advisor-chain.png)

### ChatModelCallAdvisor

非流式场景下进入 `ChatModelCallAdvisor`：

![ChatModelCallAdvisor](/hero/spring-ai-mcp-source/15-chatmodel-advisor.png)

### OpenAiChatModel 内部调用

进入 `internalCall` 方法，核心流程带注释如下：

```java
public ChatResponse internalCall(Prompt prompt, ChatResponse previousChatResponse) {
    // 1. 构造请求
    ChatCompletionRequest request = createRequest(prompt, false);
    
    // 2. 设置监控上下文
    ChatModelObservationContext observationContext = ChatModelObservationContext.builder()
        .prompt(prompt)
        .provider(OpenAiApiConstants.PROVIDER_NAME)
        .build();
    
    // 3. 调用大模型
    ChatResponse response = ChatModelObservationDocumentation.CHAT_MODEL_OPERATION
        .observation(this.observationConvention, DEFAULT_OBSERVATION_CONVENTION, 
                () -> observationContext, this.observationRegistry)
        .observe(() -> {
            ResponseEntity<ChatCompletion> completionEntity = this.retryTemplate
                .execute(ctx -> this.openAiApi.chatCompletionEntity(request, 
                    getAdditionalHttpHeaders(prompt)));
            
            var chatCompletion = completionEntity.getBody();
            
            // 4. 解析模型返回
            List<Choice> choices = chatCompletion.choices();
            List<Generation> generations = choices.stream()
                .map(choice -> buildGeneration(choice, Map.of(), request))
                .toList();
            
            // 5. 汇总 token 使用量
            OpenAiApi.Usage usage = chatCompletion.usage();
            Usage accumulatedUsage = UsageCalculator.getCumulativeUsage(
                    usage != null ? getDefaultUsage(usage) : new EmptyUsage(),
                    previousChatResponse
            );
            
            // 6. 包装成 ChatResponse
            ChatResponse chatResponse = new ChatResponse(
                    generations,
                    from(chatCompletion, null, accumulatedUsage)
            );
            
            observationContext.setResponse(chatResponse);
            return chatResponse;
        });
    
    // 7. 判断是否需要调用工具
    if (this.toolExecutionEligibilityPredicate.isToolExecutionRequired(prompt.getOptions(), response)) {
        // 8. 执行工具调用
        var toolExecutionResult = this.toolCallingManager.executeToolCalls(prompt, response);
        
        // 9. 判断是否直接返回
        if (toolExecutionResult.returnDirect()) {
            return ChatResponse.builder()
                .from(response)
                .generations(ToolExecutionResult.buildGenerations(toolExecutionResult))
                .build();
        }
        
        // 10. 递归调用，将工具结果加入对话历史
        return this.internalCall(
            new Prompt(toolExecutionResult.conversationHistory(), prompt.getOptions()),
            response
        );
    }
    
    return response;
}
```

![OpenAiChatModel](/hero/spring-ai-mcp-source/16-openai-model.png)

### 核心流程

1. 先调用一次大模型
2. 检查返回结果是否包含 `tool_calls` 字段
3. 如果有，调用注入的工具执行
4. 根据 `returnDirect` 参数决定直接返回还是再次调用模型总结
5. 递归处理，直到模型不再输出 `tool_calls`

![对话流程](/hero/spring-ai-mcp-source/17-conversation-flow.png)

### tool_calls 判断逻辑

![tool_calls 判断](/hero/spring-ai-mcp-source/18-tool-calls-check.png)

![判断逻辑详情](/hero/spring-ai-mcp-source/19-tool-calls-logic.png)

### 工具执行

在 `DefaultToolCallingManager.executeToolCalls` 中，通过 `toolCallback.call` 方法执行具体工具：

![工具执行入口](/hero/spring-ai-mcp-source/20-execute-tool.png)

![toolCallback 调用](/hero/spring-ai-mcp-source/21-toolcallback-call.png)

![工具注入回顾](/hero/spring-ai-mcp-source/22-tool-inject.png)

![回调返回](/hero/spring-ai-mcp-source/23-callback-return.png)

## 总结

整个流程串联起来：

1. **注入阶段**：通过自动配置将 `McpSyncClient` 转换为 `SyncMcpToolCallback`，注入到 ChatClient
2. **调用阶段**：ChatClient 通过 Advisor 链调用大模型，根据返回的 `tool_calls` 判断是否需要调用工具
3. **执行阶段**：调用 `SyncMcpToolCallback.call` → `mcpClient.callTool` 执行具体工具
4. **递归处理**：将工具结果加入对话历史，再次调用模型，直到不再需要工具调用
