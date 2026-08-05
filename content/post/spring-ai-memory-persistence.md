---
title: "Spring AI 记忆管理：基于 MySQL 的持久化方案"
date: 2026-08-05T10:00:00+08:00
draft: false
description: "介绍 Spring AI 中 ChatMemoryRepository 的架构设计，以及如何使用 JDBC 将对话记忆持久化到 MySQL。"
tags: ["Spring AI", "Java", "AI", "记忆管理"]
categories: ["技术"]
---

## 背景

Spring AI 提供了 `MessageWindowChatMemory` 作为默认的记忆实现，它基于内存维护一个固定大小的消息窗口（默认 20 条）。当消息数量超过限制时，旧消息会被逐出。

这种实现的问题很明显：应用重启后，所有对话记忆都会丢失。

## ChatMemoryRepository 架构

Spring AI 的记忆存储基于 `ChatMemoryRepository` 接口扩展，提供了多种持久化实现：

![ChatMemoryRepository 架构](/images/posts/spring-ai-memory-persistence/architecture.png)

默认的 `InMemoryChatMemoryRepository` 使用 `ConcurrentHashMap` 存储数据，本质上还是内存方案。除此之外，官方还提供了以下实现：

| 实现 | 存储 | 适用场景 |
|------|------|----------|
| JdbcChatMemoryRepository | 关系型数据库 | 通用场景，支持 PostgreSQL、MySQL、SQL Server、Oracle |
| CassandraChatMemoryRepository | Apache Cassandra | 高可用、需要 TTL 功能的场景 |
| Neo4jChatMemoryRepository | Neo4j | 需要图查询能力的场景 |
| CosmosDBChatMemoryRepository | Azure Cosmos DB | 全球分布式部署 |
| MongoChatMemoryRepository | MongoDB | 文档型数据库偏好 |

## MySQL 持久化实现

### 添加依赖

使用 Spring AI 提供的 JDBC 记忆仓库 starter：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
    <version>1.1.0</version>
</dependency>
```

这个 starter 会自动引入两个包：
- `spring-ai-model-chat-memory-repository-jdbc`：JDBC 操作实现
- `spring-ai-autoconfigure-model-chat-memory-repository-jdbc`：自动配置

还需要数据库驱动：

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
</dependency>
```

### 配置

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/spring_ai_db?useUnicode=true&characterEncoding=UTF-8
    username: root
    password: your_password
    driver-class-name: com.mysql.cj.jdbc.Driver
  ai:
    chat:
      memory:
        repository:
          jdbc:
            platform: mysql
            initialize-schema: always
```

`platform` 指定数据库类型，`initialize-schema: always` 会在启动时自动建表。支持的平台类型如下：

![支持的数据库平台](/images/posts/spring-ai-memory-persistence/platform-config.png)

### 建表语句

如果自动建表失败（比如权限问题），可以手动创建：

```sql
CREATE TABLE `spring_ai_chat_memory` (
  `conversation_id` varchar(36) CHARACTER SET utf8mb4 NOT NULL,
  `content` text CHARACTER SET utf8mb4 NOT NULL,
  `type` enum('USER','ASSISTANT','SYSTEM','TOOL') CHARACTER SET utf8mb4 NOT NULL,
  `timestamp` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  KEY `SPRING_AI_CHAT_MEMORY_CONVERSATION_ID_TIMESTAMP_IDX` (`conversation_id`,`timestamp`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

> 注意：官方提供的 SQL 可能存在乱码问题，建议使用上述经过验证的版本。

### 代码实现

定义 ChatMemory Bean：

```java
@Configuration
public class JdbcChatMemoryConfiguration {

    @Bean
    ChatMemory chatMemory(JdbcChatMemoryRepository chatMemoryRepository) {
        return MessageWindowChatMemory.builder()
                .chatMemoryRepository(chatMemoryRepository)
                .build();
    }
}
```

在 ChatClient 中使用：

```java
@RestController
@RequestMapping("/jdbc/memory")
public class JdbcChatMemoryController implements InitializingBean {

    @Autowired
    private ChatModel chatModel;

    @Autowired
    private ChatMemory jdbcChatMemory;

    private ChatClient chatClient;

    @GetMapping("/chat")
    public Flux<String> chat(String message, String chatId,
                             HttpServletResponse response) {
        response.setCharacterEncoding("UTF-8");
        return chatClient.prompt()
                .user(message)
                .advisors(spec -> spec.param(ChatMemory.CONVERSATION_ID, chatId))
                .stream()
                .content();
    }

    @Override
    public void afterPropertiesSet() {
        this.chatClient = ChatClient.builder(chatModel)
                .defaultAdvisors(
                    MessageChatMemoryAdvisor.builder(jdbcChatMemory).build(),
                    new SimpleLoggerAdvisor()
                )
                .defaultOptions(DashScopeChatOptions.builder().withTopP(0.7).build())
                .build();
    }
}
```

### 验证

第一次请求，建立对话：
```
GET /jdbc/memory/chat?chatId=12321&message=i am hollis
```

第二次请求，测试记忆：
```
GET /jdbc/memory/chat?chatId=12321&message=who am i?
```

查看数据库，对话记录已经持久化：

![数据库存储结果](/images/posts/spring-ai-memory-persistence/database-result.png)

重启应用后，使用相同的 chatId 继续对话，AI 仍然能够正确回忆之前的上下文。

## 小结

通过 `ChatMemoryRepository` 接口，Spring AI 实现了记忆存储的可插拔设计。对于大多数项目，JDBC + MySQL 的方案已经足够：配置简单、运维成熟、性能够用。如果对可扩展性有更高要求，可以考虑 Cassandra 或 MongoDB 等方案。
