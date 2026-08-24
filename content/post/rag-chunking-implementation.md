---
title: "RAG 文档分片代码实现"
date: 2026-08-24T10:30:00+08:00
draft: false
description: "Spring AI、Spring AI Alibaba 和 LangChain4j 文档分片实现对比"
tags: ["Spring AI", "RAG", "Java", "LangChain4j"]
categories: ["AI"]
cover: "/hero/rag-chunking-code/spring-ai-textsplitter.png"
---

介绍三种文档分片实现方式：Spring AI 固定长度分片、Spring AI Alibaba 递归分片、LangChain4j 语义分片。

## Spring AI 文档分片

在 Spring AI 的 ETL Pipeline 模块中，`TextSplitter` 是所有文本拆分器的抽象基类。目前官方仅提供一个具体实现类：**TokenTextSplitter**，按 token 数量拆分文本。

![Spring AI TextSplitter](/hero/rag-chunking-code/spring-ai-textsplitter.png)

### TokenTextSplitter 参数

- **chunkSize**：每个文本块的目标大小（以 token 为单位），默认 800
- **minChunkSizeChars**：每个文本块的最小字符数，默认 350
- **minChunkLengthToEmbed**：只有长度超过此值的块才会被发送给向量模型
- **maxNumChunks**：单个文档允许拆分出的最大块数，默认 10000
- **keepSeparator**：是否在块中保留分隔符（如换行符），默认 true

### 代码实现

基于文档预处理代码，增加文档分片环节：

```java
@GetMapping("/read")
public List<Document> readDocument(@RequestParam("path") String path) {
    File file = new File(path);
    if (!file.exists() || !file.isFile()) {
        throw new IllegalArgumentException("文件不存在或不是有效文件: " + path);
    }
    try {
        // 1. 加载文档
        List<Document> documents = selector.read(file);
        // 2. 文本清洗
        documents = cleanDocuments(documents);
        // 3. 文档分片
        documents = split(documents);
        return documents;
    } catch (IOException e) {
        throw new RuntimeException("读取文件失败: " + e.getMessage(), e);
    }
}

public List<Document> split(List<Document> documents) {
    if (CollectionUtils.isEmpty(documents)) {
        return Collections.emptyList();
    }
    TokenTextSplitter splitter = new TokenTextSplitter(
            600,    // 每块最多 600 tokens
            300,    // 每块至少 400 字符再考虑断点
            5,      // 太短的不做嵌入
            8000,   // 最多拆分 8000 块
            true    // 保留句号、换行符
    );
    return splitter.apply(documents);
}
```

### 调用示例

对同一文档的预处理，按文档页码生成 12 个 Document，经过分片操作后变成 23 个：

![分片前](/hero/rag-chunking-code/token-splitter-example1.png)

![分片后](/hero/rag-chunking-code/token-splitter-example2.png)

### Spring AI 分片缺陷

Spring AI 的 `TokenTextSplitter` 存在两个主要缺陷：

1. **不支持 overlap 机制**：相邻文本块之间没有共享内容，可能导致语义被切断
2. **不支持按自然语言结构分割**：无法根据段落、句号等结构进行分割

GitHub 上已有相关 issue 提出优化建议：https://github.com/spring-projects/spring-ai/issues/2123

### 自定义重叠段落分片器

为解决 overlap 问题，自定义 `OverlapParagraphTextSplitter`：

