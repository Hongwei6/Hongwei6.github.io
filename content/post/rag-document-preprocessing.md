---
title: "RAG 文档预处理：读取与清洗"
date: 2026-08-24T09:53:42+08:00
draft: false
description: "Spring AI RAG 文档预处理实践，包括多格式文档读取策略和数据清洗方法"
tags: ["Spring AI", "RAG", "Java", "文档处理"]
categories: ["AI"]
cover: "/hero/rag-doc-preprocess/doc-read-example.png"
---

文档预处理是将原始文档加载、解析并转换为标准化格式的过程，是构建高质量知识库的基础。

主要目标：让后续的分片和向量化在干净的数据上进行，从源头保证知识库索引质量。

基于 Spring AI 1.1.0 版本。

## 文档读取

文档读取是将不同格式（TXT、PDF、Word、HTML、Markdown 等）统一处理为标准化的 `Document` 对象。

Spring AI 提供了 `DocumentReader` 接口，主要作用：
- **统一读取接口**：从不同来源加载原始文本
- **结构化输出**：封装为 `Document` 对象，包含 `content`（文本内容）和 `metadata`（元数据）
- **支持扩展**：可自定义实现特定格式的解析器

### 策略接口定义

定义策略接口，动态路由到具体的文档读取器：

```java
public interface DocumentReaderStrategy {
    boolean supports(File file);
    List<Document> read(File file) throws IOException;
}
```

### 文本读取器

适合简单文本文件（txt、tex、text），spring-ai-commons 包自带解析器：

```java
@Component
public class TextReaderStrategy implements DocumentReaderStrategy {
    @Override
    public boolean supports(File file) {
        String name = file.getName().toLowerCase();
        return name.endsWith(".txt") || name.endsWith(".tex") || name.endsWith(".text");
    }

    @Override
    public List<Document> read(File file) throws IOException {
        Resource resource = new FileSystemResource(file);
        return new TextReader(resource).get();
    }
}
```

### PDF 读取器

依赖：
```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-pdf-document-reader</artifactId>
  <version>1.1.0</version>
</dependency>
```

提供两种读取器：
- `PagePdfDocumentReader`：按页切分
- `ParagraphPdfDocumentReader`：按语义段落切分

建议优先使用 `ParagraphPdfDocumentReader`，能更好保留信息完整性，但依赖 PDF 本身质量。

```java
@Component
public class PdfReaderStrategy implements DocumentReaderStrategy {
    @Override
    public boolean supports(File file) {
        return file.getName().toLowerCase().endsWith(".pdf");
    }

    @Override
    public List<Document> read(File file) throws IOException {
        PdfDocumentReaderConfig config = PdfDocumentReaderConfig.builder()
                .withPageTopMargin(50)
                .withPageBottomMargin(50)
                .withPagesPerDocument(1)
                .withPageExtractedTextFormatter(new ExtractedTextFormatter.Builder()
                        .withNumberOfTopTextLinesToDelete(0)
                        .build())
                .build();
        Resource resource = new FileSystemResource(file);
        return new PagePdfDocumentReader(resource, config).get();
    }
}
```

### HTML 读取器

基于 Jsoup，支持 CSS 选择器提取指定内容：

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-jsoup-document-reader</artifactId>
  <version>1.1.0</version>
</dependency>
```

```java
@Component
public class HtmlReaderStrategy implements DocumentReaderStrategy {
    @Override
    public boolean supports(File file) {
        String name = file.getName().toLowerCase();
        return name.endsWith(".html") || name.endsWith(".htm");
    }

    @Override
    public List<Document> read(File file) throws IOException {
        JsoupDocumentReaderConfig config = JsoupDocumentReaderConfig.builder()
                .selector("p")
                .charset("UTF-8")
                .includeLinkUrls(true)
                .metadataTags(List.of("author", "date"))
                .additionalMetadata("filename", file.getName())
                .build();
        Resource resource = new FileSystemResource(file);
        return new JsoupDocumentReader(resource, config).get();
    }
}
```

### Markdown 读取器

支持按标题或水平分割线分片，可选择性包含代码块或引用内容：

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-markdown-document-reader</artifactId>
  <version>1.1.0</version>
</dependency>
```

```java
@Component
public class MarkdownReaderStrategy implements DocumentReaderStrategy {
    @Override
    public boolean supports(File file) {
        return file.getName().toLowerCase().endsWith(".md");
    }

    @Override
    public List<Document> read(File file) throws IOException {
        MarkdownDocumentReaderConfig config = MarkdownDocumentReaderConfig.builder()
                .withHorizontalRuleCreateDocument(true)
                .withIncludeCodeBlock(false)
                .withIncludeBlockquote(false)
                .withAdditionalMetadata("filename", file.getName())
                .build();
        Resource resource = new FileSystemResource(file);
        return new MarkdownDocumentReader(resource, config).get();
    }
}
```

