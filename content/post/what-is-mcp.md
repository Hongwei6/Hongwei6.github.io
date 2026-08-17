---
title: "什么是 MCP？模型上下文协议与 Function Call 的区别"
date: 2026-08-17
draft: false
description: "MCP（Model Context Protocol）是 Anthropic 开源的通用协议，用于标准化智能体与外部工具的连接方式，解决传统 Function Call 的强耦合和重复开发问题。"
tags: ["MCP", "AI", "智能体", "Function Call"]
categories: ["AI"]
cover: "/hero/mcp-arch-overview.png"
---

MCP（Model Context Protocol，模型上下文协议）是 Anthropic 开源的通用标准协议，用于连接智能体应用与外部工具。它不是具体的框架或技术，而是一套规范——类似于智能体领域的 USB 接口：只要工具端实现 MCP 协议，任何支持 MCP 的客户端（Claude Desktop、Cursor、Cline 等）都能即插即用。

![MCP 概览](/hero/mcp-arch-overview.png)

## 解决的问题

传统方式下，让大模型调用外部工具需要为每个智能体编写专用的 function call 代码。换一个智能体，代码就要重写。工具规模扩大后，耦合度急剧上升。

MCP 将工具封装为独立的 MCP Server，客户端通过标准协议调用，无需重复造轮子。社区已有大量现成的 MCP Server 可直接使用。

![MCP 架构](/hero/mcp-architecture.png)

MCP 协议层作为枢纽，连接左侧的智能体客户端和右侧的工具，将点对点的硬编码集成简化为统一接口调用。

## MCP vs Function Call

**Function Call 的痛点：** 每次工具集成都是一次完整开发，工具逻辑硬编码在智能体内部。智能体数量增加时，接入代码重复书写；工具规模扩大后，扩展成本急剧上升。

**MCP 的做法：** 采用 Client-Server 分层架构，工具以独立 MCP Server 暴露能力，Client 通过 JSON-RPC 与 Server 通信，智能体统一管理权限和上下文。工具接入无需硬编码，功能边界清晰，工具可复用。

![MCP Client-Server 架构](/hero/mcp-client-server.png)

简单类比：Function Call 是"自己带工具上班"，MCP 是"去工具商店按需取用"。

## MCP 工作流程

![MCP 工作流程](/hero/mcp-workflow.jpg)

**阶段一：初始化** — 智能体通过 MCP 协议向所有连接的 MCP Server 请求工具说明书（JSON-RPC），获取标准化的 JSON 格式工具定义。

**阶段二：决策** — 智能体将用户问题和工具说明一起发给大模型，模型返回结构化的工具调用指令。

**阶段三：调用** — 智能体根据指令，通过 MCP 协议请求对应的 MCP Server 执行操作，Server 返回原始执行结果。

**阶段四：总结** — 智能体将原始问题 + 工具执行结果 + 对话历史发给大模型，生成最终回复。

初始化和调用两个阶段都依赖 MCP 协议：初始化阶段解决工具说明书的标准化获取，调用阶段负责将模型指令转发给 Server 执行。

## 工具调用的本质

在工具调用中，大模型始终扮演决策者和规划者——只负责输出"调用哪个工具、传什么参数"的指令。实际执行者是智能体框架本身。大模型不直接参与服务调用。

从底层交互看，Function Call 和 MCP 都是同一套能力：**向模型发送带工具定义（schema）的对话请求，模型返回结构化调用指令**。

![工具调用示例](/hero/mcp-tool-call-example.png)

区别在于：Function Call 要求应用自行构造工具说明并封装实现；MCP 将工具说明和执行逻辑都封装为独立外部服务，智能体通过协议即可调用。
