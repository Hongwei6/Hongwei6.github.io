---
title: "实战：使用 Spring AI 开发 MCP Client"
date: 2026-08-21
draft: false
description: "基于 Spring AI 实现 MCP Client，支持 Stdio、SSE、Streamable HTTP 三种传输模式，通过配置自动注入或手动构建方式接入 MCP Server"
tags: ["Spring AI", "Java", "MCP"]
categories: ["后端"]
cover: "/hero/mcp-client/mcp-client-chatclient.png"
---

基于之前对 MCP 原理的讲解，本文介绍如何使用 Spring AI 开发 MCP Client。与 VSCode Cline 类似，Spring AI 的 MCP Client 允许在项目中灵活定义客户端，实现更丰富的智能体功能。

## 依赖引入

传统 Web 项目使用 **webmvc**，响应式编程使用 **webflux**，二选一即可：

```xml
<!-- webmvc -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>
</dependency>

<!-- webflux -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client-webflux</artifactId>
</dependency>
```

## 自动注入方式

### 配置文件

配置三种 MCP Server 连接方式：

```yaml
spring:
  ai:
    mcp:
      client:
        enabled: true
        name: my-mcp-client
        version: 1.0.0
        request-timeout: 60s
        type: SYNC
        stdio:
          connections:
            weather-stdio:
              command: java
              args:
                - -jar
                - "/path/to/mcp-server-stdio.jar"
        sse:
          connections:
            weather-sse:
              url: http://127.0.0.1:8003
              sse-endpoint: /sse
        streamable-http:
          connections:
            weather-streamable:
              url: http://127.0.0.1:8004/stream/test/
              endpoint: api/mcp
```

### JSON 配置方式（仅支持 Stdio）

在 `resources` 目录下创建 `mcp-servers.json`：

```json
{
  "mcpServers": {
    "weather-stdio": {
      "command": "java",
      "args": ["-jar", "/path/to/mcp-server-stdio.jar"]
    }
  }
}
```

配置文件中指定：

```yaml
stdio:
  servers-configuration: classpath:/mcp-servers.json
```

### McpSyncClient 调用

配置会自动注入到 `List<McpSyncClient>` 中，遍历调用即可：

```java
public McpSchema.CallToolResult callTool(String type) {
    String toolName = "getWeather";
    Map<String, String> param = new HashMap<>();
    param.put("city", "北京");

    for (McpSyncClient client : mcpSyncClients) {
        McpSchema.Implementation clientInfo = client.getClientInfo();
        McpSchema.Implementation serverInfo = client.getServerInfo();
        
        if (clientInfo.title().contains(type)) {
            McpSchema.CallToolRequest request = McpSchema.CallToolRequest.builder()
                    .name(toolName)
                    .arguments(param)
                    .build();
            return client.callTool(request);
        }
    }
    return null;
}
```

![MCP Client 调用结果](/hero/mcp-client/mcp-client-call1.png)

![MCP Client 调用结果](/hero/mcp-client/mcp-client-call2.png)

![MCP Client 调用结果](/hero/mcp-client/mcp-client-call3.png)

### ChatClient 调用

通过 `SyncMcpToolCallbackProvider` 将 MCP 工具注入 ChatClient：

```java
@Autowired
private SyncMcpToolCallbackProvider toolCallbackProvider;

@Autowired
private OpenAiChatModel chatModel;

private ChatClient chatClient;

@PostConstruct
public void init() {
    ToolCallback[] toolCallbacks = toolCallbackProvider.getToolCallbacks();
    this.chatClient = ChatClient.builder(chatModel)
            .defaultToolCallbacks(toolCallbacks)
            .build();
}

public String chat(String userMessage) {
    return chatClient.prompt()
            .user(userMessage)
            .call()
            .content();
}
```

![ChatClient 调用示例](/hero/mcp-client/mcp-client-chatclient.png)

## 手动构建方式

手动构建 Stdio、SSE、Streamable 三种 MCP Client：

```java
@Service
public class ManualMcpClientService {

    @Autowired
    private OpenAiChatModel chatModel;

    private ChatClient chatClient;

    @PostConstruct
    public void init() {
        // STDIO
        ServerParameters parameters = ServerParameters.builder("java")
                .args("-jar", "/path/to/mcp-server-stdio.jar")
                .build();
        StdioClientTransport stdioTransport = new StdioClientTransport(parameters, McpJsonMapper.createDefault());

        McpSyncClient stdioClient = McpClient.sync(stdioTransport)
                .clientInfo(new McpSchema.Implementation("my-client", "1.0"))
                .requestTimeout(Duration.ofSeconds(10))
                .build();
        stdioClient.initialize();

        // SSE
        HttpClientSseClientTransport transport = HttpClientSseClientTransport.builder("http://127.0.0.1:8003")
                .sseEndpoint("/sse")
                .build();
        McpSyncClient sseClient = McpClient.sync(transport)
                .clientInfo(new McpSchema.Implementation("sse-client", "1.0"))
                .requestTimeout(Duration.ofSeconds(10))
                .build();
        sseClient.initialize();

        // STREAMABLE
        HttpClientStreamableHttpTransport streamableTransport = HttpClientStreamableHttpTransport.builder("http://127.0.0.1:8004/stream/test/")
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
                .build();

        ToolCallback[] callbacks = provider.getToolCallbacks();

        this.chatClient = ChatClient.builder(chatModel)
                .defaultToolCallbacks(callbacks)
                .build();
    }

    public String chat(String userMessage) {
        return chatClient.prompt()
                .user(userMessage)
                .call()
                .content();
    }
}
```

> **注意：** 手动配置 `HttpClientSseClientTransport` 和 `HttpClientStreamableHttpTransport` 时，**endpoint 和 baseUri 要分开写**。这与 Spring AI 的 URI 解析方式有关，合并在一起会导致解析错误（404）。

![手动构建示例](/hero/mcp-client/mcp-client-manual.png)
