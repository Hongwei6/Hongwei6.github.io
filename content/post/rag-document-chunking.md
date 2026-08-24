---
title: "RAG 文档分片：常见分块方式"
date: 2026-08-24T10:00:00+08:00
draft: false
description: "RAG 文档分片实践，包括固定大小分块、递归分块、语义分块等常见方式"
tags: ["Spring AI", "RAG", "Java", "文档分片"]
categories: ["AI"]
cover: "/hero/rag-chunking/chunking-overview.png"
---

由于大模型上下文窗口的限制，无法一次性将整篇文档送入模型。在 RAG 中，需要将文档拆分成多个小文本块（chunk），检索时找到与用户问题最相似的文本块，使知识库大小在模型上下文可接受范围内。

![文档分片概览](/hero/rag-chunking/chunking-overview.png)

Chunking（也叫 Splitting）是将大型文档切分成更小的、语义连贯的文本块，以便检索系统精确找到相关信息。这是连接海量原始知识库和有限 LLM 上下文窗口的关键桥梁。

## 分块方式分类

典型的分块方式分为 5 个 level：

- Fixed Size Chunking（固定大小分块）
- Recursive Chunking（递归分块）
- Document Based chunking（基于文档分块）
- Semantic Chunking（语义分块）
- LLM-based Chunking（基于 LLM 分块）

## 固定大小分块

最简单的文本分段方法，将文本分割成指定数量的字符块，不考虑内容或结构。

关键参数：
- `chunk_size`：块中的字符数量
- `chunk_overlap`：相邻块中重叠的字符数

overlap 用于避免将单个上下文片段切割成多个片段，减少语义断裂。

以一段小说介绍为例，设置 `chunk_size=20`：

![固定大小分块示例](/hero/rag-chunking/fixed-size-chunking.png)

每个分段的字符数都是 20，长度一致。

设置 `overlap=5` 后：

![重叠示例](/hero/rag-chunking/overlap-example.png)

相邻分段中有 5 个字符重复，确保每个块的开头包含前一个块的结尾信息。

优点：实现简单。缺点：完全不考虑文档结构（段落、换行等）。

## 递归分块（常用）

指定一系列分隔符来分割文档，常见分隔符：
- `"\n\n"` - 双换行符（段落分隔）
- `"\n"` - 换行
- `" "` - 空格
- `""` - 空字符

![递归分块示例](/hero/rag-chunking/recursive-chunking.png)

与固定大小分块不同，递归分块会在分段处自然切割，如下一段从新位置开始。

### 递归原理

递归分块的核心在于递归调用。以 `RecursiveCharacterTextSplitter` 为例：

```java
public RecursiveCharacterTextSplitter(int chunkSize, String[] separators) {
    this.chunkSize = chunkSize;
    this.separators = Objects.requireNonNullElse(separators,
        new String[] { "\n\n", "\n", "。", "！", "？", "；", "，", " " });
}

private void splitText(String text, int separatorIndex, List<String> chunks) {
    if (text.isEmpty()) return;
    if (text.length() <= chunkSize) {
        chunks.add(text);
        return;
    }
    if (separatorIndex >= separators.length) {
        // 最终按块大小分割
        for (int i = 0; i < text.length(); i += chunkSize) {
            int end = Math.min(i + chunkSize, text.length());
            chunks.add(text.substring(i, end));
        }
        return;
    }
    String separator = separators[separatorIndex];
    String[] splits = text.split(separator);
    for (String split : splits) {
        if (split.length() > chunkSize) {
            splitText(split, separatorIndex + 1, chunks);  // 递归调用
        } else {
            chunks.add(split);
        }
    }
}
```

流程：从分隔符数组中逐个取出字符进行切分，判断长度是否符合，符合则返回，不符合则继续用下一个字符分割。

![递归流程图](/hero/rag-chunking/recursive-flow.png)

## 基于文档分块（常用）

递归分块对 Markdown 文档切割效果不佳。基于文档分块支持根据文档内在结构进行分割，对 Markdown、Python/JS 文档、带表格的文档、带图片的 PDF 更友好。

## 语义分块（常用）

以上分块方式只关注文档内容和结构，忽略语义。语义分块从文档中提取语义意义，评估块之间的语义关系，将语义相似的块保持在一起。

需要借助 NLP 技术，通过理解文本的语义边界（如主题转换、逻辑段落）进行智能分割。

## 智能分块

基于大模型的分段方案（LLM-based Chunking 或 Agentic Chunking），借助大模型做文档切割。

例如让 DeepSeek 帮助切割，能按语义主题自动分块：
- Chunk 1：基础信息与出版
- Chunk 2：故事背景与世界观设定
- Chunk 3：市场表现与荣誉奖项
- Chunk 4：衍生作品开发
- Chunk 5：IP 价值与法律案例

## 常用分块方式

### 百炼平台

![百炼平台](/hero/rag-chunking/bailian-platform.png)

支持 6 种切分方式：
- 智能切分（递归分块+语义分块）
- 按长度切分（固定大小分块）
- 按符号切分（递归分块）
- 按页切分（类似递归分块）
- 按标题切分（类似递归分块）
- 按正则切分（类似递归分块）

### Dify 平台

![Dify平台](/hero/rag-chunking/dify-platform.png)

支持两种方式：
- 通用分块（递归分块）
- 父子分块（效果较好，网上介绍较少）

### Coze 平台

![Coze平台](/hero/rag-chunking/coze-platform.png)

## 语义分块实现

Spring AI 和 LangChain4J 都提供了语义分块实现：
- `SentenceSplitter`：Spring AI 中的语义分块
- `DocumentBySentenceSplitter`：LangChain4J 中的语义分块

两者都使用 OpenNLP 的 `SentenceDetectorME`（最大熵模型）进行句子分割：

```java
SentenceDetectorME sentenceDetector = new SentenceDetectorME(sentenceModel);
String[] texts = sentenceDetector.sentDetect(text);
```

![Spring AI代码](/hero/rag-chunking/spring-ai-code.png)

![LangChain4J代码](/hero/rag-chunking/langchain4j-code.png)

通过加载预训练的句子模型（`opennlp-en-ud-ewt-sentence-1.2-2.5.0.bin`）进行文本分块。

## 总结

目前最常用的是**递归分块**，支持传入 3 种核心参数：
- 分段标识符
- 分段最大长度
- 分段重叠度/重叠长度

其次是**语义分块**，效果较好，很多平台上的智能分块依赖语义分块实现。此外，父子分段、按层级分段等方式在特定场景下也有良好表现。