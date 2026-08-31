---
title: "Spring AI 实战 —— RAG 问题澄清"
date: 2026-08-31T11:00:00+08:00
draft: false
description: "RAG 问题澄清：当用户问题模糊或信息不足时，通过多轮对话主动收集信息，再生成规划或答案"
tags: ["RAG", "问题澄清", "LangChain4j", "对话记忆"]
categories: ["AI"]
---

问题重写、查询优化能解决一部分"用户不会提问"的问题，但并非万能。当问题存在**模糊、不完整、歧义**，或需要额外上下文才能准确回答时，还有一种简单直接的手段：**主动与用户交互，收集更多信息或确认真实意图**，这就是**问题澄清**。

以旅行规划助手为例：做行程规划需要知道目的地、时间、人数、预算、偏好等信息，而用户一开始往往提供不全，此时就需要通过问题澄清来收集信息。

## 定义 AI 服务

`TravelPlanningAiService` 通过系统提示词告知 LLM：先做信息收集，再做行程规划。提示词包含三方面内容——对话原则、信息收集、行程规划：

```java
import dev.langchain4j.service.MemoryId;
import dev.langchain4j.service.SystemMessage;
import dev.langchain4j.service.UserMessage;

public interface TravelPlanningAiService {
    @SystemMessage("""
            你是一个专业的旅行顾问，擅长制定个性化的旅行方案。

            对话原则：
            1. 保持热情、友好的语调，像朋友一样自然对话
            2. 基于已有信息给出建议和想法
            3. 如需更多信息，自然地询问细节（避免"我需要更多信息"这样的表达）
            4. 当信息足够时，生成详细的旅行规划

            根据用户输入的不同性质，你需要：

            【信息收集阶段】
            - 说"听起来很棒！具体想..."来了解细节
            - 通过建议来引出问题："这个地方我很推荐！大概预算多少合适？"
            - 每次最多问1-2个相关问题

            【规划生成阶段】
            - 当掌握了目的地、时间、预算、人员等核心信息时
            - 生成包含具体日程、住宿、交通、活动的详细规划
            - 提供实用的旅行建议和注意事项

            始终提供有价值的内容，避免让用户感觉在被"审问"。
            """)
    String chatWithTraveler(@MemoryId String memoryId, @UserMessage String userInput);
}
```

这里使用 `@MemoryId` 支持**对话记忆**：用户的核心需求往往是多轮对话汇总后才完整的，这些都是 LLM 做规划的重要依据。

## 定义 Controller

Controller 提供三个方法：

```java
@RestController
@RequestMapping("/travelPlan")
public class SmartTravelPlanningController {
    @Autowired
    private TravelPlanningAiService travelAiService;

    @RequestMapping("/start")
    public Map<String, String> startTravelPlanning(HttpServletResponse response) {
        response.setCharacterEncoding("UTF-8");
        String memoryId = UUID.randomUUID().toString();
        String welcomeMessage = """
                🌟 欢迎使用智能行程规划助手！

                我可以帮助您制定个性化的旅行计划。为了给您提供最佳的建议，我需要了解一些基本信息：

                • 您想去哪里旅行？
                • 计划什么时候出发？
                • 大概的预算范围？
                • 和谁一起旅行？
                • 您的兴趣爱好？

                请告诉我您的旅行想法，我会根据您提供的信息逐步完善行程计划！
                """;
        return Map.of(
                "sessionId", memoryId,
                "message", welcomeMessage);
    }

    @RequestMapping("/chat")
    public String chatWithPlanner(HttpServletResponse response,
                                  @RequestParam String memoryId,
                                  @RequestParam String message) {
        response.setCharacterEncoding("UTF-8");
        return travelAiService.chatWithTraveler(memoryId, message);
    }

    @RequestMapping("/force-plan")
    public String forcePlan(HttpServletResponse response, @RequestParam String memoryId) {
        response.setCharacterEncoding("UTF-8");
        return travelAiService.chatWithTraveler(memoryId,
                "请基于我们到目前为止的所有对话，生成完整详细的行程规划方案");
    }
}
```

- **start**：开启对话，生成 `memoryId`（会话 ID），返回欢迎语并列出需要收集的信息
- **chat**：对话，LLM 可能继续澄清需求，也可能直接给出行程建议
- **force-plan**：跳过澄清，基于已有对话强制生成完整行程规划

## 说明

问题澄清的核心在于**把信息收集的工作交给 LLM，通过多轮对话补齐缺失上下文**。要点：

- 系统提示词中明确"信息收集 → 规划生成"两个阶段
- 收集信息时保持自然，每次只问 1-2 个问题，避免让用户感觉在被审问
- 依赖对话记忆，多轮累积的信息才是完整上下文

问题澄清与问题改写、查询构造等手段互补，共同解决 RAG 中"输入质量"这一核心难题。
