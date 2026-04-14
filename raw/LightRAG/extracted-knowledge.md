# LightRAG 精华萃取

## 项目概述

**项目名称**: LightRAG
**项目描述**: 轻量级快速检索增强生成(RAG)框架，支持知识图谱增强和多种查询模式
**目的**: 为LLM提供高效、可扩展的检索增强能力，支持多种存储后端和查询模式

---

## 架构设计模式

### 1. 存储分层抽象

LightRAG采用**四层存储抽象**：

```python
# kg/__init__.py - STORAGE_IMPLEMENTATIONS
STORAGE_IMPLEMENTATIONS = {
    "KV_STORAGE": {
        "implementations": ["JsonKVStorage", "RedisKVStorage", "PGKVStorage", "MongoKVStorage", "OpenSearchKVStorage"],
        "required_methods": ["get_by_id", "upsert"],
    },
    "GRAPH_STORAGE": {
        "implementations": ["NetworkXStorage", "Neo4JStorage", "PGGraphStorage", "MongoGraphStorage", "MemgraphStorage", "OpenSearchGraphStorage"],
        "required_methods": ["upsert_node", "upsert_edge"],
    },
    "VECTOR_STORAGE": {
        "implementations": ["NanoVectorDBStorage", "MilvusVectorDBStorage", "PGVectorStorage", "FaissVectorDBStorage", "QdrantVectorDBStorage", "MongoVectorDBStorage", "OpenSearchVectorDBStorage"],
        "required_methods": ["query", "upsert"],
    },
    "DOC_STATUS_STORAGE": {
        "implementations": ["JsonDocStatusStorage", "RedisDocStatusStorage", "PGDocStatusStorage", "MongoDocStatusStorage", "OpenSearchDocStatusStorage"],
        "required_methods": ["get_docs_by_status"],
    },
}
```

**关键设计决策**:
- 每种存储类型定义标准接口，具体的存储实现必须实现这些接口
- 通过`verify_storage_implementation()`验证兼容性
- 通过环境变量`STORAGE_ENV_REQUIREMENTS`控制各实现的依赖

### 2. 抽象基类设计

**BaseVectorStorage** (base.py:218-353):
```python
@dataclass
class BaseVectorStorage(StorageNameSpace, ABC):
    embedding_func: EmbeddingFunc
    cosine_better_than_threshold: float = field(default=0.2)
    meta_fields: set[str] = field(default_factory=set)

    @abstractmethod
    async def query(self, query: str, top_k: int, query_embedding: list[float] = None) -> list[dict[str, Any]]:
        """Query the vector storage"""

    @abstractmethod
    async def upsert(self, data: dict[str, dict[str, Any]]) -> None:
        """Insert or update vectors"""

    @abstractmethod
    async def delete_entity(self, entity_name: str) -> None:
        """Delete a single entity"""

    @abstractmethod
    async def delete_entity_relation(self, entity_name: str) -> None:
        """Delete relations for a given entity"""
```

**BaseKVStorage** (base.py:356-401):
```python
@dataclass
class BaseKVStorage(StorageNameSpace, ABC):
    @abstractmethod
    async def get_by_id(self, id: str) -> dict[str, Any] | None
    @abstractmethod
    async def get_by_ids(self, ids: list[str]) -> list[dict[str, Any]]
    @abstractmethod
    async def filter_keys(self, keys: set[str]) -> set[str]
    @abstractmethod
    async def upsert(self, data: dict[str, dict[str, Any]]) -> None
    @abstractmethod
    async def delete(self, ids: list[str]) -> None
    @abstractmethod
    async def is_empty(self) -> bool
```

