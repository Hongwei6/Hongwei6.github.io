---
title: "Spring AI 结构化输出：让大模型返回可靠数据"
date: 2026-07-29
draft: false
description: "介绍 Spring AI 的结构化输出机制——BeanOutputConverter 原理、entity() 方法、List/Map 转换，将模型的自由文本输出映射为 Java 对象。"
tags: ["Spring AI", "Java", "AI", "结构化输出"]
categories: ["后端"]
cover: "/hero/tt1.png"
---

大模型的输出是自由格式的文本，人能看懂，但代码难以直接解析。Spring AI 提供了 `StructuredOutputConverter` 机制，在调用前后分别注入格式指令和解析逻辑，将模型输出映射为 Java 对象。

## StructuredOutputConverter 接口

```java
public interface StructuredOutputConverter<T> extends Converter<String, T>, FormatProvider {
}
```

接口继承了两个能力：

- **FormatProvider**：在 LLM 调用**前**，向提示词中注入格式指令，告诉模型应该输出什么结构
- **Converter**：在 LLM 调用**后**，将模型返回的原始文本解析为目标类型

最常用的实现是 `BeanOutputConverter`，它能将 JSON 输出自动映射为 Java Bean。

## BeanOutputConverter 原理

`BeanOutputConverter` 的 `getFormat()` 方法返回一段格式指令：

```java
@Override
public String getFormat() {
    String template = """
            Your response should be in JSON format.
            Do not include any explanations, only provide a RFC8259 compliant JSON response
            following this format without deviation.
            Do not include markdown code blocks in your response.
            Remove the ```json markdown from the output.
            Here is the JSON Schema instance your output must adhere to:
            ```%s```
            """;
    return String.format(template, this.jsonSchema);
}
```

本质就是一段提示词，告诉模型：按这个 JSON Schema 输出，不要包含解释和 markdown 代码块。这段指令通过 `{format}` 占位符注入到 Prompt 中，模型就会按照指定结构返回 JSON。

## 基础用法：输出转 Bean

### 定义目标类型

用 Java `record` 定义数据结构：

```java
public record Book(
        @JsonPropertyDescription("书籍名称") String title,
        @JsonPropertyDescription("作者") String author,
        @JsonPropertyDescription("书籍介绍") String description,
        @JsonPropertyDescription("价格") BigDecimal price) {
}
```

`@JsonPropertyDescription` 会生成到 JSON Schema 中，帮助模型理解每个字段的含义，提升输出准确率。

### 手动注入格式指令

```java
@GetMapping("/chat")
public Flux<String> chat(HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");

    BeanOutputConverter<Book> beanOutputConverter = new BeanOutputConverter<>(Book.class);

    PromptTemplate promptTemplate = new PromptTemplate("""
            请帮我推荐1本java相关的书
            {format}
            """);

    return chatClient.prompt(promptTemplate.create(
            Map.of("format", beanOutputConverter.getFormat())))
            .system("你是一个专业的图书推荐人员")
            .stream().content();
}
```

通过 `beanOutputConverter.getFormat()` 获取格式指令，注入到 `{format}` 占位符中。模型返回的 JSON 结构会严格遵循 Book 的字段定义。

### 输出转 Bean

拿到完整结果后，调用 `convert()` 方法解析：

```java
@GetMapping("/chat1")
public String chat1(HttpServletResponse response) {
    response.setCharacterEncoding("UTF-8");

    BeanOutputConverter<Book> beanOutputConverter = new BeanOutputConverter<>(Book.class);

    PromptTemplate promptTemplate = new PromptTemplate("""
            请帮我推荐1本java相关的书
            {format}
            """);

    String result = chatClient.prompt(promptTemplate.create(
                    Map.of("format", beanOutputConverter.getFormat())))
            .system("你是一个专业的图书推荐人员")
            .call().content();

    Book book = beanOutputConverter.convert(result);
    System.out.println(book);
    return result;
}
```

注意：`convert()` 只支持同步调用（`call().content()`），不支持 `Flux<String>` 流式输出——必须拿到完整结果才能解析。

## 更简洁的方式：entity()

手动管理 `{format}` 占位符比较繁琐。Spring AI 在 `ChatClient` 中提供了 `entity()` 方法，内部自动完成格式注入和结果转换：

```java
@GetMapping("/chat2")
public String chat2(HttpServletResponse response) {
    Book book = chatClient.prompt("请帮我推荐几本java相关的书")
            .system("你是一个专业的图书推荐人员")
            .call().entity(Book.class);
    return book.toString();
}
```

一行 `.entity(Book.class)` 等价于：创建 `BeanOutputConverter` → 注入格式指令 → 调用模型 → 解析输出。底层流程：

1. 创建 `BeanOutputConverter<Book>`
2. 调用 `getFormat()` 生成 JSON Schema 指令
3. 将指令附加到 Prompt 中发送给模型
4. 模型返回 JSON 后，调用 `convert()` 映射为 `Book` 对象

## 转换为 List 和 Map

除了 Bean，Spring AI 还支持转换为集合类型。

### List

使用 `ParameterizedTypeReference` 可以拿到泛型化的 List：

```java
List<Book> result = chatClient.prompt("请帮我推荐几本java相关的书")
        .system("你是一个专业的图书推荐人员")
        .call().entity(new ParameterizedTypeReference<List<Book>>() {});
```

这种方式底层仍然使用 `BeanOutputConverter`，能正确解析为 `List<Book>`。

内置的 `ListOutputConverter` 只能返回 `List<String>`，无法直接映射为 `List<Bean>`，实际开发中推荐用 `ParameterizedTypeReference` 的方式。

### Map

```java
Map<String, Object> result = chatClient.prompt("请帮我推荐几本java相关的书")
        .system("你是一个专业的图书推荐人员")
        .call().entity(new MapOutputConverter());
```

`MapOutputConverter` 返回的是 `Map<String, Object>`，字段类型不够精确。如果需要结构化的 Map，建议在提示词中明确定义 key/value 结构，或者先拿到 List 再自行转换。

## 小结

| 方式 | 适用场景 | 限制 |
|------|---------|------|
| `BeanOutputConverter` + 手动注入 | 需要精细控制格式指令时 | 代码较冗长 |
| `entity(Class)` | 快速将输出转为单个 Bean | 仅支持同步调用 |
| `ParameterizedTypeReference<List<T>>` | 返回一组对象 | 仅支持同步调用 |
| `MapOutputConverter` | 返回动态结构 | 字段类型不精确 |

结构化输出的核心思路是：**用 JSON Schema 约束模型输出，用 Converter 解析为 Java 类型**。`entity()` 方法封装了这个流程的大部分场景，优先使用即可。
