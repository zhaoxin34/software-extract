# memory

> Sources: memory project, 2024
> Raw: [memory extracted-knowledge](../../raw/memory/extracted-knowledge.md); [memory primary-report](../../raw/memory/primary-report.md)

## Overview

memory 是一个**个人知识库系统**，支持语义搜索和基于 LLM 的问答。核心特点是**多级分块 Fallback**（正则 → tree-sitter → 固定大小）、**Provider 抽象**（Embedding/LLM）、**存储分离**（VectorStore 和 MetadataStore 分开）、以及 **Pipeline 编排**（摄取管道和查询管道分离）。

## 整体架构

```
memory/
├── core/               # 核心算法
│   ├── chunking.py              # 通用分块策略
│   ├── markdown_chunking.py     # Markdown 语义分块（正则实现）
│   └── tree_sitter_chunking.py  # Markdown 语法树分块（tree-sitter）
├── pipelines/          # 业务流程
│   ├── ingestion.py             # 文档摄取管道
│   └── query.py                 # 查询管道
├── providers/           # AI 提供者抽象
│   ├── base.py                 # 抽象基类定义
│   ├── openai_embd.py          # OpenAI Embedding
│   └── local.py                # 本地模型
├── storage/             # 存储抽象
│   ├── base.py                 # 存储接口定义
│   ├── sqlite.py              # SQLite 元数据存储
│   └── chroma.py               # Chroma 向量存储
└── entities/            # 数据模型
```

## 模块：分块算法

### 1. 通用分块 (chunking.py)

滑动窗口分块：固定大小的 chunk_size 和 chunk_overlap，安全限制防止无限循环（max_iterations = text_length * 2 + 10）。

### 2. Markdown 语义分块 (markdown_chunking.py)

使用正则表达式解析 Markdown 结构，识别 heading、paragraph、list、code、table、blockquote。

**关键特性**：
- 标题层级上下文追踪（heading_stack）
- smart_merge_chunks 智能合并保持语义边界
- 重叠时保留前一个 chunk 的最后部分
- detect_chunk_type 推断 chunk 类型

**Fallback**：分块失败时回退到 tree-sitter_chunking，tree-sitter 也不可用时回退到通用固定大小分块。

### 3. Markdown 语法树分块 (tree_sitter_chunking.py)

使用 tree-sitter 解析 Markdown 语法树，SemanticNode 表示语法节点。

**关键特性**：
- 5 秒解析超时保护
- 表格完整保留（_reconstruct_table）
- ERROR 和 WHITESPACE 节点过滤

## 模块：Provider 抽象

### EmbeddingProvider 接口

```python
class EmbeddingProvider(ABC):
    def embed_text(self, text: str) -> list[float]: ...
    def embed_batch(self, texts: list[str]) -> list[list[float]]: ...
    def get_dimension(self) -> int: ...
    def get_max_tokens(self) -> int: ...
```

实现类：OpenAIEmbeddingProvider、LocalEmbeddingProvider（sentence-transformers）

### LLMProvider 接口

```python
class LLMProvider(ABC):
    def generate(self, prompt, system_prompt, max_tokens, temperature) -> str: ...
    def count_tokens(self, text: str) -> int: ...
```

## 模块：存储抽象

### VectorStore 抽象

```python
class VectorStore(ABC):
    def initialize(self) -> None: ...
    def add_embedding(self, embedding, chunk): ...
    def add_embeddings_batch(self, embeddings, chunks): ...
    def search(query_vector, top_k, repository_id, filters) -> list[SearchResult]: ...
    def delete_by_document_id(self, document_id): ...
```

可选方法 `hybrid_search`：向量 + BM25，默认抛出 NotImplementedError。

实现类：ChromaVectorStore、QdrantVectorStore、FAISSVectorStore

### MetadataStore 抽象

管理 Document、Chunk、Repository 元数据。实现类：SQLiteMetadataStore

## 模块：摄取管道 (IngestionPipeline)

**流程步骤**：
1. 查找现有文档（按 source_path 和 repository_id）
2. 内容变更检测（MD5 比较，content_unchanged 时跳过）
3. 删除旧数据（cascade 删除）
4. 存储文档元数据
5. 创建块
6. 批量嵌入生成（batch_size 控制）
7. 存储嵌入

**关键设计**：
- **幂等性**：content_unchanged 时跳过处理
- **回滚机制**：失败时恢复到原始状态
- **批量处理**：batch_size 控制嵌入生成批次

## 模块：查询管道 (QueryPipeline)

**search 流程**：
1. 生成查询嵌入
2. 执行向量搜索或混合搜索
3. 丰富结果（加载文档元数据）

**answer 流程**：
1. 调用 search 获取相关块
2. 构建上下文（限制总长度）
3. 调用 LLM 生成答案

```python
# Answer Prompt 模板
Context:
{context}

Question: {query}

Answer:
```

## 关键设计决策

### 1. Provider 抽象

定义 EmbeddingProvider 和 LLMProvider 抽象接口，支持多种 AI 提供者，便于测试和切换。

### 2. 存储分离

VectorStore 和 MetadataStore 分离，向量存储和元数据存储有不同需求，分离便于扩展。

### 3. 多级分块 Fallback

Markdown 分块：正则 → tree-sitter → 固定大小，逐级回退保证分块成功。

### 4. 内容变更检测

MD5 哈希检测内容变更，高效检测，避免不必要重新处理。

## See Also

- [LightRAG](../rag/lightrag.md) — 类似的 RAG 框架存储抽象和多存储后端设计