```java
public class OverlapParagraphTextSplitter extends TextSplitter {
    private final int chunkSize;
    private final int overlap;

    public OverlapParagraphTextSplitter(int chunkSize, int overlap) {
        if (chunkSize <= 0) throw new IllegalArgumentException("chunkSize 必须大于 0");
        if (overlap < 0) throw new IllegalArgumentException("overlap 不能为负数");
        if (overlap >= chunkSize) throw new IllegalArgumentException("overlap 不能大于等于 chunkSize");
        this.chunkSize = chunkSize;
        this.overlap = overlap;
    }

    public List<String> splitText(String text) {
        if (StringUtils.isBlank(text)) return Collections.emptyList();
        String[] paragraphs = text.split("\\n+");
        List<String> allChunks = new ArrayList<>();
        StringBuilder currentChunk = new StringBuilder();

        for (String paragraph : paragraphs) {
            if (StringUtils.isBlank(paragraph)) continue;
            int start = 0;
            while (start < paragraph.length()) {
                int remainingSpace = chunkSize - currentChunk.length();
                int end = Math.min(start + remainingSpace, paragraph.length());
                if (currentChunk.length() > 0) currentChunk.append("\n");
                currentChunk.append(paragraph, start, end);
                if (currentChunk.length() >= chunkSize) {
                    allChunks.add(currentChunk.toString());
                    String overlapText = "";
                    if (overlap > 0) {
                        int overlapStart = Math.max(0, currentChunk.length() - overlap);
                        overlapText = currentChunk.substring(overlapStart);
                    }
                    currentChunk = new StringBuilder();
                    if (!overlapText.isEmpty()) currentChunk.append(overlapText);
                }
                start = end;
            }
        }
        if (currentChunk.length() > 0) allChunks.add(currentChunk.toString());
        return allChunks;
    }

    public List<Document> apply(List<Document> documents) {
        if (CollectionUtils.isEmpty(documents)) return Collections.emptyList();
        List<Document> result = new ArrayList<>();
        for (Document doc : documents) {
            List<String> chunks = splitText(doc.getText());
            for (String chunk : chunks) {
                result.add(new Document(chunk));
            }
        }
        return result;
    }
}
```

使用自定义分片器：

```java
OverlapParagraphTextSplitter splitter = new OverlapParagraphTextSplitter(400, 100);
List<Document> apply = splitter.apply(documents);
```

### 重叠效果

相邻文本块之间有 100 字符的重叠，保证文本块的延续性：

![重叠示例1](/hero/rag-chunking-code/overlap-example1.png)

![重叠示例2](/hero/rag-chunking-code/overlap-example2.png)

文本块从 23 个变成 48 个。

## Spring AI Alibaba 递归分片

使用 `RecursiveCharacterTextSplitter` 实现递归分片：

```java
RecursiveCharacterTextSplitter splitter = new RecursiveCharacterTextSplitter(100);
List<String> chunks = splitter.splitText("""
        《斗破苍穹》是中国网络作家天蚕土豆创作的玄幻小说...
        """);
chunks.forEach(System.out::println);
```

分段结果：

![递归分片结果1](/hero/rag-chunking-code/recursive-result1.png)

![递归分片结果2](/hero/rag-chunking-code/recursive-result2.png)

**注意**：使用递归分片时，文档不能预先清洗掉空格、换行等符号，否则会影响分段效果。

## LangChain4j 语义分片

使用 `DocumentBySentenceSplitter` 实现语义分片：

```java
DocumentBySentenceSplitter splitter = new DocumentBySentenceSplitter(100, 10);
for (String textSegment : splitter.split("""
        《斗破苍穹》是中国网络作家天蚕土豆创作的玄幻小说...
        """)) {
    System.out.println(textSegment);
}
```

**注意**：`DocumentBySentenceSplitter` 不支持中文，需要使用英文文本测试。

英文分段效果：

```
Harry Potter is a series of seven fantasy novels written by British author J. K.
Rowling.
The novels chronicle the lives of a young wizard, Harry Potter, and his friends...
```

### 语义分片对比

与其他分片器对比，语义分片效果更好：

![语义分片对比](/hero/rag-chunking-code/semantic-comparison.png)

## 总结

| 分片方式 | 优点 | 缺点 |
|---------|------|------|
| TokenTextSplitter | 简单易用 | 不支持 overlap，不支持按结构分割 |
| 自定义 OverlapParagraphTextSplitter | 支持 overlap，按段落分割 | 需要自行实现 |
| RecursiveCharacterTextSplitter | 递归分块，效果较好 | 不支持 overlap |
| DocumentBySentenceSplitter | 语义分块，效果最好 | 不支持中文 |

**建议**：
1. 将 LangChain4j 与 Spring AI 配合使用，以 Spring AI 为主体框架，LangChain4j 用于特定功能
2. 自定义 Spring AI 的 TextSplitter，实现定制化分片功能
3. 使用 Spring AI Alibaba 提供的分片工具