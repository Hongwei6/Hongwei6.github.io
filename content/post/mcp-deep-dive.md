---
title: "深入理解 MCP 技术原理：协议设计、数据层与传输层"
date: 2026-08-17
draft: false
description: "从原理层面拆解 MCP 协议——为什么必须是协议而非框架，JSON-RPC 数据层如何实现工具标准化，Stdio/SSE/Streamable HTTP 三种传输机制的设计差异。"
tags: ["MCP", "AI", "智能体", "JSON-RPC", "协议"]
categories: ["AI"]
cover: "/hero/mcp-deep-intro-banner.png"
---

![MCP 技术原理](/hero/mcp-deep-intro-banner.png)

前文介绍了 MCP 的基本概念和工作流程，本文从原理层面拆解 MCP 的设计：为什么必须是协议而非框架、如何通过 Client-Server 结构将工具从智能体中解耦、数据层和传输层分别如何实现标准化。

## 为什么 MCP 是协议，而不是框架

MCP 不是 SDK，不是框架，是一种**协议约定**。

核心原因：**真正的复用发生在框架之外。** 工具一旦绑定在某个 SDK 或框架内部，就无法脱离该框架供其他智能体使用。Java SDK 写的工具，Python 智能体无法直接调用。

要实现工具的可移植、可复用、可组合，唯一方式是把工具做成**任何人、任何语言、任何框架都能访问的独立能力端点**。MCP 的本质是能力协商标准，类似于"HTTP 之于 Web"，而非"MyBatis 之于数据库"。

## Client-Server 架构：工具拆分

MCP 通过 Client-Server 结构实现工具与智能体的分离：

- **MCP Host**：智能体应用本身
- **MCP Client**：Host 内部的组件，负责与 Server 通信，一对一连接
- **MCP Server**：独立的工具服务，提供工具说明和执行逻辑

![MCP 架构](/hero/mcp-deep-arch1.png)

![Host-Client-Server 关系](/hero/mcp-deep-arch2.png)

一个 Host 可以接入多个 Client，每个 Client 连接一个 Server。工具代码不再依赖智能体的开发语言和运行环境，被包装为独立运行的能力端点。智能体只与 Server 通信，不关心工具是 Java、Python 还是其他语言实现的。

## 数据层标准化：JSON-RPC

工具拆出智能体后，需要统一的描述和通信标准。MCP 数据层基于 **JSON-RPC 2.0** 协议。

![数据层](/hero/mcp-deep-datalayer.png)

### JSON-RPC 简介

JSON-RPC 是两个服务之间通过 JSON 数据通信的协议，核心特点：

- **RPC**：在 A 服务上调用函数，实际在 B 服务上执行
- **JSON 格式**：语言无关，任何智能体都能解析
- **无状态**：每次请求独立，协议本身不记忆状态

相比 RESTful API，JSON-RPC 只关心**方法名和参数**，与大模型调用工具的逻辑天然契合。

### 请求/响应结构

请求体：

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "city": "Beijing" }
  },
  "id": 1
}
```

响应体：

```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [{ "type": "text", "text": "北京今天晴，气温 25 度" }]
  },
  "id": 1
}
```

错误结构：

```json
{
  "jsonrpc": "2.0",
  "error": { "code": -32601, "message": "Method not found" },
  "id": 1
}
```

### MCP 中的 JSON-RPC 交互阶段

**初始化** — Client 向 Server 发送 `initialize` 请求，协商协议版本和能力。初始化完成后必须发送 `notifications/initialized` 通知，否则 Server 会拒绝后续操作。

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": { "roots": { "listChanged": true } },
    "clientInfo": { "name": "ExampleClient", "version": "1.0.0" }
  }
}
```

**工具发现** — Client 调用 `tools/list` 获取所有工具的标准化描述。MCP Server 暴露三类核心能力：Resources（可读取的数据）、Tools（可执行的工具）、Prompts（提示词模板）。

**工具调用** — Client 根据大模型决策，构建 `tools/call` 请求发送给 Server，Server 解析工具名和参数后执行并返回结果。

**工具变更通知** — Server 通过 `notifications/tools/list_changed` 通知 Client 工具列表已更新，Client 重新获取最新工具列表。

## 传输层标准化

JSON-RPC 解决了"说什么语言"的问题，传输层解决"用什么通道通信"。MCP 定义了三种传输机制。

![传输层](/hero/mcp-deep-transport.png)

### Stdio（标准输入输出）

本地直连方式，Claude、Cursor 等客户端的首选。

- **原理**：进程间通信，Host fork 子进程运行 Server
- **下行**：Host 写入子进程 stdin
- **上行**：Server 写入 stdout，Host 监听
- **错误**：日志写入 stderr，不干扰 JSON 数据流
- **特点**：生命周期绑定（Host 关闭则 Server 销毁）、无网络开销、延迟极低

![Stdio 传输](/hero/mcp-deep-stdio.png)

### SSE（Server-Sent Events）

远程工具调用的早期方案，HTTP/1.1 环境下使用。

- **接收通道**：Client 发起 `GET /sse` 长连接，Server 保持连接不关闭，有消息时推送
- **发送通道**：Client 向 `/messages` 发起独立的 HTTP POST 请求
- **痛点**：需要维护两条通道、无法自动断点重连、架构复杂

![SSE 传输](/hero/mcp-deep-sse.png)

### Streamable HTTP（新）

替代 SSE 的新方案，解决读写分离的复杂度问题。

- **单一连接**：Client 只需连接 `POST /mcp` 一个端点
- **流式响应**：Server 使用分块传输编码（Chunked Encoding）实时写入响应
- **核心优势**：架构简化、双向异步、支持断点重连（Session ID + Last-Event-ID）、兼容标准 HTTP 基础设施、向后兼容 SSE

![Streamable HTTP 传输](/hero/mcp-deep-streamable-http.png)

### 三种传输方式对比

| 传输方式 | 原理 | 优势 | 典型场景 |
|---------|------|------|---------|
| **Stdio** | 父子进程 stdin/stdout | 无网络开销、延迟极低 | 本地客户端（Cursor、Claude） |
| **SSE** | GET 长连接 + POST 短连接 | 支持远程通信 | HTTP/1.1 环境下的远程工具 |
| **Streamable HTTP** | POST 流式分块传输 | 简化架构、双向异步、易恢复 | 远程工具调用，替代 SSE |

## 小结

MCP 通过 Client-Server 结构将工具从智能体中彻底解耦，通过 JSON-RPC 2.0 实现数据层标准化，通过 Stdio/SSE/Streamable HTTP 三种传输机制适配不同场景。工具成为独立的标准化服务，任何智能体都能即插即用，智能体生态从孤立模式转向协作模式。
