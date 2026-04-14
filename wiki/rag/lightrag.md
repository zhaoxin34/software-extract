# LightRAG

> Sources: HKUDS, 2024
> Raw: [LightRAG extracted-knowledge](../../raw/LightRAG/extracted-knowledge.md); [LightRAG primary-report](../../raw/LightRAG/primary-report.md)

## Overview

LightRAG 是一个轻量级快速检索增强生成（RAG）框架，核心特点是**四层存储抽象** + **多模式查询** + **统一 Token 控制**。支持知识图谱增强、多种查询模式（local/global/hybrid/naive/mix/bypass）、以及丰富的存储后端（NetworkX/Neo4j/Milvus/PostgreSQL/MongoDB/Redis/OpenSearch）。

## 架构设计模式

### 1. 四层存储抽象

LightRAG 的核心设计是**四层存储抽象**，每种存储类型定义标准接口：

```python
STORAGE_IMPLEMENTATIONS = {
    "KV_STORAGE": {
        "implementations": ["JsonKVStorage", "RedisKVStorage", "PGKVStorage", ...],
        "required_methods": ["get_by_id", "upsert"],
    },
    "GRAPH_STORAGE": {
        "implementations": ["NetworkXStorage", "Neo4JStorage", "PGGraphStorage", ...],
        "required_methods": ["upsert_node", "upsert_edge"],
    },
    "VECTOR_STORAGE": {
        "implementations": ["NanoVectorDBStorage", "MilvusVectorDBStorage", ...],
        "required_methods": ["query", "upsert"],
    },
    "DOC_STATUS_STORAGE": {...},
}
```

**关键设计**：通过 `verify_storage_implementation()` 验证兼容性，通过 `STORAGE_ENV_REQUIREMENTS` 控制依赖。

### 2. 抽象基类设计

每种存储都有对应的抽象基类（`BaseVectorStorage`、`BaseKVStorage`、`BaseGraphStorage`），定义必须实现的接口：

```python
@dataclass
class BaseVectorStorage(StorageNameSpace, ABC):
    embedding_func: EmbeddingFunc
    cosine_better_than_threshold: float = 0.2

    @abstractmethod
    async def query(self, query: str, top_k: int, query_embedding: list[float]) -> list[dict]: ...

    @abstractmethod
    async def upsert(self, data: dict[str, dict[str, Any]]) -> None: ...
```

## 核心功能模块

### 模块 1: LightRAG 主入口 (lightrag.py)

**多模式查询**：支持 6 种查询模式
- `local`: 关注上下文依赖信息
- `global`: 利用全局知识
- `hybrid`: 结合 local 和 global
- `naive`: 基础搜索
- `mix`: 集成 KG 和向量检索（默认）
- `bypass`: 绕过检索

**Token 预算控制**：
```python
@dataclass
class QueryParam:
    max_entity_tokens: int  # 实体上下文上限
    max_relation_tokens: int  # 关系上下文上限
    max_total_tokens: int  # 总预算
```

### 模块 2: 操作层 (operate.py)

**文本分块** (`chunking_by_token_size`)：
- 支持按字符分割或按 token 分割
- 重叠 token 保持上下文连续性
- 超出限制时自动处理

**实体关系提取** (Map-Reduce 风格)：
1. 文本分块
2. 对每个 chunk 调用 LLM 提取实体/关系
3. 处理 N-ary 关系分解为二元关系
4. 合并重复实体和关系

**描述合并策略**：迭代式 map-reduce 平衡质量和成本
```python
while True:
    chunks = split_by_token_limit(descriptions)
    summaries = [summarize(chunk) for chunk in chunks]
    if len(summaries) <= 2:
        return final_summary
```

### 模块 3: 提示词工程 (prompt.py)

实体提取提示词模板结构化输出格式：
```
entity{tuple_delimiter}name{type}{description}
relation{src}{tgt}{keywords}{description}
```

通过 few-shot 示例减少 LLM 输出格式偏差。

### 模块 4: 重排序 (rerank.py)

`chunk_documents_for_rerank`：将长文档切分为适合 reranker 处理的小块
- max_tokens: 480（留 margin 给 512 限制）
- overlap_tokens: 32（保持上下文）

## 关键设计决策

### 决策 1: 异步优先架构

全面采用 asyncio 异步模式，I/O 密集型操作获得并发优势。`priority_limit_async_func_call` 支持优先级调度。

### 决策 2: 统一 Token 控制

在 QueryParam 中定义完整 token 预算，避免 LLM 上下文溢出，控制成本。

### 决策 3: Map-Reduce 描述合并

迭代式 map-reduce 处理任意长度描述，cooperative yield 避免阻塞事件循环。

## 接口设计

```python
# EmbeddingFunc Protocol
class EmbeddingFunc(Protocol):
    async def __call__(self, texts: list[str]) -> list[list[float]]: ...
    model_name: str | None
    embedding_dim: int
    max_token_size: int | None

# Storage 接口
interface VectorStorage:
    async def query(query: str, top_k: int) -> list[dict]
    async def upsert(data: dict[str, dict]) -> None

interface GraphStorage:
    async def upsert_node(node_id: str, data: dict) -> None
    async def upsert_edge(src: str, tgt: str, data: dict) -> None
```

## 扩展点

### 添加新的存储后端

1. 在 `kg/__init__.py` 添加实现映射
2. 继承对应基类（如 `BaseVectorStorage`）实现接口

### 添加新的 LLM 适配器

在 `llm/` 目录创建新文件，实现标准接口。

### 自定义分块策略

实现 `chunking_func` 签名并在初始化时配置：
```python
rag = LightRAG(chunking_func=custom_chunking)
```

## See Also

- [memory 知识库系统](../knowledge-base/memory.md) — 类似的知识库抽象和分块策略
