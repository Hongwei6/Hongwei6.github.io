---
title: "跳过 MCP 的模型总结"
date: 2026-08-22T09:28:00+08:00
draft: false
description: "分析 returnDirect 参数在 MCP 中的局限性，以及通过继承 SyncMcpToolCallback 实现跳过模型迭代总结的方案"
tags: ["Spring AI", "Java", "MCP", "源码分析"]
categories: ["后端"]
cover: "/hero/mcp-return-direct/12-mcp-success.png"
---

Spring AI 中 MCP 调用工具后，会将结果再次丢给大模型做迭代总结。但在某些场景下，这个机制会成为性能瓶颈。

## 为什么要跳过模型总结？

### 多智能体协作（Multi-Agent）

![多智能体协作](/hero/mcp-return-direct/01-multi-agent.png)

多个 Agent 持续调用工具、互相传递结果，每一步都让模型总结会带来：
- 延迟倍增
- token 成本增加
- 智能体逻辑链路变长
- 用户等待时间过长

某些 Agent 只需执行中间任务（计算、数据清洗），无需总结，只需获取结果。

### 工具输出就是最终答案

很多 MCP 工具返回的是结构化、可直接使用的业务数据（JSON、对象、列表）。让模型再总结会：
- 浪费时间和算力
- 增加 token 成本
- 延长响应延迟
- 模型可能破坏结构化数据（字段名被改写、格式被重组）

特别是像 **LightRAG** 这类工具，内部已经走完「检索 → 推理 → 生成」全流程，再次总结毫无价值。

## returnDirect 参数

Spring AI 提供了 `returnDirect` 参数用于跳过模型迭代总结：

```java
@Tool(description = "根据城市名称查询天气信息", returnDirect = true)
public String getWeather(String city) {
    if (city == null) {
        return "请提供城市名称";
    }
    return switch (city) {
        case "北京" -> "北京: 晴, 25°C";
        case "上海" -> "上海: 多云, 22°C";
        case "深圳" -> "深圳: 小雨, 28°C";
        default -> city + ": 下雪, -20°C";
    };
}
```

但实际测试发现，MCP 模式下设置 `returnDirect = true` 并未生效：

![MCP 下未生效](/hero/mcp-return-direct/02-return-direct-fail.png)

## 源码分析

定位到 `DefaultToolCallingManager` 的 `executeToolCall` 方法：

![DefaultToolCallingManager](/hero/mcp-return-direct/03-default-tool-manager.png)

它从 `toolCallback` 的 `toolMetadata` 中获取 `returnDirect`，但默认返回 false：

![ToolMetadata 获取](/hero/mcp-return-direct/04-tool-metadata.png)

![默认返回 false](/hero/mcp-return-direct/05-default-return.png)

查看 `SyncMcpToolCallback` 实现类，发现完全没有对元数据进行实现：

![SyncMcpToolCallback](/hero/mcp-return-direct/06-mcp-callback.png)

### FunctionCall 能否生效？

查看 `FunctionToolCallback`（传统 Function-Call），发现有元数据相关操作：

![FunctionToolCallback](/hero/mcp-return-direct/07-function-callback.png)

使用 FunctionCall 方式测试，`returnDirect` 成功读取为 true：

![returnDirect 为 true](/hero/mcp-return-direct/08-return-direct-true.png)

![FunctionCall 成功](/hero/mcp-return-direct/09-function-success.png)

**结论：** `returnDirect` 参数在 MCP 模式下 Spring AI 未实现。

## MCP 跳过总结方案

核心思路：通过继承接管 `SyncMcpToolCallback` 和 `SyncMcpToolCallbackProvider`，重写元数据获取方式。

### 自定义 ToolCallback

```java
public class ReturnDirectSyncMcpToolCallback extends SyncMcpToolCallback {

    private final boolean returnDirect;

    public ReturnDirectSyncMcpToolCallback(McpSyncClient client, McpSchema.Tool tool, boolean returnDirect) {
        super(client, tool);
        this.returnDirect = returnDirect;
    }

    @Override
    public ToolMetadata getToolMetadata() {
        return ToolMetadata.builder()
                .returnDirect(returnDirect)
                .build();
    }
}
```

### 自定义 Provider

```java
@Slf4j
public class DirectReturnMcpToolCallbackProvider extends SyncMcpToolCallbackProvider {

    private final List<McpSyncClient> mcpClients;
    private boolean returnDirect;

    public DirectReturnMcpToolCallbackProvider(List<McpSyncClient> mcpClients, boolean returnDirect) {
        super(mcpClients);
        this.mcpClients = mcpClients;
        this.returnDirect = returnDirect;
    }

    @Override
    public ToolCallback[] getToolCallbacks() {
        var toolCallbacks = new ArrayList<>();

        for (McpSyncClient mcpClient : mcpClients) {
            List<McpSchema.Tool> toolList = Collections.emptyList();
            try {
                toolList = mcpClient.listTools().tools();
            } catch (Exception e) {
                continue;
            }

            for (var tool : toolList) {
                toolCallbacks.add(new ReturnDirectSyncMcpToolCallback(mcpClient, tool, returnDirect));
            }
        }

        var array = toolCallbacks.toArray(new ToolCallback[0]);
        validateToolCallbacks(array);
        return array;
    }

    private void validateToolCallbacks(ToolCallback[] toolCallbacks) {
        List<String> duplicateToolNames = ToolUtils.getDuplicateToolNames(toolCallbacks);
        if (!duplicateToolNames.isEmpty()) {
            throw new IllegalStateException(
                    "Multiple tools with the same name (%s)".formatted(String.join(", ", duplicateToolNames)));
        }
    }
}
```

### 注入 ChatClient

```java
DirectReturnMcpToolCallbackProvider callbackProvider = 
    new DirectReturnMcpToolCallbackProvider(clients, true);

this.chatClient = ChatClient.builder(chatModel)
        .defaultToolCallbacks(callbackProvider)
        .build();
```

### 效果验证

![MCP 跳过总结成功](/hero/mcp-return-direct/12-mcp-success.png)

成功跳过模型迭代总结。返回结果中的 `text` 来自 `SyncMcpToolCallback` 的 `call` 方法返回值：

![call 方法返回](/hero/mcp-return-direct/13-call-return.png)

![call 方法源码](/hero/mcp-return-direct/14-call-method.png)

![结果格式](/hero/mcp-return-direct/15-result-format.png)

通过这种方式，不仅可以实现跳过模型总结，还可以重写 `call` 方法在调用 MCP 工具时做定制化处理。
