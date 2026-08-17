---
title: "实战：使用 Spring AI 开发 MCP Server（Stdio / SSE / Streamable HTTP）"
date: 2026-08-17
draft: false
description: "使用 Spring AI 三种传输模式开发 MCP Server 的实战记录，包含 Stdio、SSE、Streamable HTTP 的配置要点、JSON-RPC 生命周期验证和 Cline 接入。"
tags: ["MCP", "Spring AI", "Java", "智能体", "Cline"]
categories: ["AI", "后端"]
cover: "/hero/mcp-spring-ai-sse-cline.png"
---

Spring AI 提供了开箱即用的 MCP Server Starter，支持 Stdio、SSE、Streamable HTTP 三种传输模式。本文以天气查询工具为例，记录三种模式的配置要点和接入验证过程。

工具代码三种模式通用：

```java
@Service
public class WeatherService {
    @Tool(description = "根据城市名称查询天气信息")
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
}
```

注册工具到 `ToolCallbackProvider`：

```java
@Bean
public ToolCallbackProvider weatherTools(WeatherService weatherService) {
    return MethodToolCallbackProvider.builder().toolObjects(weatherService).build();
}
```

## Stdio 模式

本地客户端（Cursor、Cline 等）的首选方式，通过 stdin/stdout 通信，无网络开销。

**Maven 依赖：**

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
</dependency>
```

响应式项目用 `spring-ai-starter-mcp-server-webflux`，但不要与 webmvc 混用，IO 密集场景会阻塞。

**关键配置：** Stdio 模式要求 stdout 必须是纯 JSON-RPC 数据流，任何多余输出（日志、banner）都会导致客户端解析失败。

```yaml
spring:
  main:
    web-application-type: none
    banner-mode: off
  ai:
    mcp:
      server:
        name: mcp-server
        version: 1.0.0
        stdio: true
        enabled: true
        type: SYNC
logging:
  level:
    root: OFF
```

**Cline 接入配置：**

```json
{
  "mcpServers": {
    "weather-stdio": {
      "disabled": false,
      "timeout": 60,
      "type": "stdio",
      "command": "java",
      "args": ["-jar", "/path/to/mcp-server-stdio.jar"]
    }
  }
}
```

无需手动启动 Spring Boot，Cline 直接通过 `java -jar` 启动进程。

![Cline Stdio 接入成功](/hero/mcp-spring-ai-stdio-cline.png)

## SSE 模式

基于 HTTP 的远程传输方案，采用双端点架构：`sse-endpoint` 用于客户端订阅推送（类似收音机），`sse-message-endpoint` 用于客户端发送请求（类似寄信）。

**依赖与 Stdio 相同。** 配置改为：

```yaml
server:
  port: 8003
  servlet:
    encoding:
      charset: UTF-8
      force: true
      enabled: true

spring:
  application:
    name: mcp-weather-sse
  ai:
    mcp:
      server:
        enabled: true
        name: weather-sse-server
        version: 1.0.0
        type: SYNC
        capabilities:
          tool: true
          resource: false
          prompt: false
          completion: false
        sse-message-endpoint: /mcp/messages
        sse-endpoint: /sse
```

启动后访问 `/sse` 端点即建立长连接，返回的 `sessionId` 用于后续消息发送：

![SSE 长连接建立](/hero/mcp-spring-ai-sse-endpoint.png)

### JSON-RPC 生命周期验证

通过 Postman 可以手动验证完整的 JSON-RPC 交互流程。

**初始化** — 发送 `initialize` 请求，服务端通过 SSE 长连接返回初始化结果：

![初始化请求](/hero/mcp-spring-ai-sse-init.png)

**客户端就绪** — 发送 `notifications/initialized` 通知，此后服务端才允许后续操作。

**工具发现** — 调用 `tools/list` 获取工具列表，可以看到完整的工具定义和参数说明：

![工具列表响应](/hero/mcp-spring-ai-sse-tools.png)

**工具调用** — 发送 `tools/call` 请求，获取标准 JSON-RPC 格式结果：

![工具调用结果](/hero/mcp-spring-ai-sse-call.png)

> 浏览器查看 SSE 响应可能出现中文乱码，因为默认使用 ISO-8859-1 编码。在 `server.servlet.encoding` 中强制 UTF-8 即可解决。

**Cline 接入配置：**

```json
{
  "mcpServers": {
    "weather-sse": {
      "type": "sse",
      "url": "http://127.0.0.1:8003/sse",
      "timeout": 60,
      "disabled": false
    }
  }
}
```

![Cline SSE 接入成功](/hero/mcp-spring-ai-sse-cline.png)

## Streamable HTTP

2025 年 3 月 MCP 正式提出的新传输标准，替代 SSE。通过单一 HTTP 端点实现双向通信，支持断点重连和会话恢复。

```yaml
server:
  port: 8004
  servlet:
    encoding:
      charset: UTF-8
      force: true
      enabled: true