**BaseGraphStorage** (base.py:405-600+):
```python
@dataclass
class BaseGraphStorage(StorageNameSpace, ABC):
    """All operations related to edges in graph should be undirected."""
    @abstractmethod
    async def has_node(self, node_id: str) -> bool
    @abstractmethod
    async def has_edge(self, source_node_id: str, target_node_id: str) -> bool
    @abstractmethod
    async def node_degree(self, node_id: str) -> int
    @abstractmethod
    async def get_node(self, node_id: str) -> dict[str, str] | None
    @abstractmethod
    async def upsert_node(self, node_id: str, node_data: dict[str, str]) -> None
    # ... 批量操作优化方法
    async def upsert_nodes_batch(self, nodes: list[tuple[str, dict[str, str]]]) -> None
    async def has_nodes_batch(self, node_ids: list[str]) -> set[str]
```

---

## 核心功能模块

### 模块1: LightRAG 主入口 (lightrag.py)

**描述**: 核心配置管理和API暴露
**目的**: 作为RAG系统的主入口，管理配置和协调各组件

**features**:
  - name: 多模式查询
    description: 支持6种查询模式
    specs:
      - name: 查询模式
        description: |
          - `local`: 关注上下文依赖信息
          - `global`: 利用全局知识
          - `hybrid`: 结合local和global
          - `naive`: 基础搜索
          - `mix`: 集成KG和向量检索
          - `bypass`: 绕过检索
        decision: 不同模式适用于不同查询场景，mix为默认

      - name: Token预算控制
        description: |
          统一token控制参数:
          - `max_entity_tokens`: 实体上下文token上限
          - `max_relation_tokens`: 关系上下文token上限
          - `max_total_tokens`: 总token预算
        decision: 通过预算控制避免上下文溢出

  - name: 存储后端配置
    description: 支持多种存储后端配置
    specs:
      - name: 可插拔存储
        description: |
          - `kv_storage`: KV存储 (默认JsonKVStorage)
          - `vector_storage`: 向量存储 (默认NanoVectorDBStorage)
          - `graph_storage`: 图存储 (默认NetworkXStorage)
          - `doc_status_storage`: 文档状态存储
        decision: 存储后端可替换，适应不同部署需求

  - name: LLM配置
    description: 灵活的LLM集成
    specs:
      - name: 模型配置
        description: |
          - `llm_model_func`: LLM调用函数
          - `embedding_func`: 向量化函数
          - `rerank_model_func`: 重排函数(可选)
        decision: 函数式配置，便于集成各种LLM服务商

### 模块2: 操作层 (operate.py)

**描述**: 核心操作逻辑：文本分块、实体提取、查询执行
**purpose**: 实现RAG的核心数据处理流程

**features**:
  - name: 文本分块算法
    description: 基于token大小的智能分块
    specs:
      - name: chunking_by_token_size
        description: |
          ```python
          def chunking_by_token_size(
              tokenizer: Tokenizer,
              content: str,
              split_by_character: str | None = None,
              split_by_character_only: bool = False,
              chunk_overlap_token_size: int = 100,
              chunk_token_size: int = 1200,
          ) -> list[dict[str, Any]]:
          ```
        decision: |
          - 支持按字符分割或按token分割
          - 重叠token保持上下文连续性
          - 超出限制时自动处理

  - name: 实体关系提取
    description: 从文本中提取实体和关系构建知识图谱
    specs:
      - name: extract_entities流程
        description: |
          1. 文本分块
          2. 对每个chunk调用LLM提取实体/关系
          3. 处理N-ary关系分解为二元关系
          4. 合并重复实体和关系
        decision: Map-Reduce风格的并行处理

      - name: 描述合并策略
        description: |
          ```python
          async def _handle_entity_relation_summary(
              description_type: str,
              entity_or_relation_name: str,
              description_list: list[str],
              separator: str,
              global_config: dict,
          ) -> tuple[str, bool]:
          ```
          使用迭代式map-reduce:
          1. 如果总tokens < context_size且描述数 < force_llm_summary，不需LLM
          2. 如果总tokens < max_tokens，直接LLM总结
          3. 否则分块总结，递归处理
        decision: 平衡质量和成本的动态策略

  - name: 查询执行
    description: 多种查询模式的实现
    specs:
      - name: naive_query
        description: 纯向量检索，直接从chunk搜索

      - name: kg_query
        description: |
          基于知识图谱的查询:
          1. 提取查询中的关键词
          2. 从KG检索相关实体
          3. 扩展到关联实体和关系
          4. 获取关联的chunks
        decision: 利用KG的结构化信息增强检索

