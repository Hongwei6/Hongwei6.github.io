---
title: "Spring AI 提示词模板：参数化与外部管理"
date: 2026-07-29
draft: false
description: "介绍 Spring AI 的 PromptTemplate 机制——占位符替换、自定义分隔符、外部文件管理提示词，实现提示词与代码解耦。"
tags: ["Spring AI", "Java", "AI", "提示词工程"]
categories: ["后端"]
cover: "/hero/tt3.png"
---

硬编码提示词会导致代码臃肿且难以维护。Spring AI 提供了 `PromptTemplate` 机制，支持占位符替换、自定义分隔符和外部文件加载，让提示词与业务代码解耦。

## 基础用法：占位符替换

`PromptTemplate` 内置了 `StTemplateRenderer`（基于 StringTemplate），通过 `{变量名}` 语法定义占位符，运行时替换为实际值：

```java
@GetMapping("/chat6")
public Flux<String> chat6(@RequestParam String message, HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");

    PromptTemplate promptTemplate = new PromptTemplate("请给我推荐几个关于{topic}的开源项目");
    promptTemplate.add("topic", message);

    return chatClient.prompt(promptTemplate.create())
            .system("你是一个专业的 GitHub 项目收集人员")
            .stream().content();
}
```

模板中的 `{topic}` 在调用 `create()` 时被替换为用户输入的实际内容。

## 多占位符替换

当模板包含多个占位符时，可以通过 `Map` 一次性传入所有变量：

```java
@GetMapping("/chat7")
public Flux<String> chat7(@RequestParam String message, HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");

    Map<String, Object> variables = new HashMap<>();
    variables.put("language", "Java");
    variables.put("topic", message);

    PromptTemplate promptTemplate = PromptTemplate.builder()
            .template("请给我推荐几个关于{topic}的开源项目，要求是和编程语言{language}相关的。")
            .variables(variables)
            .build();

    return chatClient.prompt(promptTemplate.create())
            .system("你是一个专业的 GitHub 项目收集人员")
            .stream().content();
}
```

`{topic}` 和 `{language}` 会被同时替换。Builder 模式下通过 `.variables(map)` 批量注入，代码更清晰。

## 自定义分隔符

默认使用花括号 `{}` 作为分隔符。如果提示词本身包含花括号（如 JSON 格式示例），可以自定义分隔符避免冲突：

```java
PromptTemplate promptTemplate = PromptTemplate.builder()
        .renderer(StTemplateRenderer.builder()
                .startDelimiterToken('<')
                .endDelimiterToken('>')
                .build())
        .template("告诉我 5 部由 <composer> 作曲的电影名称。")
        .build();

String prompt = promptTemplate.render(Map.of("composer", "John Williams"));
```

将分隔符改为 `<>` 后，模板中的花括号就不会被误解析。

## 外部文件管理提示词

提示词和代码混在一起会导致代码混乱，修改也不方便。更好的做法是像管理 SQL 文件一样，将提示词独立到外部文件中。

### 定义模板文件

创建 `src/main/resources/prompts/open-source-system-prompt.st`：

```text
请给我推荐几个关于{topic}的开源项目，要求是和编程语言{language}相关的。
```

### 注入并使用

通过 Spring 的 `@Value` 注解将文件内容注入到 Bean 中：

```java
@Value("classpath:prompts/open-source-system-prompt.st")
private Resource systemText;

@GetMapping("/chat2")
public Flux<String> chat2(@RequestParam String message, HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");

    Map<String, Object> variables = new HashMap<>();
    variables.put("language", "Java");
    variables.put("topic", message);

    PromptTemplate promptTemplate = PromptTemplate.builder()
            .resource(systemText)
            .variables(variables)
            .build();

    return chatClient.prompt(promptTemplate.create())
            .system("你是一个专业的 GitHub 项目收集人员")
            .stream().content();
}
```

`.resource(systemText)` 直接加载文件内容作为模板，无需手动读取文件字符串。

### 优势

| 对比项 | 硬编码 | 外部文件 |
|--------|--------|---------|
| 可读性 | 提示词散落在代码各处 | 统一目录，一目了然 |
| 可维护性 | 修改需改代码、重新编译 | 修改文件即可，无需改代码 |
| 复用性 | 难以跨项目共享 | 文件可独立管理、版本化 |

## 小结

`PromptTemplate` 解决了提示词管理的三个核心问题：

1. **参数化**：通过占位符实现动态替换，避免字符串拼接
2. **灵活性**：支持自定义分隔符，适配复杂模板场景
3. **可维护性**：外部文件管理，提示词与代码解耦

在实际项目中，建议将所有提示词统一放在 `resources/prompts/` 目录下，按功能命名（如 `summarize.st`、`translate.st`），通过 `@Value` 注入使用。
