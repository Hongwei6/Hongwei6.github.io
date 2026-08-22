---
title: "MCP 工具过滤"
date: 2026-08-22T09:34:00+08:00
draft: false
description: "Spring AI MCP 工具过滤机制，通过 McpToolFilter 实现精准控制工具注入，降低 Token 成本并提升模型选择准确性"
tags: ["Spring AI", "Java", "MCP"]
categories: ["后端"]
cover: "/hero/mcp-tool-filter/07-filter-result.png"
---

在企业级场景中，MCP Server 往往包含大量工具。不加区分地将所有工具交给大模型会带来性能与准确性问题。

## 为什么要做工具过滤？

### 1. 降低上下文压力与 Token 成本

每个工具的 Schema 和描述都会放入系统提示词，占用大模型上下文。如果 MCP Server 有几十个工具，而当前任务只需其中少量，其余工具描述就是无意义的上下文负担。

### 2. 提升模型工具选择准确性

工具数量过多时，模型需要在更大的候选集中判断，干扰项越多，选错工具或产生幻觉的概率越高。过滤无关工具可让模型在更干净的列表中决策。

### 3. 多智能体角色划分

在多智能体架构中，不同智能体负责不同任务。例如：
- 告警分析智能体只需要告警查询工具
- 资产分析智能体只需要资产查询工具

不过滤会导致所有智能体看到全部接口，增加复杂度，可能造成误调用甚至越权风险。

## 工具过滤原理

Spring AI 提供了 `McpToolFilter` 实现工具过滤。

![SyncMcpToolCallbackProvider](/hero/mcp-tool-filter/01-provider-class.png)

在 `getToolCallbacks` 方法中，使用 `McpToolFilter` 对工具进行过滤：

![getToolCallbacks 方法](/hero/mcp-tool-filter/02-get-tool-callbacks.png)

![filter test](/hero/mcp-tool-filter/03-filter-test.png)

`McpToolFilter` 继承于 `BiPredicate`，接收两个参数返回 boolean：

![BiPredicate](/hero/mcp-tool-filter/04-bi-predicate.png)

两个参数分别是：
- `McpConnectionInfo`：连接的 Server 信息
- `McpTool`：工具本身

![Filter 参数](/hero/mcp-tool-filter/05-filter-params.png)

**返回 true 表示保留该工具，false 表示丢弃。** Spring AI 默认行为是全放行。

![默认 Filter](/hero/mcp-tool-filter/06-default-filter.png)

## 实现方案

### MCP Server 端

定义 4 个工具，其中 3 个以 `weather` 开头，1 个不以 `weather` 开头：

```java
@Service
@Slf4j
public class WeatherService {

    @Tool(name = "weatherQueryByCity", description = "根据城市名称查询天气信息")
    public String getWeatherByCity(String city) {
        if (city == null) return "请提供城市名称";
        return switch (city) {
            case "北京" -> "北京: 晴, 25°C";
            case "上海" -> "上海: 多云, 22°C";
            case "深圳" -> "深圳: 小雨, 28°C";
            default -> city + ": 下雪, -20°C";
        };
    }

    @Tool(name = "weatherForecast", description = "查询未来天气预报")
    public String getWeatherForecast(String city) {
        if (city == null) return "请提供城市名称";
        return city + ": 明天多云，后天有小雨。";
    }

    @Tool(name = "weatherAlert", description = "获取城市天气预警信息")
    public String getWeatherAlert(String city) {
        if (city == null) return "请提供城市名称";
        return city + ": 暴雨黄色预警，注意安全。";
    }

    @Tool(name = "climateIndex", description = "查询城市气候指数")
    public String getClimateIndex(String city) {
        return city + ": 舒适度 72/100，相对湿度 65%。";
    }
}
```

### MCP Client 端

在构建 `SyncMcpToolCallbackProvider` 时传入 `toolFilter` 表达式：

```java
HttpClientStreamableHttpTransport streamableTransport = HttpClientStreamableHttpTransport
        .builder("http://127.0.0.1:8004/stream/test/")
        .endpoint("api/mcp")
        .build();

McpSyncClient streamableClient = McpClient.sync(streamableTransport)
        .clientInfo(new McpSchema.Implementation("streamable-client", "1.0"))
        .requestTimeout(Duration.ofSeconds(10))
        .build();

streamableClient.initialize();

List<McpSyncClient> clients = List.of(streamableClient);

SyncMcpToolCallbackProvider provider = SyncMcpToolCallbackProvider.builder()
        .mcpClients(clients)
        // 关键过滤方法：只有 weather 开头的工具才会被注入
        .toolFilter((conn, tool) -> tool.name().startsWith("weather"))
        .build();

ToolCallback[] callbacks = provider.getToolCallbacks();

this.chatClient = ChatClient.builder(chatModel)
        .defaultToolCallbacks(callbacks)
        .build();
```

### 效果验证

运行代码，生成的 toolcallback 只有 3 个 `weather` 开头的工具，`climateIndex` 被过滤：

![过滤结果](/hero/mcp-tool-filter/07-filter-result.png)

根据业务实际需求，设置合适的 `BiPredicate` 表达式即可实现灵活的工具过滤。
