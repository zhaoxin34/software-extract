# 萃取的精华

## memory 系统设计

**description**: 个人知识库系统，支持语义搜索和基于 LLM 的问答

**purpose**: 为个人或团队提供本地运行的知识库解决方案，支持文档摄取、语义搜索和 AI 问答

---

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
│   ├── openai_llm.py           # OpenAI LLM
│   └── local.py                # 本地模型
├── storage/             # 存储抽象
│   ├── base.py                 # 存储接口定义
│   ├── sqlite.py              # SQLite 元数据存储
│   └── chroma.py               # Chroma 向量存储
├── entities/            # 数据模型
├── config/              # 配置管理
├── interfaces/          # CLI 接口
└── service/             # 服务层
```

---

## 模块：分块算法

**description**: 多种分块策略，支持通用文本和 Markdown 语义分块

**purpose**: 将文档分割成适合嵌入的块，同时保持语义完整性

### 1. 通用分块 (chunking.py)

**算法名称**: 滑动窗口分块

**核心逻辑**:
- 固定大小的 chunk_size 和 chunk_overlap
- 迭代提取文本片段
- 安全限制防止无限循环（max_iterations = text_length * 2 + 10）

**关键特性**:
- min_chunk_size 过滤过小块
- 重叠窗口保持上下文连续性
- 日志记录分块进度

**fallback 策略**:
- 无内置 fallback，返回空

### 2. Markdown 语义分块 (markdown_chunking.py)

**算法名称**: Markdown 正则语义分块

**核心逻辑**:
- 使用正则表达式解析 Markdown 结构
- 识别 heading、paragraph、list、code、table、blockquote
- MarkdownChunk 类表示语义单元
- smart_merge_chunks 智能合并保持语义边界

**关键特性**:
- 标题层级上下文追踪（heading_stack）
- heading context 前缀保持语义完整性
- 重叠（overlap）时保留前一个 chunk 的最后部分
- detect_chunk_type 推断 chunk 类型（content/heading/code/list/blockquote）

**fallback 策略**:
- 分块失败时回退到 tree-sitter_chunking
- tree-sitter 也不可用时回退到通用固定大小分块

### 3. Markdown 语法树分块 (tree_sitter_chunking.py)

**算法名称**: Tree-sitter 语法树分块

**核心逻辑**:
- 使用 tree-sitter 解析 Markdown 语法树
- SemanticNode 表示语法节点
- extract_semantic_nodes 提取语义单元
- merge_to_target_size 合并到目标大小

**关键特性**:
- 5秒解析超时保护
- 支持 fenced_code_block 语言提取
- 表格完整保留（_reconstruct_table）
- 标题层级追踪（level 1-6）
- ERROR 和 WHITESPACE 节点过滤

**fallback 策略**:
- tree_sitter 不可用返回 None
- 解析超时（5秒）返回 None
- 回退到正则 Markdown 分块

---

## 模块：Provider 抽象

**description**: Embedding 和 LLM 提供者抽象接口

**purpose**: 支持多种 AI 模型提供者，便于切换和扩展

### 1. EmbeddingProvider 抽象

**接口方法**:
- `embed_text(text: str) -> list[float]`: 单文本嵌入
- `embed_batch(texts: list[str]) -> list[list[float]]`: 批量嵌入
- `get_dimension() -> int`: 嵌入维度
- `get_max_tokens() -> int`: 最大 token 数
- `close() -> None`: 资源清理

**实现类**:
- OpenAIEmbeddingProvider: OpenAI text-embedding-3 等
- LocalEmbeddingProvider: sentence-transformers 本地模型

### 2. LLMProvider 抽象

**接口方法**:
- `generate(prompt, system_prompt, max_tokens, temperature) -> str`: 文本生成
- `count_tokens(text: str) -> int`: Token 计数

**实现类**:
- OpenAILLMProvider: OpenAI GPT 系列

### 3. ProviderConfig 配置模型

```python
class ProviderConfig(BaseModel):
    provider_type: str
    model_name: str
    api_key: str | None = None
    extra_params: dict[str, Any] = {}
