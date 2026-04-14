# 初级分析报告：plato

## 项目概述

- **项目名称**: Plato - 运营策划编辑系统
- **项目类型**: 前后端分离的 Web 应用
- **技术栈**:
  - 后端：Python + FastAPI + SQLAlchemy + MySQL
  - 前端：React 19 + Vite + TypeScript + Ant Design

## 项目结构

```
plato/
├── backend/               # FastAPI + SQLAlchemy + MySQL
│   ├── src/app/
│   │   ├── api/          # FastAPI 路由
│   │   ├── auth_providers/ # 认证提供者（GitHub OAuth等）
│   │   ├── core/         # 核心配置
│   │   ├── dependencies/ # 依赖注入
│   │   ├── llm/          # LLM 架构层（抽象接口）
│   │   │   ├── base.py       # BaseLLM 抽象基类
│   │   │   ├── registry.py   # LLM 注册表
│   │   │   └── providers/     # 具体 Provider 实现
│   │   ├── models/      # SQLAlchemy 模型
│   │   ├── prompts/      # 业务 Prompt 模板
│   │   ├── schemas/      # Pydantic 模型
│   │   └── services/     # 业务逻辑层
│   └── migrations/       # 数据库迁移脚本
├── frontend/              # React + Vite + TypeScript
│   └── src/
│       ├── api/          # API 客户端（axios）
│       ├── components/    # React 组件
│       ├── layouts/      # 布局组件
│       └── pages/         # 页面组件
├── openspec/             # OpenSpec 工作流
└── docs/                # 文档
```

## 核心发现

### 1. 产品定位

**背景**: 运营策划对话式编辑系统

**核心理念**: "一个文档，多个版本，LLM 驱动编辑，类 Git 版本控制"

**核心功能**:
- 对话编辑：通过自然语言对话修改 Markdown 文档
- 版本控制：每次修改自动创建版本，记录 diff
- 历史回溯：支持回退到任意版本
- 异步摘要生成：LLM 自动生成版本变化说明

### 2. 后端架构（三层设计）

这是 Plato 最精华的设计，严格分层：

1. **LLM 架构层** (`llm/`)
   - 提供商无关的抽象（`BaseLLM` 接口）
   - 注册表模式管理提供商（`LLMRegistry`）
   - 支持：mock、placeholder、OpenAI 兼容 API
   - **核心原则**：此层不包含业务逻辑

2. **业务 Prompt 层** (`prompts/`)
   - 业务特定的 prompt 模板
   - 为 LLM 层构建 `Prompt` 对象
   - 示例：`version_diff.py` 生成版本变化摘要
   - **核心原则**：业务逻辑在此层，不在 LLM 层

3. **服务层** (`services/`)
   - `llm_service.py`：连接业务层和 LLM 层
   - `document_service.py`：文档 CRUD + 版本管理
   - `version_service.py`：版本历史操作

### 3. LLM 抽象设计

**BaseLLM 抽象基类**：
```python
class BaseLLM(ABC):
    @abstractmethod
    def generate(self, prompt: Prompt) -> LLMResponse:
        pass
```

**LLMRegistry 注册表**：
```python
class LLMRegistry:
    def register(self, name: str, provider_class: Type[BaseLLM]):
        ...

    def get(self, name: str = "mock") -> BaseLLM:
        ...

    def list_providers(self) -> list[str]:
        ...
```

**Provider 实现**：
- `MockLLM`：开发测试用
- `PlaceholderLLM`：占位实现
- `OpenAILLM`：OpenAI 兼容 API

### 4. 版本控制设计

**关键**：回退操作创建新版本，永不覆盖历史。

**流程**：
1. 用户修改文档（对话或直接编辑）
2. 使用 `difflib.unified_diff` 生成 diff
3. 创建 `DocumentVersion` 记录
4. 更新 `Document.current_content`
5. 异步后台任务通过 LLM 生成变化摘要

### 5. 异步后台任务

使用 FastAPI 的 `BackgroundTasks` 进行非阻塞操作：
```python
@router.post("/chat")
def chat(request: ChatRequest, background_tasks: BackgroundTasks, db: Session):
    background_tasks.add_task(generate_diff_summary, version_id, old, new)
```

**注意**：不要在同步上下文中使用 `asyncio.create_task()`（不会执行）。

### 6. 前端架构

**布局**：Header / Content（分屏：Markdown 编辑器 | 对话面板）/ Footer

**核心组件**：
- `MarkdownEditor.tsx`：MDEditor 文档编辑器
- `ChatPanel.tsx`：LLM 编辑的对话界面
- `VersionPanel.tsx`：版本历史

**API 客户端**：统一封装 axios 实例，配置响应拦截器

### 7. 数据库设计

**Schema**（MySQL 两表）：
- `documents`：当前文档状态（id, current_content, current_version）
- `document_versions`：版本历史（id, document_id, version_number, content_md, diff, change_summary）

## 项目价值评估

- **设计完整性**: ★★★★★（三层架构清晰，职责分明）
- **代码实现**: ★★★★☆（后端完整，前端骨架）
- **可复用性**: ★★★★★（LLM 抽象、版本控制模式可复用）
- **文档完整性**: ★★★★★（CLAUDE.md 详细）

## 萃取建议

该项目适合萃取的"精华"：

1. **LLM 抽象架构**：BaseLLM + LLMRegistry + Provider 模式
2. **三层分层设计**：LLM架构层 → 业务Prompt层 → 服务层
3. **版本控制模式**：类 Git 但"回退创建新版本"
4. **异步摘要生成**：BackgroundTasks + LLM 生成变化说明
5. **Monorepo 结构**：前后端分离项目组织
6. **前后端 API 约定**：RESTful API + axios 封装
