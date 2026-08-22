---
title: "使用 LlamaIndex 构建一个简单的 RAG 系统"
date: 2026-08-22T13:48:00+08:00
draft: false
description: "使用 LlamaIndex + DashScope 构建 RAG 系统的完整流程，包含数据加载、向量化、索引构建、查询引擎"
tags: ["RAG", "AI", "LlamaIndex", "Python"]
categories: ["AI"]
cover: "/hero/llamaindex-rag/01-run-result.png"
---

LlamaIndex 是一个专为大语言模型设计的数据连接与检索框架，核心目标是让 LLM 高效、准确地访问私有或结构化/非结构化的外部数据。

## 主要功能

- **数据索引构建**：支持 PDF、Word、网页、数据库、API 等多种格式转换为 LLM 可理解的索引结构
- **高效检索**：提供向量相似性搜索、混合检索、子问题分解等多种查询引擎
- **与 LLM 无缝集成**：自动将检索到的上下文注入 prompt，供 LLM 生成基于事实的回答
- **模块化与可扩展**：支持自定义文档加载器、文本分割器、嵌入模型、LLM 后端

## 快速搭建 RAG

### 初始化环境

```bash
uv init llamaindex_test
cd llamaindex_test
uv venv
source .venv/bin/activate
```

### 添加依赖

```bash
uv add llama-index llama-index-llms-dashscope llama-index-embeddings-dashscope
```

### 准备测试数据

```bash
mkdir data
cd data
echo "《斗破苍穹》是中国网络作家天蚕土豆创作的玄幻小说..." > test.txt
```

### 编写代码

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.embeddings.dashscope import DashScopeEmbedding
from llama_index.llms.dashscope import DashScope
import os

DASHSCOPE_API_KEY = "sk-xx"  # 替换为你的 API Key

# 1. 加载文档
documents = SimpleDirectoryReader("data").load_data()

# 2. 设置嵌入模型
embed_model = DashScopeEmbedding(
    model_name="text-embedding-v2",
    api_key=DASHSCOPE_API_KEY
)

# 3. 设置 LLM
llm = DashScope(
    model_name="qwen-max",
    temperature=0.1,
    api_key=DASHSCOPE_API_KEY
)

# 4. 构建索引
index = VectorStoreIndex.from_documents(documents, embed_model=embed_model)

# 5. 创建查询引擎
query_engine = index.as_query_engine(llm=llm)

# 6. 提问
response = query_engine.query("请总结这份文档的主要内容。")
print(response)
```

### 运行

```bash
uv run main.py
```

![运行结果](/hero/llamaindex-rag/01-run-result.png)

## 核心组件说明

### SimpleDirectoryReader

数据加载器，自动读取指定目录下的多种格式文件（`.txt`、`.pdf`、`.docx` 等），转换为 `Document` 对象列表。

### DashScopeEmbedding

调用阿里云 DashScope 的 embedding API，使用 `text-embedding-v2` 模型生成高质量向量。构建索引时，LlamaIndex 会自动对每个文档块调用此模型。

### VectorStoreIndex.from_documents()

核心索引构建方法，默认使用**内存中的简单向量存储（InMemoryVectorStore）**，自动对文档进行分块（默认 chunk_size=1024, chunk_overlap=20），然后嵌入并建立索引。

### as_query_engine(llm=llm)

将索引转换为可查询引擎，默认使用 **Retrieval + Synthesis** 流程：
1. 用嵌入模型对 query 嵌入
2. 在向量库中检索 top-k 最相似的文档块
3. 将块作为上下文拼接到 prompt
4. 调用 LLM 生成最终回答

## 生产环境优化方向

以上代码实现了简单 RAG，但生产环境还需要优化：

- 文档分块（Chunking）策略优化
- 使用持久化向量数据库
- 查询优化
- 文档检索优化
- 对话记忆
- 重排序
- 混合检索
- 多模态
- RAG 效果测评