---

### 模块3: 提示词工程 (prompt.py)

**description**: 精心设计的LLM提示词模板
**purpose**: 指导LLM准确提取实体和关系

**features**:
  - name: 实体提取提示词
    description: 结构化的实体关系提取
    specs:
      - name: entity_extraction_system_prompt
        description: |
          ```
          ---Role---
          You are a Knowledge Graph Specialist

          ---Instructions---
          1. Entity Extraction:
             - Identify meaningful entities
             - Extract: entity_name, entity_type, entity_description
             - Format: entity{tuple_delimiter}name{type}{description}

          2. Relationship Extraction:
             - Identify relationships between entities
             - Decompose N-ary to binary relations
             - Format: relation{src}{tgt}{keywords}{description}

          3. Delimiter Protocol: {tuple_delimiter} is atomic marker

          4. Language: Output in {language}
          ```
        decision: 清晰的格式要求减少解析错误

      - name: 示例系统
        description: 提供few-shot示例帮助LLM理解格式要求
        decision: 通过示例减少LLM输出格式偏差

---

### 模块4: 重排序 (rerank.py)

**description**: 检索后重排序提升质量
**purpose**: 对初步检索结果进行精细排序

**features**:
  - name: chunk_documents_for_rerank
    description: |
      将长文档切分为适合reranker处理的小块:
      - max_tokens: 480 (留margin给512限制)
      - overlap_tokens: 32 (保持上下文)
      - tokenizer_model: tiktoken或字符近似
    decision: 平衡rerank质量和上下文保留

---

## 关键设计决策

### 决策1: 异步优先架构

**决策**: 全面采用asyncio异步模式
**理由**:
- I/O密集型操作(数据库、网络)获得并发优势
- `async/await`提供清晰的异步流程控制
- `priority_limit_async_func_call`支持优先级调度

**fallback_strategies**:
- 使用`always_get_an_event_loop()`确保事件循环可用
- 异步操作支持超时控制

### 决策2: 统一Token控制

**决策**: 在QueryParam中定义完整的token预算
```python
@dataclass
class QueryParam:
    max_entity_tokens: int  # 实体上下文上限
    max_relation_tokens: int  # 关系上下文上限
    max_total_tokens: int  # 总预算
```
**理由**: 避免LLM上下文溢出，控制成本

### 决策3: 多存储后端可插拔

**决策**: 通过字符串配置选择存储实现
```python
kv_storage: str = "JsonKVStorage"  # 或 "RedisKVStorage", "PGKVStorage"
vector_storage: str = "NanoVectorDBStorage"  # 或 "MilvusVectorDBStorage"
```
**理由**: 适应不同部署环境，从开发到生产

### 决策4: Map-Reduce描述合并

**决策**: 迭代式map-reduce合并实体描述
```python
while True:
    # Map: 分块
    chunks = split_by_token_limit(descriptions)

    # Reduce: 总结每块
    summaries = [summarize(chunk) for chunk in chunks]

    # 检查是否可以结束
    if len(summaries) <= 2:
        return final_summary
```

**理由**:
- 处理任意长度描述
- 平衡LLM调用次数和结果质量
- cooperative yield避免阻塞事件循环

---

## 可复用的提示词模板

### 模板1: 实体关系提取

```python
ENTITY_EXTRACTION_PROMPT = """---Role---
You are a Knowledge Graph Specialist

---Instructions---
1. Extract entities with: name, type, description
2. Extract relationships between entities
3. Decompose N-ary to binary relations
4. Output format: {tuple_delimiter} separated fields
5. Use {completion_delimiter} when done

---Entity Types---
[{entity_types}]

---Input---
{input_text}

---Output---
"""
```

### 模板2: 描述总结

```python
SUMMARIZE_PROMPT = """---Task---
Summarize {description_type}: {description_name}

---Instructions---
1. Create a coherent summary
2. Keep important details
3. Target length: ~{summary_length} chars
4. Language: {language}

---Descriptions---
{description_list}

---Output---
"""
```