### JSON 读取器

可处理 JSON 数组或对象，不支持嵌套字段，适合简单 JSON 文件：

```java
@Component
public class JsonReaderStrategy implements DocumentReaderStrategy {
    public boolean supports(File file) {
        return file.getName().toLowerCase().endsWith(".json");
    }

    @Override
    public List<Document> read(File file) throws IOException {
        Resource resource = new FileSystemResource(file);
        JsonReader jsonReader = new JsonReader(resource, "description", "content");
        return jsonReader.get();
    }
}
```

### Tika 通用读取器

可处理大部分常见文档格式（word、pdf、ppt 等），自动识别文档类型：

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-tika-document-reader</artifactId>
  <version>1.1.0</version>
</dependency>
```

```java
@Component
public class TikaReaderStrategy implements DocumentReaderStrategy {
    @Override
    public boolean supports(File file) {
        String name = file.getName().toLowerCase();
        return name.endsWith(".doc") || name.endsWith(".docx");
    }

    @Override
    public List<Document> read(File file) throws IOException {
        Resource resource = new FileSystemResource(file);
        return new TikaDocumentReader(resource).get();
    }
}
```

### 策略选择器

```java
@Service
public class DocumentReaderStrategySelector {
    private final List<DocumentReaderStrategy> strategies;

    public DocumentReaderStrategySelector(List<DocumentReaderStrategy> strategies) {
        this.strategies = strategies;
    }

    public List<Document> read(File file) throws IOException {
        for (DocumentReaderStrategy strategy : strategies) {
            if (strategy.supports(file)) {
                return strategy.read(file);
            }
        }
        throw new IllegalArgumentException("不支持的文件类型: " + file.getName());
    }
}
```

### 测试接口

```java
@RestController
@RequestMapping("/rag")
public class RagController {
    private final DocumentReaderStrategySelector selector;

    @Autowired
    public RagController(DocumentReaderStrategySelector selector) {
        this.selector = selector;
    }

    @GetMapping("/read")
    public List<Document> readDocument(@RequestParam("path") String path) {
        File file = new File(path);
        if (!file.exists() || !file.isFile()) {
            throw new IllegalArgumentException("文件不存在或不是有效文件: " + path);
        }
        try {
            return selector.read(file);
        } catch (IOException e) {
            throw new RuntimeException("读取文件失败: " + e.getMessage(), e);
        }
    }
}
```

### 调用示例

以 PDF 文档为例，根据选择器的路由，PDF 被交给 PDF 读取器处理。设置了 `withPagesPerDocument` 参数，生成结果为每页一个 Document：

![文档读取示例](/hero/rag-doc-preprocess/doc-read-example.png)

## 数据清洗

读取结果往往包含大量无效干扰字符，需要进行数据清洗：去除多余空格、换行符、无意义的特殊符号和重复内容等。

![清洗前数据](/hero/rag-doc-preprocess/data-clean-before.png)

### Document 清洗实现

```java
public List<Document> cleanDocuments(List<Document> documents) {
    if (CollectionUtils.isEmpty(documents)) {
        return documents;
    }
    return documents.stream()
        .map(doc -> {
            if (doc == null || doc.getText() == null) {
                return doc;
            }
            String text = doc.getText();
            
            // 1. 去掉多余空白字符
            text = text.replaceAll("\\s+", " ").trim();
            
            // 2. 去掉无意义的乱码或特殊符号
            text = text.replaceAll("[^\\p{L}\\p{N}\\p{P}\\p{Z}\\n]", "");
            
            // 3. 按换行拆分段落，去除重复段落
            String[] paragraphs = text.split("\\n+");
            Set<String> seen = new LinkedHashSet<>();
            for (String para : paragraphs) {
                String trimmed = para.trim();
                if (!trimmed.isEmpty()) {
                    seen.add(trimmed);
                }
            }
            text = String.join("\n", seen);
            return new Document(text);
        })
        .collect(Collectors.toList());
}
```

### 清洗效果

清洗前后对比：

![清洗后效果1](/hero/rag-doc-preprocess/data-clean-after1.png)

![清洗后效果2](/hero/rag-doc-preprocess/data-clean-after2.png)

通过文本清洗，原始数据中的噪声、冗余和格式不一致问题得到有效处理，为后续的文档分片、向量化以及检索生成等环节提供更高质量的基础支撑。