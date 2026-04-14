# memory 初级分析报告

## 项目基本信息

- **项目名称**: memory
- **项目类型**: Python 源码项目（个人知识库系统）
- **核心功能**: 支持语义搜索和基于 LLM 问答的个人知识库系统
- **Python 版本**: >=3.11
- **许可证**: MIT

## 项目结构

```
memory/
├── src/memory/
│   ├── core/           # 核心功能
│   │   ├── chunking.py           # 通用分块策略
│   │   ├── markdown_chunking.py   # Markdown 语义分块（正则）
│   │   ├── tree_sitter_chunking.py # Markdown 语法树分块
│   │   └── logging.py            # 日志系统
│   ├── pipelines/      # 业务流程编排
│   │   ├── ingestion.py          # 文档摄取管道
│   │   └── query.py              # 查询管道
│   ├── providers/       # AI 提供者抽象
│   │   ├── base.py              # 抽象基类
│   │   ├── openai_embd.py       # OpenAI Embedding
│   │   ├── openai_llm.py       # OpenAI LLM
│   │   └── local.py             # 本地模型
│   ├── storage/        # 存储抽象
│   │   ├── base.py              # 存储抽象基类
│   │   ├── sqlite.py            # SQLite 元数据存储
│   │   └── chroma.py            # Chroma 向量存储
│   ├── entities/       # 数据实体
│   │   ├── document.py
│   │   ├── chunk.py
│   │   ├── embedding.py
│   │   └── search_result.py
│   ├── config/         # 配置管理
│   │   ├── schema.py
│   │   └── loader.py
│   ├── interfaces/     # CLI 接口
│   │   └── cli.py
│   ├── service/        # 服务层
│   │   ├── repository.py
│   │   └── stores.py
│   └── eval/          # 评估模块
│       └── evaluate.py
├── tests/             # 测试
└── pyproject.toml     # 依赖配置
```

## 技术栈详细分析

### 依赖配置文件: pyproject.toml

#### 核心依赖 (dependencies)

| 类别 | 库名称 | 版本 | 说明 |
|------|--------|------|------|
| **数据验证** | pydantic | >=2.0.0 | 数据模型和验证 |
| **配置管理** | pydantic-settings | >=2.0.0 | Pydantic 配置支持 |
| **CLI框架** | typer | >=0.9.0 | CLI 应用程序框架 |
| **终端输出** | rich | >=13.0.0 | 富文本终端输出 |
| **日志** | structlog | >=23.0.0 | 结构化日志 |
| **环境变量** | python-dotenv | >=1.0.0 | .env 文件加载 |
| **数据库** | aiosqlite | >=0.22.1 | 异步 SQLite |
| **HTTP客户端** | httpx[socks] | >=0.28.1 | 异步 HTTP 客户端 |
| **中文分词** | jieba | >=0.42.1 | 中文分词库 |
| **向量存储** | chromadb | >=1.5.2 | 向量数据库 |
| **分词器** | snowballstemmer | >=3.0.1 | 词干提取 |

#### 可选依赖 (optional-dependencies)

**Embedding 提供者**:
| 类别 | 库名称 | 版本 |
|------|--------|------|
| openai | openai | >=1.0.0 |
| local | sentence-transformers | >=2.2.0 |

**向量存储**:
| 类别 | 库名称 | 版本 |
|------|--------|------|
| chroma | chromadb | >=0.4.0 |
| qdrant | qdrant-client | >=1.7.0 |
| faiss | faiss-cpu | >=1.7.0 |

**文档处理**:
| 类别 | 库名称 | 版本 |
|------|--------|------|
| pdf | pypdf, pdfplumber | >=3.0.0, >=0.10.0 |
| web | beautifulsoup4, requests | >=4.12.0, >=2.31.0 |

**高级分块**:
| 类别 | 库名称 | 版本 |
|------|--------|------|
| tree-sitter | tree-sitter, tree-sitter-markdown | >=0.23.0, >=0.4.0 |

**评估**:
| 类别 | 库名称 | 版本 |
|------|--------|------|
| eval | ragas, langchain-openai | >=0.1.0 |

**开发**:
| 类别 | 库名称 | 版本 |
|------|--------|------|
| 测试 | pytest, pytest-asyncio, pytest-cov | >=7.4.0 |
| 类型检查 | mypy | >=1.5.0 |
| 代码格式 | ruff, black | >=0.1.0, >=23.0.0 |

## 核心模块

1. **分块系统** - 通用分块、Markdown 语义分块（正则）、Markdown 语法树分块（tree-sitter）
2. **存储抽象** - VectorStore 和 MetadataStore 接口分离
3. **Provider 抽象** - EmbeddingProvider 和 LLMProvider 接口
4. **摄取管道** - 文档摄取完整流程（加载、分块、嵌入、存储）
5. **查询管道** - 语义搜索和 LLM 问答
6. **CLI 接口** - Typer 构建的命令行工具

## 关键技术决策

1. **Provider 抽象模式** - 支持多种 Embedding/LLM 提供者（OpenAI、本地模型）
2. **存储分离** - VectorStore 和 MetadataStore 分离，支持多种向量存储
3. **Markdown 语义分块** - 多级 fallback：正则 → tree-sitter → 固定大小
4. **Pipeline 编排** - 清晰的摄取/查询流程分离
5. **结构化日志** - 使用 structlog 统一日志格式