---

## 接口设计

### 接口1: EmbeddingFunc

```python
# utils.py
class EmbeddingFunc(Protocol):
    """Embedding function protocol"""
    async def __call__(self, texts: list[str]) -> list[list[float]]:
        """Generate embeddings for texts"""
        ...

    model_name: str | None  # 可选：模型名称
    embedding_dim: int  # 向量维度
    max_token_size: int | None  # 最大token数
```

### 接口2: Tokenizer

```python
class Tokenizer(Protocol):
    """Tokenizer protocol"""
    def encode(self, text: str) -> list[int]: ...
    def decode(self, tokens: list[int]) -> str: ...
    def get_token_count(self, text: str) -> int: ...
```

### 接口3: Storage接口

```python
# Vector Storage
interface VectorStorage:
    async def query(query: str, top_k: int) -> list[dict]
    async def upsert(data: dict[str, dict]) -> None
    async def delete(ids: list[str]) -> None

# KV Storage
interface KVStorage:
    async def get_by_id(id: str) -> dict | None
    async def upsert(data: dict[str, dict]) -> None
    async def filter_keys(keys: set[str]) -> set[str]

# Graph Storage
interface GraphStorage:
    async def upsert_node(node_id: str, data: dict) -> None
    async def upsert_edge(src: str, tgt: str, data: dict) -> None
    async def get_node(node_id: str) -> dict | None
    async def has_node(node_id: str) -> bool
```

---

## 技术栈

- **语言**: Python 3.10+
- **异步**: asyncio, async/await
- **向量数据库**: NanoVectorDB, Milvus, Faiss, Qdrant, PGVector, MongoDB, OpenSearch
- **图数据库**: NetworkX, Neo4j, Memgraph, PostgreSQL(AGE), MongoDB, OpenSearch
- **KV存储**: JSON文件, Redis, PostgreSQL, MongoDB, OpenSearch
- **LLM适配**: OpenAI, Anthropic, Ollama, Gemini, Azure OpenAI, HuggingFace, lmdeploy
- **Tokenization**: tiktoken
- **日志**: logging with SafeStreamHandler

---

## 关键文件路径

| 功能 | 文件路径 |
|------|----------|
| 主入口 | `lightrag/lightrag.py` |
| 核心操作 | `lightrag/operate.py` |
| 存储基类 | `lightrag/base.py` |
| 提示词 | `lightrag/prompt.py` |
| 工具函数 | `lightrag/utils.py` |
| 图工具 | `lightrag/utils_graph.py` |
| 重排序 | `lightrag/rerank.py` |
| 存储实现 | `lightrag/kg/*.py` |
| LLM适配 | `lightrag/llm/*.py` |
| API服务 | `lightrag/api/lightrag_server.py` |

---

## 扩展点

### 1. 添加新的存储后端

```python
# 1. 在 kg/__init__.py 添加实现映射
STORAGES["MyVectorDBStorage"] = ".kg.my_vector_impl"

# 2. 实现基类方法
class MyVectorDBStorage(BaseVectorStorage):
    async def query(self, query: str, top_k: int) -> list[dict]:
        ...
    async def upsert(self, data: dict) -> None:
        ...
```

### 2. 添加新的LLM适配器

```python
# 1. 在 llm/ 目录创建新文件
# 2. 实现标准接口
async def my_llm_call(prompt: str, **kwargs) -> str:
    # 调用你的LLM
    return response
```

### 3. 自定义分块策略

```python
# 实现分块函数签名
def custom_chunking(
    tokenizer: Tokenizer,
    content: str,
    split_by_character: str | None,
    split_by_character_only: bool,
    chunk_overlap_token_size: int,
    chunk_token_size: int,
) -> list[dict[str, Any]]:
    return [{"tokens": 100, "content": "...", "chunk_order_index": 0}]

# 配置使用
rag = LightRAG(chunking_func=custom_chunking)
```