```

---

## 模块：存储抽象

**description**: VectorStore 和 MetadataStore 分离设计

**purpose**: 解耦向量存储和元数据存储，支持多种后端

### 1. VectorStore 抽象

**接口方法**:
- `initialize() -> None`: 初始化
- `add_embedding(embedding, chunk)`: 添加单个嵌入
- `add_embeddings_batch(embeddings, chunks)`: 批量添加
- `search(query_vector, top_k, repository_id, filters) -> list[SearchResult]`: 向量搜索
- `delete_by_document_id(document_id)`: 删除文档嵌入
- `delete_by_chunk_id(chunk_id)`: 删除 chunk 嵌入
- `delete_by_repository(repository_id)`: 删除仓库所有嵌入
- `count() -> int`: 计数
- `close() -> None`: 关闭连接

**可选方法**:
- `hybrid_search(query_text, query_vector, top_k, repository_id, filters)`: 混合搜索（向量 + BM25）
  - 默认抛出 NotImplementedError
  - ChromaVectorStore 提供实现

**实现类**:
- ChromaVectorStore: ChromaDB 向量存储
- QdrantVectorStore: Qdrant 向量存储
- FAISSVectorStore: FAISS 向量存储

### 2. MetadataStore 抽象

**接口方法**:
- `initialize() -> None`: 初始化
- `add_document(document)`: 添加文档
- `get_document(document_id) -> Document`: 获取文档
- `add_chunk(chunk)`: 添加块
- `get_chunk(chunk_id) -> Chunk`: 获取块
- `get_chunks_by_document(document_id) -> list[Chunk]`: 获取文档所有块
- `delete_document(document_id)`: 删除文档
- `list_documents(limit, offset, repository_id)`: 列出文档
- `add_repository(repository)`: 添加仓库
- `get_repository(repository_id) -> Repository`: 获取仓库
- `get_repository_by_name(name) -> Repository`: 按名称获取
- `list_repositories() -> list[Repository]`: 列出仓库
- `delete_repository(repository_id)`: 删除仓库
- `delete_by_repository(repository_id)`: 删除仓库所有数据
- `close() -> None`: 关闭连接

**实现类**:
- SQLiteMetadataStore: SQLite 元数据存储

---

## 模块：摄取管道 (IngestionPipeline)

**description**: 文档摄取完整流程编排

**purpose**: 协调文档加载、分块、嵌入生成和存储

### 流程步骤

1. **查找现有文档** - 按 source_path 和 repository_id 查找
2. **内容变更检测** - MD5 比较，content_unchanged 时跳过
3. **删除旧数据** - cascade 删除（嵌入 + 元数据）
4. **存储文档元数据** - 写入 metadata_store
5. **创建块** - 调用 create_chunks
6. **批量嵌入生成** - 按 batch_size 分批调用 embedding_provider
7. **存储嵌入** - 批量写入 vector_store

### 错误处理

- 异常时自动回滚（restore 原始文档和块）
- 日志记录每个步骤
- IngestionError 异常包装

### 关键设计

- **幂等性**: content_unchanged 时跳过处理
- **回滚机制**: 失败时恢复到原始状态
- **批量处理**: batch_size 控制嵌入生成批次

---

## 模块：查询管道 (QueryPipeline)

**description**: 语义搜索和 LLM 问答

**purpose**: 提供知识库查询和问答能力

### 1. search(query, top_k, filters, repository_id, use_hybrid)

**流程**:
1. 生成查询嵌入
2. 执行向量搜索或混合搜索
3. 丰富结果（加载文档元数据）

**混合搜索 fallback**:
- 尝试 hybrid_search
- NotImplementedError 时回退到普通向量搜索

### 2. answer(query, top_k, max_context_length, repository_id, use_hybrid)

**流程**:
1. 调用 search 获取相关块
2. 构建上下文（限制总长度）
3. 调用 LLM 生成答案

**Prompt 模板**:
```
Context:
{context}

