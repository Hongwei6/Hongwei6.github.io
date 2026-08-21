---
title: "MCP 调试工具"
date: 2026-08-21T16:52:00+08:00
draft: false
description: "MCP Inspector 官方可视化调试工具的使用指南，支持 Stdio、SSE、Streamable HTTP 三种传输模式"
tags: ["Spring AI", "Java", "MCP", "调试"]
categories: ["后端"]
cover: "/hero/mcp-inspector/04-tools-list.png"
---

**MCP Inspector** 是 MCP 官方推出的可视化调测工具，用于验证 MCP Server 实现是否符合规范，支持查看 tools、resources、events 等内容，是目前最方便的 MCP 开发辅助工具。

## 安装

要求本地有 Node.js 环境：https://nodejs.org/zh-cn

```bash
npx @modelcontextprotocol/inspector@latest
```

![安装启动](/hero/mcp-inspector/01-install.png)

启动后访问：http://localhost:6274/

右上角 Transport Type 包含 3 种类型：Stdio、SSE、Streamable

![传输类型](/hero/mcp-inspector/02-transport-types.png)

## Streamable HTTP

URL 输入框填写：

```
http://127.0.0.1:8004/stream/test/api/mcp
```

点击 Connect：

![Streamable 连接](/hero/mcp-inspector/03-streamable-connect.png)

连接成功后，点击 Tools 页签可查看所有工具列表，并支持在线调试：

**点击工具名称 → 填写参数 → Run Tool → 获取结果**

![工具列表](/hero/mcp-inspector/04-tools-list.png)

## SSE

URL 输入：

```
http://localhost:8003/test/sse
```

> **注意：** 旧版本（0.7.0）不支持带前缀的 SSE，新版本已修复此问题。该工具不支持自签名证书，本地调试需使用 HTTP。生产环境通常通过 Nginx 转发 HTTPS。

![SSE 连接](/hero/mcp-inspector/05-sse-connect.png)

## Stdio

Command 输入：

```
java -jar /path/to/mcp-server-stdio.jar
```

![Stdio 连接](/hero/mcp-inspector/06-stdio-connect.png)
