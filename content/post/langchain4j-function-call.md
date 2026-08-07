---
title: "理解 Function Calling"
date: 2026-08-07T11:00:00+08:00
draft: false
description: "剖析大模型 Function Calling（Tool Calling）的完整流程：函数定义、工具如何注入提示词、以及模型与应用端各自承担的职责。"
tags: ["LangChain4j", "Java", "AI", "LLM", "Function Calling"]
categories: ["技术"]
---

大模型自身存在局限——无法获取最新信息、无法访问私域知识。Function Calling（又称 Tool Calling）让大模型能够与外部交互，从而获取训练数据之外的数据。

例如问模型「今天的天气如何」，模型训练时没有这部分数据，输出必然是错的。但如果提供一个函数（工具），就能通过函数调用拿到真实的天气信息。

> 本文为表述方便，Function、Tool、函数、工具指代同一概念。

需要特别澄清一个常见误解：Function Calling 听上去像是模型自己调用工具，**但大模型并不负责函数的调用和执行**。模型的作用是根据用户问题理解需求，判断「该调用哪个函数、需要什么参数」；真正的函数执行由应用端（Python 或 Java 代码）完成。

## Function Calling 的过程

OpenAI 给出的函数调用交互流程如下：

![Function Calling 交互流程](/images/posts/langchain4j-function-call/function-call-flow.png)

关键在于——函数的执行调用靠开发者完成，借助应用代码来落地。整个流程可以归纳为 5 步：

1. 向模型发送包含可调用工具的请求
2. 从模型接收工具调用决策（包括具体工具和参数）
3. 在应用端执行代码，使用工具调用的入参
4. 用工具输出向模型发起第二次请求（带工具调用结果）
5. 接收模型返回的最终响应（或更多工具调用）

## 函数定义

要让模型做 Function Calling，首先要定义函数（工具）。OpenAI 规范中，一个函数定义包含以下字段：

| 字段 | 描述 |
|------|------|
| `type` | 固定值：`function` |
| `name` | 函数名称（例如 `get_weather`） |
| `description` | 何时以及如何使用该函数的详细信息 |
| `parameters` | 函数输入参数的 JSON Schema |
| `strict` | 是否对函数调用执行严格模式 |

以官方示例为例：

```json
{
  "type": "function",
  "name": "get_weather",
  "description": "Retrieves current weather for the given location.",
  "parameters": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "City and country e.g. Bogotá, Colombia"
      },
      "units": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "Units the temperature will be returned in."
      }
    },
    "required": ["location", "units"],
    "additionalProperties": false
  },
  "strict": true
}
```

当设置 `strict: true` 时开启结构化输出，确保模型为 Function Calling 生成的参数与函数定义中的 JSON Schema 完全匹配（详见 [Introducing Structured Outputs](https://openai.com/index/introducing-structured-outputs-in-the-api/)）。

## 工具如何注入提示词

有了函数定义后，就可以向模型发送请求。一个值得探究的问题是：工具定义是如何变成模型能理解的提示词的？

开源模型的答案藏在 `chat_template` 里——它定义了用户输入与模型输出之间的拼装模板。以 Qwen3-235B-A22B 为例（[HuggingFace 模板](https://huggingface.co/Qwen/Qwen3-235B-A22B?chat_template=default)），模板开头会判断是否有工具：

![Qwen chat_template 中的工具处理](/images/posts/langchain4j-function-call/qwen-chat-template.png)

其核心逻辑是：**如果有工具，就在 system 段拼装一段与工具调用相关的 prompt**，把工具签名和调用约定注入进去：

```jinja
{%- if tools %}
    {{- '<|im_start|>system\n' }}
    {%- if messages[0].role == 'system' %}
        {{- messages[0].content + '\n\n' }}
    {%- endif %}
    {{- "# Tools\n\nYou may call one or more functions to assist with the user query.\n\n"
        + "You are provided with function signatures within <tools></tools> XML tags:\n<tools>" }}
    {%- for tool in tools %}
        {{- "\n" }}
        {{- tool | tojson }}
    {%- endfor %}
    {{- "\n</tools>\n\nFor each function call, return a json object with function name and arguments "
        + "within <tool_call></tool_call> XML tags:\n<tool_call>\n{\"name\": <function-name>, "
        + "\"arguments\": <args-json-object>}\n</tool_call><|im_end|>\n" }}
{%- else %}
    ...
{%- endif %}
```

也就是说，所有工具会被序列化成 JSON，连同调用格式说明一起塞进 system 消息。

### 一个完整的请求示例

假设应用发送这样的请求（带一个 `get_horoscope` 工具）：

```json
{
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "What is my horoscope? I am an Aquarius." }
  ],
  "tools": [
    {
      "type": "function",
      "name": "get_horoscope",
      "description": "Get today's horoscope for an astrological sign.",
      "parameters": {
        "type": "object",
        "properties": {
          "sign": {
            "type": "string",
            "description": "An astrological sign like Taurus or Aquarius"
          }
        },
        "required": ["sign"]
      }
    }
  ]
}
```

经过 chat_template 拼装后，模型实际收到的提示词是：

```text
<|im_start|>system
You are a helpful assistant.
# Tools
You may call one or more functions to assist with the user query.
You are provided with function signatures within <tools></tools> XML tags:
<tools>
{"type": "function", "name": "get_horoscope", "description": "Get today's horoscope for an astrological sign.", "parameters": {"type": "object", "properties": {"sign": {"type": "string", "description": "An astrological sign like Taurus or Aquarius"}}, "required": ["sign"]}}
</tools>
For each function call, return a json object with function name and arguments within <tool_call></tool_call> XML tags:
<tool_call>
{"name": <function-name>, "arguments": <args-json-object>}
</tool_call><|im_end|>
<|im_start|>user
What is my horoscope? I am an Aquarius.<|im_end|>
<|im_start|>assistant
```

其中 `<|im_start|>`、`<|im_end|>` 是 special tokens（特殊标记），用于让模型区分不同段落。

### 模型的返回

把上述提示词直接喂给模型，第一次返回的就是一个 Function Call 结果——告诉调用方该用哪个函数、参数是什么：

![模型返回的 Function Call](/images/posts/langchain4j-function-call/function-call-result.png)

应用端拿到这个决策后执行 `get_horoscope`，再把执行结果回传给模型；模型据此组装最终回答，例如「星座运势是 Everything is good!」。这恰好对应前面流程图里的第 4、5 步。

## 小结

Function Calling 的本质是：**模型只负责「决策」（调什么、传什么），应用端负责「执行」（真正调用并拿到结果）**。工具定义通过 chat_template 被序列化成 system 段的 prompt，模型基于这段约定产出结构化的工具调用决策，再由应用代码完成真正的函数执行与结果回传。理解这一点，也就理解了 [LangChain4j 低层次 API](/post/langchain4j-low-level-api/) 中那段繁琐工具调用代码每一步在做什么——它正是在手动完成这套「决策→执行→回传」的循环。
