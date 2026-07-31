---
title: "Spring AI 提示词工程：从基础到进阶"
date: 2026-07-29
draft: false
description: "在 Spring AI 中实践提示词工程——系统提示词、Few-shot、输出格式约束、分步指令和思维链，用代码示例演示每种技巧的用法。"
tags: ["Spring AI", "Java", "AI", "提示词工程"]
categories: ["后端"]
cover: "/hero/tt2.png"
---

提示词工程是大模型应用开发中最基础也最关键的环节。Spring AI 遵循 OpenAI 的提示词规范，将输入分为 **System Prompt** 和 **User Prompt** 两层，并通过 `ChatClient` 的流式 API 提供了灵活的组装方式。本文通过代码示例演示几种常用的提示词技巧。

## 系统提示词与用户提示词

Spring AI 中，提示词由两部分组成：

- **System Prompt**：定义模型的角色、行为约束和输出风格
- **User Prompt**：用户的实际输入

通过 `ChatClient` 可以分别设置：

```java
@RestController
@RequestMapping("/ai/prompt")
public class PromptEngineerController implements InitializingBean {

    @Autowired
    private DashScopeChatModel chatModel;

    private ChatClient chatClient;

    @GetMapping("/chat")
    public Flux<String> chat(@RequestParam String message, HttpServletResponse response) {
        response.setCharacterEncoding("UTF-8");
        return chatClient.prompt()
                .system("你是一个毒舌博主，说话很噎人，请根据用户问题，怼他")
                .user(message)
                .stream().content();
    }

    @Override
    public void afterPropertiesSet() {
        this.chatClient = ChatClient.builder(chatModel).build();
    }
}
```

调用 `/ai/prompt/chat?message=我饿了`，模型会以毒舌博主的语气回复。System Prompt 设定了角色身份，User Prompt 传入用户的具体问题。

如果某些提示词是固定的，可以在构建 `ChatClient` 时通过 `defaultSystem()` 和 `defaultUser()` 预设，后续调用会自动应用。

## Few-shot：用示例引导输出

Few-shot 是在提示词中提供少量示例，让模型从示例中学习输出模式。这种方式比纯文字描述规则更有效。

### 数字推理示例

```java
@GetMapping("/chat2")
public Flux<String> chat2(@RequestParam String message, HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");
    return chatClient.prompt("""
            请根据用户输入的数字，给出结果，不需要思考过程，直接给出数字结果即可，
            推理过程参考：
            1 = 5
            2 = 10
            3 = 15
            ，如果用户给的不是个数字，请回复:无法回答，请输入数字
            """)
            .system("你是个ai")
            .user(message)
            .stream().content();
}
```

通过 `1=5, 2=10, 3=15` 三个示例，模型能推断出规律是「乘以 5」，并对新的输入给出结果。

### 文本改写示例

```java
@GetMapping("/shot")
public String shot(String message) {
    return chatClient.prompt().system("""
            请你根据用户输入的问题做改写，主要有以下改写策略：
            1、改写其中的错别字。
            2、做内容精简，帮用户的一堆废话精简成简单的一句话
            可以参考以下实例：

            Input：ni好
            Output ：{"错别字改写":"你好","内容精简":""}

            Input：我今天心情不错，我想知道今天是什么天气才让我心情这么好的？
            Output ：{"错别字改写":"","内容精简":"今天是什么天气？"}
            """)
            .user(message)
            .call().content();
}
```

两个示例分别展示了「错别字修正」和「内容精简」两种改写策略，模型会按照相同的 JSON 格式处理新的输入。

## 指定输出格式

通过提示词明确要求输出格式，可以让模型返回结构化数据：

```java
@GetMapping("/chat3")
public Flux<String> chat3(@RequestParam String message, HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");
    return chatClient.prompt(
            "请生成包括书名、作者和类别的三本虚构的、非真实存在的中文书籍清单，" +
            "并以 JSON 格式提供，其中包含以下键:book_id、title、author、genre。")
            .system("你是一个富有创意的作家")
            .user(message)
            .stream().content();
}
```

这种方式简单直接，但依赖模型对指令的遵循程度。对于更严格的结构化输出需求，Spring AI 提供了 `Structured Output` 机制，后续在结构化输出章节展开。

## 指定步骤

将复杂任务拆解为明确的步骤，可以引导模型按顺序执行：

```java
@GetMapping("/chat4")
public Flux<String> chat4(@RequestParam String message, HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");
    return chatClient.prompt("""
            执行以下操作：
                1-用一句话概括下面文本。
                2-将摘要翻译成英语。
                3-在英语摘要中列出每个人名。
                4-输出一个 JSON 对象，其中包含以下键：english_summary，num_names。
            请用换行符分隔您的答案。
            """)
            .system("你是个ai")
            .user(message)
            .stream().content();
}
```

分步指令的优势在于每个步骤的输出都是下一步的输入，模型需要按序处理，减少了跳步或遗漏的风险。

## 思维链（Chain of Thought）

思维链是一种让模型「展示推理过程」的技巧。通过在提示词中要求模型逐步思考，可以显著提升复杂推理任务的准确率。

```java
@GetMapping("/chat5")
public Flux<String> chat5(@RequestParam String message, HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");
    return chatClient.prompt("""
            一个水果摊有5箱苹果，每箱重15公斤。今天卖掉了35公斤，还剩下多少公斤苹果？
            请一步一步思考，并给出最终答案。
            """)
            .system("你是个ai")
            .user(message)
            .stream().content();
}
```

关键在于「请一步一步思考」这句话。它迫使模型将推理过程显式化：先算总量（5×15=75），再减去已售（75-35=40），最后给出答案。相比直接回答，分步推理减少了计算错误。

## 小结

| 技巧 | 核心思路 | 适用场景 |
|------|---------|---------|
| System/User Prompt | 分层控制角色和输入 | 所有场景 |
| Few-shot | 用示例替代规则描述 | 格式转换、分类、风格模仿 |
| 指定输出格式 | 约束返回结构 | 需要结构化数据时 |
| 指定步骤 | 拆解任务为有序步骤 | 多步骤复合任务 |
| 思维链 | 要求展示推理过程 | 数学计算、逻辑推理 |

这些技巧可以组合使用——一个 System Prompt 可以同时设定角色、提供 Few-shot 示例、指定输出格式和推理步骤。实际开发中，根据任务复杂度灵活组合即可。
