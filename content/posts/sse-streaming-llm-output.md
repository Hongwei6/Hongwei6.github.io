---
title: "基于 SSE 实现大模型流式输出"
date: 2026-07-27
draft: false
description: "大模型回答为什么是一个字一个字蹦出来的?拆解 SSE 流式输出的协议格式,以及 Spring 中的三种实现方式。"
tags: ["Spring AI", "SSE", "Java", "AI"]
categories: ["后端"]
---

用 HTTP Client 同步调用大模型时,接口要等整段回复生成完才一次性返回,首字延迟很高。主流 AI 对话产品之所以能「边生成边显示」,靠的是**流式输出**(streaming)—— 内容在生成过程中就被逐段推送给客户端,而不是攒齐了再返回。

在 Web 端实现服务端到客户端的单向推送,最常用的协议就是 **SSE**(Server-Sent Events)。它基于 HTTP,允许服务器在一条长连接上持续向客户端推送数据。因为能持续推送,模型每吐出一个 token 就可以立刻转发出去,前端逐段拼接,便得到了打字机效果。

## SSE 的协议格式

SSE 本质上是一次标准的 HTTP 请求,只是连接会保持打开,由服务器持续写入数据。以调用百炼(OpenAI 兼容接口)为例,开启 `stream=true`:

```bash
curl -X POST https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  --no-buffer \
  -d '{
    "model": "qwen-plus",
    "messages": [
      {"role": "user", "content": "你是谁?"}
    ],
    "stream": true
  }'
```

连接建立后不会立即关闭,服务器随时可以向客户端写入,客户端无需重新发起请求。数据以 UTF-8 文本、按「事件流」分段返回,每段以 `data:` 前缀开头。响应体大致长这样:

```text
data: {"choices":[{"delta":{"content":"","role":"assistant"},"index":0,"finish_reason":null}],"object":"chat.completion.chunk","model":"qwen-plus","id":"chatcmpl-4bbf022b-..."}
data: {"choices":[{"delta":{"content":"我是"},"index":0,"finish_reason":null}],"object":"chat.completion.chunk","model":"qwen-plus","id":"chatcmpl-4bbf022b-..."}
data: {"choices":[{"delta":{"content":"通义千"},"index":0,"finish_reason":null}],"object":"chat.completion.chunk","model":"qwen-plus","id":"chatcmpl-4bbf022b-..."}
data: {"choices":[{"delta":{"content":"问（Q"},"index":0,"finish_reason":null}],"object":"chat.completion.chunk","model":"qwen-plus","id":"chatcmpl-4bbf022b-..."}
data: {"choices":[{"delta":{"content":"wen），是"},"index":0,"finish_reason":null}],"object":"chat.completion.chunk","model":"qwen-plus","id":"chatcmpl-4bbf022b-..."}
data: {"choices":[{"delta":{"content":"阿里巴巴集团旗下的通义实验室"},"index":0,"finish_reason":null}],"object":"chat.completion.chunk","model":"qwen-plus","id":"chatcmpl-4bbf022b-..."}
```

关键在 `delta.content` 字段——每一帧只携带一小片文字(`我是`、`通义千`、`问（Q`...),客户端逐帧拼接,就还原出了完整回答。`finish_reason` 在最后一帧变为非 `null`,标志流结束。

## Spring 中的三种实现

在 Spring 里落地 SSE,常见有三种写法。

### 1. SseEmitter

`SseEmitter` 是 Spring MVC 提供的服务端推送抽象。它的模型是「手动驱动」:

- `send()` 推送一帧事件;
- 全部输出完成后调 `complete()` 通知客户端;
- 出错则调 `completeWithError()` 把异常透传给客户端。

```java
@RestController
@RequestMapping("/stream/output")
public class SseEmitterController {

    @GetMapping("/sse/emitter")
    public SseEmitter sse() {
        SseEmitter emitter = new SseEmitter(60_000L); // 超时时间

        Executors.newSingleThreadExecutor().submit(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    emitter.send("Message " + i);
                    Thread.sleep(1000);
                }
                emitter.complete();
            } catch (Exception ex) {
                emitter.completeWithError(ex);
            }
        });

        return emitter;
    }
}
```

这种写法需要自己管理线程、发送、异常和完成回调,样板代码偏多。

### 2. ResponseEntity + StreamingResponseBody

`StreamingResponseBody` 是一个函数式接口,直接持有 `OutputStream`,把数据逐步写入响应流,实现非阻塞的异步传输。Spring 会在响应提交前才回调 `writeTo(OutputStream)`,每次写入后手动 `flush()` 强制刷新缓冲区,客户端就能实时收到内容。

```java
@GetMapping("/sse/streaming")
public ResponseEntity<StreamingResponseBody> chat() {
    StreamingResponseBody body = outputStream -> {
        for (int i = 0; i < 10; i++) {
            String data = "data chunk " + i + "\n";
            outputStream.write(data.getBytes(StandardCharsets.UTF_8));
            outputStream.flush();
            try {
                Thread.sleep(500); // 模拟延迟
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    };

    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_TYPE, MediaType.TEXT_EVENT_STREAM_VALUE)
        .body(body);
}
```

相比 `SseEmitter` 灵活一些,但仍然要手动控制 `flush()` 和响应头。

### 3. Flux + WebFlux

Spring WebFlux 是响应式 Web 框架,底层用 Netty 非阻塞 I/O。返回一个 `Flux<String>`,框架会自动把它当作事件流推送给客户端——声明式、零样板。

```java
@GetMapping(value = "/sse/flux")
public Flux<String> fluxStream() {
    return Flux.interval(Duration.ofSeconds(1))
        .map(seq -> "Stream element - " + seq);
}
```

需要引入依赖:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

代码量已经说明问题。更重要的是,这条路径与 Spring AI 的设计一脉相承——Spring AI 对接大模型时本就大量返回 `Flux`,用 WebFlux 可以把「上游模型流」和「下游 SSE 推送」无缝串起来,不需要在中间做线程转换或手动 `flush`。

## 三种方式对比

| 方式 | 框架 | 特点 |
|------|------|------|
| `SseEmitter` | Spring MVC | 手动管理线程与发送,需自行处理异常与完成回调 |
| `StreamingResponseBody` | Spring MVC | 直接操作 `OutputStream`,需手动 `flush()` 并设置响应头 |
| `Flux<String>` | Spring WebFlux | 响应式、声明式,代码最少,与 Spring AI 风格一致 |

已有的 Spring MVC 项目里,`SseEmitter` 改动最小;新项目或要对接 Spring AI 的场景,直接用 WebFlux 的 `Flux` 是最省心的选择,也是后续接入大模型流式接口的基础。
