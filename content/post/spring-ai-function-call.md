---
title: "使用 Spring AI 实现 Function Calling"
date: 2026-08-12T09:00:00+08:00
draft: false
description: "在 Spring AI 中通过 @Tool/@ToolParam 定义新工具、用 Function Bean 包装已有方法，以及控制工具是否自动执行。"
tags: ["Spring AI", "Java", "AI", "Function Calling"]
categories: ["技术"]
---

[上一篇](/post/langchain4j-function-call/)讲清了 Function Calling 的原理：模型只负责「决策」（调什么、传什么），应用端负责「执行」。Spring AI 在这之上做了封装，提供了两种把代码暴露给模型的方式。

## 把已有方法转成工具

如果已经有一个现成的 Spring 服务，想把它变成模型可调用的工具，可以定义一个返回 `Function` 类型的 Bean，内部委托给已有方法：

```java
@Configuration
public class FunctionCallConfiguration {

    @Bean
    @Description("根据用户输入的时区获取该时区的当前时间")
    public Function<TimeService.Request, TimeService.Response> getTimeFunction(TimeService timeService) {
        return timeService::getTimeByZoneId;
    }
}
```

这里定义一个 Bean，返回值是 `Function` 类型，方法参数注入已有的 Spring 服务，方法体中用方法引用 `timeService::getTimeByZoneId` 直接复用现有逻辑。`@Description` 必须写清楚这个 Function 的用途——模型据此判断什么时候该调用它。

`Function<T, R>` 有两个泛型参数：

```java
@FunctionalInterface
public interface Function<T, R> {
    // T 表示入参类型，R 表示出参类型
}
```

入参和出参就是 Request、Response。它们的定义同样需要给字段加上描述：

```java
@Service
public class TimeService {

    public Response getTimeByZoneId(Request request) {
        ZoneId zid = ZoneId.of(request.zoneId);
        ZonedDateTime zonedDateTime = ZonedDateTime.now(zid);
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss z");
        return new Response(zonedDateTime.format(formatter));
    }

    public record Request(
            @JsonProperty(required = true, value = "zoneId")
            @JsonPropertyDescription("时区，比如 Asia/Shanghai") String zoneId) {
    }

    public record Response(String time) {
    }
}
```

入参字段上用 `@JsonPropertyDescription` 标注说明，让模型清楚每个参数的含义，才能正确生成调用入参。

## 定义新工具

如果没有现成方法可复用，要全新定义一个工具，直接用 `@Tool` 注解：

```java
public class TimeTools {

    @Tool(description = "Get time by zone id")
    public String getTimeByZoneId(
            @ToolParam(description = "Time zone id, such as Asia/Shanghai") String zoneId) {
        ZoneId zid = ZoneId.of(zoneId);
        ZonedDateTime zonedDateTime = ZonedDateTime.now(zid);
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss z");
        return zonedDateTime.format(formatter);
    }
}
```

`@Tool` 把一个方法声明为工具，`@ToolParam` 描述每个入参。这种方式比 `Function` Bean 更直接，是 Spring AI 推荐的工具定义方式。

定义好后，把工具实例传给 `ChatClient` 的 `tools` 方法即可使用：

```java
@GetMapping("/functionCall1")
public Flux<String> functionCall1(HttpServletResponse response, String city) {
    response.setCharacterEncoding("UTF-8");
    return chatClient
            .prompt()
            .tools(new TimeTools())
            .user(city + "现在几点了？")
            .stream()
            .content();
}
```

## 工具是否自动执行

默认情况下，模型一旦决策需要调用工具，Spring AI 会**直接自动执行**该工具，开发者无感知。

如果想自己接管工具的调用过程（例如后续实现 ReAct Agent 时需要手动控制每一步），可以通过 `internalToolExecutionEnabled` 参数关闭自动执行：

```java
internalToolExecutionEnabled(false)
```

设置为 `false` 后，Spring AI 不会再自动调用工具，而把工具调用决策交还给开发者处理——这正好对应 [Function Calling 原理](/post/langchain4j-function-call/)里「决策与执行分离」的原始形态。

## 调用第三方工具

Spring AI Alibaba 还集成了一批现成的第三方工具，可以直接复用，避免重复造轮子。可用工具列表见官方文档：

https://java2ai.com/docs/1.0.0.2/practices/integrations/tool-calling/