spring:
  application:
    name: mcp-weather-streamable
  ai:
    mcp:
      server:
        protocol: STREAMABLE
        name: streamable-mcp-server
        version: 1.0.0
        type: SYNC
        instructions: "这个服务是用来查询城市天气的。"
        resource-change-notification: true
        tool-change-notification: true
        prompt-change-notification: true
        streamable-http:
          mcp-endpoint: /api/mcp
          keep-alive-interval: 30s
```

关键配置说明：
- `protocol: STREAMABLE` — 开启 Streamable HTTP 模式
- `instructions` — 定义 Server 的提示词，指导模型行为
- `streamable-http.mcp-endpoint` — 单一端点路径（SSE 需要两个端点）
- `keep-alive-interval` — HTTP 连接心跳间隔

### 请求头要求

Streamable HTTP 协议规定，服务端可能返回 SSE 流或普通 JSON 响应。客户端必须在 Header 中声明两种格式都能处理：

![请求头设置](/hero/mcp-spring-ai-streamable-headers.png)

初始化成功后，响应头中会返回 `Mcp-Session-Id`，后续请求必须携带此 Header：

![Session ID](/hero/mcp-spring-ai-streamable-session.png)

### 有状态 vs 无状态

`protocol` 可切换为 `STATELESS`，无状态模式下 Server 不保存会话，每个请求独立处理，适合单次调用场景（如查询、计算），不依赖上下文。优势是简单、易扩展、适合 serverless 架构。

有状态模式下，Server 通过 `Mcp-Session-Id` 维护会话，支持断点重连和多轮交互。

**Cline 接入配置：**

```json
{
  "mcpServers": {
    "weather-streamable": {
      "url": "http://127.0.0.1:8004/api/mcp",
      "type": "streamableHttp",
      "timeout": 60,
      "disabled": false
    }
  }
}
```

![Cline Streamable HTTP 接入成功](/hero/mcp-spring-ai-streamable-cline.png)

## POJO 参数与返回值

Spring AI 的 MCP Server 支持 POJO 作为工具的入参和出参。用 `@ToolParam` 注解描述字段含义，大模型能准确理解业务字段：

```java
@Tool(name = "query_weather_by_city&date", description = "根据城市和日期获取天气信息")
public WeatherResponse queryWeather(WeatherRequest request) {
    double temp = Math.random() * 15 + 10;
    return new WeatherResponse(
        request.getCity(), request.getDate(),
        request.getI(), request.getS(),
        "晴朗，有微风", temp
    );
}

@Data
public class WeatherRequest {
    @ToolParam(description = "城市")
    private String city;
    @ToolParam(description = "日期")
    private String date;
    @ToolParam(description = "区县")
    private String i;
    @ToolParam(description = "街道")
    private String s;
}
```

字段名含义不明确时（如 `i`、`s`），`@ToolParam` 的描述尤为重要，否则大模型无法推断字段意图。

![POJO 参数识别](/hero/mcp-spring-ai-pojo-param.png)

## 三种模式对比

| 模式 | 依赖 | 通信方式 | 适用场景 |
|------|------|---------|---------|
| **Stdio** | webmvc / webflux | stdin/stdout | 本地客户端（Cline、Cursor） |
| **SSE** | webmvc / webflux | 双端点 HTTP | 远程调用，HTTP/1.1 环境 |
| **Streamable HTTP** | webmvc / webflux | 单端点 HTTP | 远程调用，替代 SSE，官方推荐 |