Question: {query}

Answer:
```

---

## 模块：数据实体

### 1. Document
```python
- id: UUID
- repository_id: UUID
- source_path: str
- doc_type: DocumentType (MARKDOWN/TEXT/PDF/HTML/UNKNOWN)
- title: str
- content: str
- content_md5: str | None  # 内容哈希，用于变更检测
- metadata: dict
```

### 2. Chunk
```python
- id: UUID
- repository_id: UUID
- document_id: UUID
- content: str
- chunk_index: int
- start_char: int
- end_char: int
- metadata: dict  # 包含 chunk_type 等
```

### 3. Embedding
```python
- id: UUID
- chunk_id: UUID
- vector: list[float]
- model: str
- dimension: int
```

### 4. Repository
```python
- id: UUID
- name: str
- description: str | None
- created_at: datetime
```

---

## 关键设计决策

### 1. Provider 抽象
**决策**: 定义 EmbeddingProvider 和 LLMProvider 抽象接口
**理由**: 支持多种 AI 提供者，便于测试和切换

### 2. 存储分离
**决策**: VectorStore 和 MetadataStore 分离
**理由**: 向量存储和元数据存储有不同需求，分离便于扩展

### 3. 多级分块 Fallback
**决策**: Markdown 分块：正则 → tree-sitter → 固定大小
**理由**: 逐级回退保证分块成功，每级利用更精确的语法解析

### 4. Pipeline 编排
**决策**: IngestionPipeline 和 QueryPipeline 分离
**理由**: 清晰职责分离，便于维护和测试

### 5. 内容变更检测
**决策**: MD5 哈希检测内容变更
**理由**: 高效检测，避免不必要重新处理

### 6. 批量嵌入
**决策**: batch_size 控制嵌入生成批次
**理由**: 控制内存使用，支持大文档处理

---

## 可复用的提示词模板

```
创建一个个人知识库系统：

架构：
- Python >= 3.11
- src/ 目录组织代码
- 模块：core（算法）、pipelines（业务流程）、providers（AI抽象）、storage（存储）、entities（数据模型）

核心模块：

1. 分块系统（多级 Fallback）：
   - 通用分块：固定大小 + 重叠窗口
   - Markdown 语义分块：正则解析 Markdown 结构（heading、list、code、table、blockquote）
   - Markdown 语法树分块：tree-sitter 解析
   - fallback 链：正则 → tree-sitter → 固定大小

2. Provider 抽象：
   - EmbeddingProvider: embed_text, embed_batch, get_dimension, get_max_tokens
   - LLMProvider: generate, count_tokens
   - ProviderConfig: provider_type, model_name, api_key, extra_params

3. 存储抽象：
   - VectorStore: add_embedding, search, delete_by_document_id
   - MetadataStore: add_document, get_document, list_documents
   - 分离设计支持多种向量存储后端

4. 摄取管道：
   - 文档发现 → 变更检测（MD5）→ 分块 → 嵌入生成 → 存储
   - 失败回滚机制
   - 批量处理控制

5. 查询管道：
   - 向量搜索或混合搜索
   - LLM 问答生成
   - 上下文长度限制

依赖选择：
- 数据验证：pydantic >= 2.0.0
- CLI：typer + rich
- 日志：structlog
- 数据库：aiosqlite
- 向量存储：chromadb
- 异步：httpx
```

---

## 扩展点

### 添加新的 Provider
1. 实现 EmbeddingProvider 或 LLMProvider 抽象
2. 在 config 中注册

### 添加新的 VectorStore
1. 实现 VectorStore 抽象
2. 实现 hybrid_search 方法（如支持）

### 添加新的分块策略
1. 在 core/ 创建新模块
2. 实现 chunk_document 函数
3. 在 chunking.py 的 create_chunks 中添加 fallback

### 添加新的文档类型
1. 在 DocumentType 枚举中添加
2. 在 _detect_document_type 中添加扩展名映射
3. 如需特殊分块，在 create_chunks 中添加处理逻辑
