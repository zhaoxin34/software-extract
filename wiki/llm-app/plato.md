# Plato

> Sources: Plato project, 2024
> Raw: [plato extracted-knowledge](../../raw/plato/extracted-knowledge.md); [plato primary-report](../../raw/plato/primary-report.md)

## Overview

Plato 是一个**运营策划编辑系统**，核心理念是"一个文档，多个版本，LLM 驱动编辑，类 Git 版本控制"。最精华的设计是**三层分层架构**（LLM架构层 → 业务Prompt层 → 服务层）、**版本控制模式**（回退创建新版本，永不覆盖历史）、以及**异步摘要生成**（BackgroundTasks + LLM）。

## 整体架构

```
plato (Monorepo)
├── backend/               # FastAPI + SQLAlchemy + MySQL
│   ├── llm/          # LLM 架构层（抽象接口）
│   ├── prompts/      # 业务 Prompt 模板
│   └── services/     # 业务逻辑层
└── frontend/           # React 19 + Vite + TypeScript + Ant Design
```

## 模块：LLM 架构层

这是 Plato 最核心的设计 — **提供商无关的 LLM 抽象**。

### BaseLLM 抽象基类

```python
class BaseLLM(ABC):
    @abstractmethod
    def generate(self, prompt: Prompt) -> LLMResponse:
        pass
```

### LLMRegistry 注册表

```python
class LLMRegistry:
    def register(self, name: str, provider_class: Type[BaseLLM]): ...
    def get(self, name: str = "mock") -> BaseLLM: ...
    def list_providers(self) -> list[str]: ...
```

**条件注册**：OpenAI Provider 只有配置了 API Key 才注册，支持本地开发（无 API Key 时使用 mock）。

### Provider 实现

- `MockLLM`：开发测试用
- `PlaceholderLLM`：占位实现
- `OpenAILLM`：OpenAI 兼容 API

## 模块：三层分层架构

```
LLM 架构层 (llm/)
    ↓ 生成 Prompt
业务 Prompt 层 (prompts/)
    ↓ 接收 Prompt
服务层 (services/)
    ↓ 调用
LLM 架构层 → BaseLLM.generate()
```

**核心原则**：
- LLM 架构层不包含业务逻辑
- 业务 Prompt 层不关心具体 LLM 实现
- 服务层是连接点，只关心业务

## 模块：版本控制

**关键原则**：回退不是覆盖历史，而是创建新版本。

### 流程

1. 用户修改文档（对话或直接编辑）
2. 使用 `difflib.unified_diff` 生成 diff
3. 创建 `DocumentVersion` 记录
4. 更新 `Document.current_content`
5. 异步后台任务通过 LLM 生成变化摘要

### 回退策略

读取目标版本内容 → 将该内容作为新修改 → 调用正常更新流程 → 生成新版本记录（不修改历史）

## 模块：异步后台任务

使用 FastAPI 的 `BackgroundTasks` 进行非阻塞操作：

```python
@router.post("/chat")
def chat(request: ChatRequest, background_tasks: BackgroundTasks, db: Session):
    background_tasks.add_task(generate_diff_summary, version_id, old, new)
```

**注意**：不要在同步上下文中使用 `asyncio.create_task()`（不会执行）。

## 模块：前端架构

**布局**：Header / Content（分屏：Markdown 编辑器 | 对话面板）/ Footer

**核心组件**：
- `MarkdownEditor.tsx`：MDEditor 文档编辑器
- `ChatPanel.tsx`：LLM 编辑的对话界面
- `VersionPanel.tsx`：版本历史

**API 客户端**：统一封装 axios 实例，配置响应拦截器统一错误处理。

## 数据库设计

**Schema**（MySQL 两表）：
- `documents`：当前文档状态（id, current_content, current_version）
- `document_versions`：版本历史（id, document_id, version_number, content_md, diff, change_summary）

## 关键设计决策

### 1. 三层分层架构

LLM 架构层、业务 Prompt 层、服务层严格分离，职责分明。

### 2. 回退创建新版本

回退操作创建新版本，永不覆盖历史，保证审计追溯。

### 3. 条件注册 Provider

OpenAI Provider 只有配置了 API Key 才注册，支持本地开发。

### 4. 单文档模式

系统中始终只有一个文档，简化设计，聚焦核心功能。

### 5. 异步摘要生成

版本变化摘要通过 BackgroundTasks 异步生成，不阻塞主流程。

## See Also

- [memory 知识库系统](../knowledge-base/memory.md) — 类似的三层分离设计（providers/pipelines/storage）
