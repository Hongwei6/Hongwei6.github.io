---
title: "Spring AI 实战 —— RAG 查询构造（Text2SQL）"
date: 2026-08-31T10:30:00+08:00
draft: false
description: "RAG 查询构造：通过 Text2SQL 将自然语言转换为 SQL 查询语句，用于关系型/图数据库的数据源路由"
tags: ["Spring AI", "RAG", "查询构造", "Text2SQL"]
categories: ["AI"]
---

查询路由中，向量数据库可以直接做相似度检索，但如果路由目标是**图数据库或关系型数据库**，就需要先把自然语言转成 Cypher 或 SQL。因此在查询这一步，需要提前构造出对应的查询内容，这就是 **Text2SQL** 能力。

## Text2SQL

**Text2SQL**（自然语言到 SQL）是一种将用户用自然语言表达的查询意图，自动转换为结构化 SQL 查询语句的技术。它融合了自然语言处理与数据库查询技术，降低非技术人员使用数据库的门槛。

下面是用 LLM 生成 SQL 的实现方式。

### 提示词模板

```text
# 角色
你是一个SQL专家。请根据以下表结构信息将用户问题转换为SQL查询语句。特别注意，你只能查询，不能做修改、删除等操作。

# 表结构信息
{tables}

# 用户问题
{user_query}

# 要求
1. 只返回SQL语句，不需要包含任何解释和说明
2. 确保SQL语法正确
3. 使用上下文中提供的表名和字段名
4. 如果根据所提供的表无法做查询，请直接返回空字符串""

# 其他说明
今天是:{today}
```

其中 `{tables}`、`{user_query}`、`{today}` 需要实时替换：

- **表结构 `{tables}`**：提供详细的建表语句，LLM 才知道 SQL 该怎么写
- **时间 `{today}`**：涉及"最近几天、今天"等查询时需要知道当前时间

### 代码实现

```java
@Service
@Slf4j
public class SqlQueryService {

    private ChatClient chatClient;
    private ChatModel chatModel;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public SqlQueryService(OpenAiChatModel chatModel) {
        this.chatModel = chatModel;
    }

    @PostConstruct
    public void init() {
        chatClient = ChatClient.builder(chatModel).build();
    }

    private static final String TEXT_2_SQL_PROMPT = """
            # 角色
            你是一个SQL专家。请根据以下表结构信息将用户问题转换为SQL查询语句。特别注意，你只能查询，不能做修改、删除等操作。

            # 表结构信息

            {tables}

            # 用户问题

            {user_query}

            # 要求
            1. 只返回SQL语句，不需要包含任何解释和说明
            2. 确保SQL语法正确
            3. 使用上下文中提供的表名和字段名
            4. 如果根据所提供的表无法做查询，请直接返回空字符串""

            # 其他说明
            今天是:{today}
            """;

    private static final String TABLES = """
            CREATE TABLE "app"."staff_info"
            (
             "id" bigint NOT NULL ,
             "emp_id" varchar(64) NOT NULL ,
             "name" varchar(128) NOT NULL ,
             "job" varchar(128) NOT NULL ,
             "entry_time" date NOT NULL ,
             "gender" varchar(32) NOT NULL DEFAULT 'F'::character varying ,
             "birthday" date NOT NULL ,
             "educational_background" varchar(64) NOT NULL ,
             "director_id" varchar(64) ,
             "dept_id" varchar(64) NOT NULL ,
             "duty" varchar(512) ,
             "motto" varchar(1024) ,
             "pic_url" varchar(512) ,
            CONSTRAINT "pk_app_staff_info" PRIMARY KEY ("id")
            )
            WITH (
                FILLFACTOR = 100,
                OIDS = FALSE
            );
            ALTER TABLE "app"."staff_info" OWNER TO llmentor;
            COMMENT ON COLUMN "app"."staff_info"."emp_id" IS '工号';
            COMMENT ON COLUMN "app"."staff_info"."name" IS '姓名';
            COMMENT ON COLUMN "app"."staff_info"."job" IS '岗位';
            COMMENT ON COLUMN "app"."staff_info"."entry_time" IS '入职时间';
            COMMENT ON COLUMN "app"."staff_info"."gender" IS '性别：M表示女性，F表示男性';
            COMMENT ON COLUMN "app"."staff_info"."birthday" IS '出生年月日';
            COMMENT ON COLUMN "app"."staff_info"."educational_background" IS '学历：junior、undergraduate、master、doctor';
            COMMENT ON COLUMN "app"."staff_info"."director_id" IS '主管id';
            COMMENT ON COLUMN "app"."staff_info"."dept_id" IS '部门id';
            COMMENT ON COLUMN "app"."staff_info"."duty" IS '工作职责';
            COMMENT ON COLUMN "app"."staff_info"."motto" IS '个性签名';
            COMMENT ON COLUMN "app"."staff_info"."pic_url" IS '头像地址';

            CREATE TABLE "app"."dept_info"
            (
             "id" bigint NOT NULL ,
             "name" varchar(128) NOT NULL ,
             "owner_id" varchar(64) NOT NULL ,
             "parent_dept_id" bigint NOT NULL ,
             "desc" varchar(1024)
            )
            WITH (
                FILLFACTOR = 100,
                OIDS = FALSE
            );
            ALTER TABLE "app"."dept_info" OWNER TO llmentor;
            COMMENT ON COLUMN "app"."dept_info"."id" IS '部门id';
            COMMENT ON COLUMN "app"."dept_info"."name" IS '部门名称';
            COMMENT ON COLUMN "app"."dept_info"."owner_id" IS '主管工号';
            COMMENT ON COLUMN "app"."dept_info"."parent_dept_id" IS '父级部门id';
            COMMENT ON COLUMN "app"."dept_info"."desc" IS '部门描述';
            COMMENT ON TABLE "app"."dept_info" IS '部门信息';
            """;

    public String text2sql(String query) {
        PromptTemplate promptTemplate = new PromptTemplate(TEXT_2_SQL_PROMPT);
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        Prompt prompt = promptTemplate.create(Map.of(
                "user_query", query,
                "tables", TABLES,
                "today", sdf.format(new Date())));
        String sql = chatClient.prompt(prompt).call().content();
        System.out.println("sql:" + sql);
        return sql;
    }

    public Object sqlQuery(String query, Class clazz) {
        if (StringUtils.isBlank(query)) {
            return null;
        }
        String sql = text2sql(query);
        if (StringUtils.isBlank(sql)) {
            return null;
        }
        return jdbcTemplate.queryForObject(sql, clazz);
    }
}
```

## 说明

- 提示词中明确限制"只能查询，不能做修改、删除"，避免 LLM 生成有破坏性的 SQL
- 表结构信息越详细（含字段注释），LLM 生成的 SQL 越准确
- 若提供的表无法回答该问题，要求直接返回空字符串，由调用方兜底处理

Text2SQL 是查询路由落到结构化数据源时的关键一环，让关系型/图数据库也能通过自然语言查询。
