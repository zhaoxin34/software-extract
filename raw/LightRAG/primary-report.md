# LightRAG 初级分析报告

## 项目概述

**项目名称**: LightRAG (HKUDS/LightRAG)
**项目类型**: 源码项目 (Python RAG框架)
**技术栈**: Python 3.10+, asyncio, NetworkX, Neo4j, Milvus, PostgreSQL, MongoDB, Redis, OpenSearch
**开源协议**: License文件中未明确声明
**项目地址**: https://github.com/HKUDS/LightRAG

## 项目结构

```
LightRAG/
├── lightrag/                    # 核心库
│   ├── lightrag.py             # 主入口 (~199KB)
│   ├── operate.py              # 核心操作逻辑 (~205KB)
│   ├── base.py                 # 基础抽象类 (~32KB)
│   ├── prompt.py               # 提示词模板 (~29KB)
│   ├── utils.py                # 工具函数 (~123KB)
│   ├── utils_graph.py          # 图工具 (~73KB)
│   ├── rerank.py               # 重排序 (~20KB)
│   ├── constants.py            # 常量定义
│   ├── types.py                # 类型定义
│   ├── exceptions.py           # 异常定义
│   ├── namespace.py            # 命名空间
│   ├── kg/                     # 存储实现
│   │   ├── networkx_impl.py    # NetworkX图存储
│   │   ├── neo4j_impl.py      # Neo4j图存储
│   │   ├── milvus_impl.py     # Milvus向量存储
│   │   ├── postgres_impl.py   # PostgreSQL通用存储
│   │   ├── mongo_impl.py      # MongoDB通用存储
│   │   ├── opensearch_impl.py # OpenSearch统一存储
│   │   ├── redis_impl.py      # Redis存储
│   │   ├── json_kv_impl.py    # JSON文件KV存储
│   │   ├── json_doc_status_impl.py
│   │   ├── nano_vector_db_impl.py
│   │   └── shared_storage.py  # 共享存储
│   ├── llm/                    # LLM适配器
│   │   ├── openai.py
│   │   ├── anthropic.py
│   │   ├── ollama.py
│   │   ├── gemini.py
│   │   ├── azure_openai.py
│   │   ├── hf.py
│   │   ├── jina.py
│   │   └── lmdeploy.py
│   ├── api/                    # REST API服务
│   │   ├── lightrag_server.py
│   │   ├── routers/
│   │   └── run_with_gunicorn.py
│   └── evaluation/             # 评估模块
├── lightrag_webui/            # Web界面
├── tests/                     # 测试用例
├── reproduce/                 # 复现脚本
└── docs/                      # 文档
```

## 核心功能概述

LightRAG是一个**轻量级快速检索增强生成(RAG)框架**，其核心特点：

1. **多模式查询**: 支持 local、global、hybrid、naive、mix、bypass 六种查询模式
2. **知识图谱增强**: 从文本中提取实体和关系构建知识图谱
3. **统一Token控制**: 完整的token预算管理系统
4. **多存储后端**: 支持多种向量数据库、图数据库、KV存储
5. **重排序机制**: 支持Reranker提升检索质量
6. **异步架构**: 基于asyncio的异步处理

## 模块职责分析

| 模块 | 职责 | 关键文件 |
|------|------|----------|
| lightrag.py | 主入口，配置管理，API暴露 | ~199KB |
| operate.py | 文本分块、实体提取、查询执行 | ~205KB |
| base.py | 存储抽象基类定义 | ~32KB |
| prompt.py | LLM提示词模板 | ~29KB |
| kg/* | 各种存储后端实现 | 多种 |
| llm/* | LLM服务商适配器 | 多种 |

## 判断

LightRAG是一个**高质量的生产级RAG框架**，具有以下特点：

**优点**:
- 完整的架构设计，模块职责清晰
- 支持多种存储后端，可扩展性强
- 完善的异步处理机制
- 丰富的提示词模板和查询模式
- 统一token控制系统

**精华部分**:
- 实体/关系提取的提示词工程
- Map-Reduce风格的描述合并策略
- Token预算控制算法
- 多存储后端的抽象接口设计
- 重排序机制

**适合萃取**: 架构设计模式、存储抽象、多模态查询编排、Token控制策略
