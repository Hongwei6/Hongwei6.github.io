---
title: "SSE MCP Server 如何实现重连"
date: 2026-08-21T13:33:00+08:00
draft: false
description: "SSE 模式下 MCP Client 断线重连机制的实现方案，通过心跳检测和自动重试保障智能体系统的可靠性"
tags: ["Spring AI", "Java", "MCP", "SSE"]
categories: ["后端"]
cover: "/hero/sse-reconnect/08-reconnect-success.png"
---

在 MCP 早期版本中，主要依赖 Stdio（本地）和 SSE（远程）两种方式。SSE 是典型的单向流式机制，通过一个端点持续监听服务器指令，再通过另一个端点发送消息。

## 问题背景

SSE 模式**对长连接依赖极强**，一旦网络出现抖动、服务端重启、代理回收空闲连接，SSE 管道就会立即断开。

Spring AI MCP Client **没有提供自动重连能力**，SSE 中断后客户端会失去与 MCP Server 的指令通道，工具虽然注册着但不再响应调用，整个智能体陷入无工具状态。

对于需要长时间运行的企业级智能体系统，必须实现可靠的重连能力。

## 问题复现

### 正常连接

启动 MCP Server SSE 项目（端口 8003）和 MCP Client（端口 8001）：

![MCP Server 启动](/hero/sse-reconnect/01-server-start.png)

![MCP Client 启动](/hero/sse-reconnect/02-client-start.png)

手动构建 SSE Client 并注入 ChatClient：

```java
HttpClientSseClientTransport transport = HttpClientSseClientTransport
        .builder("http://127.0.0.1:8003")
        .sseEndpoint("/sse")
        .build();

McpSyncClient sseClient = McpClient.sync(transport)
        .clientInfo(new McpSchema.Implementation("sse-client", "1.0"))
        .requestTimeout(Duration.ofSeconds(10))
        .build();

sseClient.initialize();

SyncMcpToolCallbackProvider provider = SyncMcpToolCallbackProvider.builder()
        .mcpClients(List.of(sseClient))
        .build();

ToolCallback[] callbacks = provider.getToolCallbacks();

this.chatClient = ChatClient.builder(chatModel)
        .defaultToolCallbacks(callbacks)
        .defaultTools()
        .build();
```

连接成功后调用工具正常：

![工具调用成功](/hero/sse-reconnect/03-tool-call-success.png)

### 断线场景

MCP Server 挂掉后，Client 立即报错：

![Server 断开](/hero/sse-reconnect/04-server-disconnect.png)

再次调用接口，工具已无法使用：

![工具调用失败](/hero/sse-reconnect/05-tool-call-fail.png)

### 恢复失败

重新启动 MCP Server 后，Client 依然无法感知重连，工具调用继续失败：

![Server 重启](/hero/sse-reconnect/06-server-restart.png)

![仍然失败](/hero/sse-reconnect/07-still-fail.png)

## 重连机制实现

核心思路：为 SSE 模式增加弹性连接管理机制，自动检测 SSE 中断并主动重新建立连接、重新初始化会话与工具注册。

### 关键方法

- **`McpSyncClient.ping()`**：作为心跳检测手段
- **`@Scheduled`**：定时任务执行心跳检测
- **`AtomicBoolean`**：保证重试线程唯一性

### 实现方案

```java
@Service
@Slf4j
public class RetrySSEMcpServer {

    @Autowired
    private OpenAiChatModel chatModel;

    private ChatClient chatClient;
    private McpSyncClient sseClient;
    
    private final AtomicBoolean retrying = new AtomicBoolean(false);
    private final ExecutorService retryExecutor = Executors.newSingleThreadExecutor();

    @PostConstruct
    public void init() {
        log.info("Initializing SSE MCP Client...");
        this.sseClient = buildClient();
        
        try {
            this.sseClient.initialize();
            log.info("SSE MCP client initialized.");
        } catch (Exception e) {
            log.error("Initial SSE initialize failed, will rely on retry thread.", e);
            startRetryInitialize();
        }

        SyncMcpToolCallbackProvider provider = SyncMcpToolCallbackProvider.builder()
                .mcpClients(List.of(this.sseClient))
                .build();
        ToolCallback[] callbacks = provider.getToolCallbacks();

        this.chatClient = ChatClient.builder(chatModel)
                .defaultToolCallbacks(callbacks)
                .defaultTools()
                .build();
    }

    private McpSyncClient buildClient() {
        HttpClientSseClientTransport transport = HttpClientSseClientTransport
                .builder("http://127.0.0.1:8003")
                .sseEndpoint("/sse")
                .build();
        return McpClient.sync(transport)
                .clientInfo(new McpSchema.Implementation("sse-client", "1.0"))
                .requestTimeout(Duration.ofSeconds(10))
                .build();
    }

    /**
     * 定时任务：每 5 秒 ping 一次 SSE
     * ping 不通则触发 initialize 重试线程
     */
    @Scheduled(fixedDelay = 5000)
    public void pingSse() {
        log.info("SSE MCP ping...");
        if (sseClient == null) {
            log.warn("SSE client not initialized yet.");
            startRetryInitialize();
            return;
        }

        try {
            sseClient.ping();
            log.debug("SSE MCP ping OK.");
        } catch (Exception e) {
            log.error("SSE MCP ping failed: {}", e.getMessage());
            startRetryInitialize();
        }
    }

    /**
     * 启动 initialize 重试线程
     */
    private void startRetryInitialize() {
        if (!retrying.compareAndSet(false, true)) {
            return;
        }

        retryExecutor.submit(() -> {
            log.warn("Start retrying SSE MCP initialize...");
            while (true) {
                try {
                    this.sseClient = buildClient();
                    this.sseClient.initialize();
                    log.info("SSE MCP re-initialized successfully.");

                    SyncMcpToolCallbackProvider provider = SyncMcpToolCallbackProvider.builder()
                            .mcpClients(List.of(this.sseClient))
                            .build();
                    ToolCallback[] callbacks = provider.getToolCallbacks();

                    this.chatClient = ChatClient.builder(chatModel)
                            .defaultToolCallbacks(callbacks)
                            .defaultTools()
                            .build();

                    retrying.set(false);
                    return;
                } catch (Exception e) {
                    log.warn("Retry initialize failed, will retry in 10s. Reason: {}", e.getMessage());
                }

                try {
                    Thread.sleep(10000);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        });
    }

    public String chat(String userMessage) {
        return chatClient.prompt()
                .user(userMessage)
                .call()
                .content();
    }
}
```

### 注意事项

除了 `McpSyncClient` 需要重新初始化，**ChatClient 也需要重建**。因为 ChatClient 内部的 ToolCallback 在初始化时注入，绑定的是旧的 McpSyncClient。即使重新初始化了 sseClient，ChatClient 没有同步更新仍会调用旧客户端。

## 效果验证

启动 Client 和 Server，然后断开 Server，Client 报错：

![重连成功日志](/hero/sse-reconnect/08-reconnect-success.png)

![重试报错](/hero/sse-reconnect/09-retry-error.png)

重新启动 Server 后，自动重连成功：

![重试成功](/hero/sse-reconnect/10-retry-success.png)

再次调用 ChatClient 正常工作：

![对话成功](/hero/sse-reconnect/11-chat-success.png)
