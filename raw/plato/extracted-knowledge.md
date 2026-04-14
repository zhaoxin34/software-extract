# 萃取的精华

## Plato 系统设计

**description**: 运营策划编辑系统 - 对话式 Markdown 文档编辑，支持版本控制和 LLM 驱动编辑

**purpose**: 解决运营策划人员需要通过自然语言对话编辑文档、追踪历史版本、回溯修改的需求

---

## 整体架构

```
plato (Monorepo)
├── backend/           # FastAPI + SQLAlchemy + MySQL
│   ├── llm/         # LLM 架构层（抽象）
│   ├── prompts/      # 业务 Prompt 层
│   ├── services/    # 服务层
│   ├── api/         # API 路由
│   └── models/      # SQLAlchemy 模型
└── frontend/         # React + Vite + TypeScript
    ├── components/   # UI 组件
    ├── layouts/      # 布局
    └── api/         # API 客户端
```

---

## 模块：LLM 架构层

**description**: 提供商无关的 LLM 抽象层

**purpose**: 分离业务逻辑和 LLM 提供商，支持灵活切换和扩展

**responsibilities**:
- 定义 LLM 抽象接口
- 管理 LLM Provider 注册表
- 提供 Provider 实例化

**algorithms**:

- **Feature: BaseLLM 抽象基类**
  - **file**: `backend/src/app/llm/base.py`
  - **description**: 所有 LLM Provider 必须继承的抽象基类
  - **key_methods**:
    - `generate(prompt: Prompt) -> LLMResponse`: 核心生成方法
    - `get_model_name() -> str`: 返回模型名称（可选）
  - **decision**:
    - 同步接口（暂不支持 streaming）
    - 不暴露提供商特定参数
    - 错误处理由具体实现决定

- **Feature: LLMRegistry 注册表模式**
  - **file**: `backend/src/app/llm/registry.py`
  - **description**: 管理所有 LLM Provider 的注册和获取
  - **key_methods**:
    - `register(name, provider_class)`: 注册 Provider
    - `get(name) -> BaseLLM`: 获取 Provider 实例
    - `list_providers() -> list[str]`: 列出所有已注册 Provider
  - **decision**:
    - 使用 Dict[str, Type[BaseLLM]] 存储注册表
    - 条件注册：OpenAI Provider 只有配置了 API Key 才注册
    - 全局单例模式

- **Feature: Provider 实例化策略**
  - **description**: 不同 Provider 使用不同的构造策略
  - **decision**:
    - OpenAI: 传递配置参数（model, api_key, base_url, temperature, max_tokens, stream）
    - 其他: 使用默认构造

**interfaces**:

- **LLM 生成**
  - input: `Prompt` 对象（system_prompt, user_prompt）
  - output: `LLMResponse` 对象（content, usage, model）
  - usage: 业务层调用 LLM 生成文本

- **Provider 获取**
  - input: Provider 名称（如 "openai", "mock"）
  - output: `BaseLLM` 实例
  - usage: 服务层获取 LLM 实例

**tech_stack**:
- 抽象基类（ABC）
- 注册表模式（Registry Pattern）
- 工厂模式（Factory）

**key_files**:
- `llm/base.py`: BaseLLM 抽象
- `llm/registry.py`: LLMRegistry
- `llm/providers/`: 具体 Provider 实现

---

## 模块：业务 Prompt 层

**description**: 业务特定的 Prompt 模板

**purpose**: 将 Prompt 模板作为业务资产管理，与 LLM 架构层解耦

**responsibilities**:
- 构建业务特定的 Prompt 对象
- 管理 Prompt 模板
- 与 LLM 层通过标准 Prompt 对象交互

**algorithms**:

- **Feature: Prompt 构建模式**
  - **description**: 分离 Prompt 构建和 LLM 调用
  - **key_prompts**:
    - `build_document_edit_prompt()`: 构建文档编辑 Prompt
    - `build_version_diff_prompt()`: 构建版本差异分析 Prompt
  - **decision**:
    - Prompt 构建是业务逻辑，在 prompts/ 模块
    - LLM 层只接收标准 Prompt 对象，不关心业务

---

## 模块：服务层

**description**: 业务逻辑编排层

**purpose**: 连接 LLM 层和 API 层，处理具体业务逻辑

**responsibilities**:
- 文档 CRUD + 版本管理
- 调用 LLM 生成内容
- 管理版本历史

**features**:

- **Feature: DocumentService**
  - **description**: 文档管理 + 版本创建
  - **key_methods**:
    - `get_or_create_document()`: 获取或创建文档
    - `update_document()`: 更新文档并创建版本
  - **Spec: 版本创建流程**
    - description: |
        1. 获取旧内容
        2. 生成 unified_diff
        3. 创建 DocumentVersion 记录
        4. 更新 Document.current_content
        5. 触发异步摘要生成
  - **decision**:
    - 单文档模式（系统中始终只有一个文档）
    - 版本号自增，不复用

- **Feature: LLMService**
  - **description**: LLM 调用服务
  - **key_methods**:
    - `generate_modified_document()`: 生成修改后的文档
    - `generate_version_diff()`: 生成版本差异说明
  - **Spec: 重试逻辑**
    - description: |
        - 最大重试 3 次
        - 递增等待时间（1s * retry_count）
        - 失败返回 None

**interfaces**:

- **文档更新**
  - input: new_content, background_tasks
  - output: Document 对象
  - side_effect: 创建版本记录，触发异步摘要生成

- **版本差异生成**
  - input: old_content, new_content
  - output: 差异说明（Markdown 格式）或 None
  - retry: 最多 3 次

**tech_stack**:
- FastAPI BackgroundTasks
- difflib（标准库）
- SQLAlchemy ORM

**fallback_strategies**:

- **异步摘要生成失败**:
  - description: BackgroundTasks 未提供时跳过，不阻塞主流程
  - 处理: 记录日志警告，不抛出异常

---

## 模块：版本控制

**description**: 类 Git 的版本控制系统

**purpose**: 记录文档历史，支持回溯，但不覆盖历史

**responsibilities**:
- 创建版本记录
- 生成 diff
- 管理回退逻辑

**algorithms**:

- **Feature: 差异生成**
  - **description**: 使用 difflib 生成 unified diff
  - **implementation**:
    ```python
    diff_lines = list(
        difflib.unified_diff(
            old_content.splitlines(keepends=True),
            new_content.splitlines(keepends=True),
            lineterm="",
        )
    )
    ```

- **Feature: 回退策略**
  - **description**: 回退不是覆盖，而是创建新版本
  - **decision**:
    - 读取目标版本内容
    - 将该内容作为新修改
    - 调用正常的更新流程
    - 生成新版本记录（不修改历史）

---

## 模块：前端架构

**description**: React + Ant Design 前端应用

**purpose**: 提供可视化的文档编辑和对话交互界面

**responsibilities**:
- 文档编辑
- 对话交互
- 版本历史展示

**features**:

- **Feature: 布局结构**
  - **description**: Header / Content（分屏）/ Footer
  - **Content 分屏**:
    - 左侧：Markdown 编辑器
    - 右侧：对话面板

- **Feature: ChatPanel**
  - **description**: 对话交互界面
  - **key_behavior**:
    - 调用后端 `/chat` API
    - 使用后端返回的完整 Markdown 更新编辑器
    - 错误由 HTTP 拦截器统一处理

- **Feature: VersionPanel**
  - **description**: 版本历史展示
  - **key_behavior**:
    - 显示版本列表和变化摘要
    - 支持 Markdown 渲染
    - 摘要超过 100 字符时展开/收起

**interfaces**:

- **API 客户端封装**
  - **description**: 统一的 axios 实例
  - **config**:
    - baseURL: `http://localhost:8000`
    - timeout: 30000
    - 响应拦截器统一错误处理

**tech_stack**:
- React 19
- Vite
- TypeScript
- Ant Design
- axios

**key_components**:
- `MarkdownEditor.tsx`: MDEditor 编辑器
- `ChatPanel.tsx`: 对话面板
- `VersionPanel.tsx`: 版本历史

---

## 关键设计决策

### 1. 三层分层架构

```
LLM 架构层 (llm/)
    ↓ 生成 Prompt
业务 Prompt 层 (prompts/)
    ↓ 接收 Prompt
服务层 (services/)
    ↓ 调用
LLM 架构层 → BaseLLM.generate()
```

**决策理由**:
- LLM 架构层不包含业务逻辑
- 业务 Prompt 层不关心具体 LLM 实现
- 服务层是连接点，只关心业务

### 2. 回退创建新版本

**决策**: 回退不是覆盖历史，而是创建新版本

**决策理由**:
- 保证审计追溯
- 任何操作都有记录
- 类 Git 的设计哲学

### 3. 条件注册 Provider

**决策**: OpenAI Provider 只有配置了 API Key 才注册

**决策理由**:
- 支持本地开发（无 API Key 时使用 mock）
- 渐进式配置

### 4. 单文档模式

**决策**: 系统中始终只有一个文档

**决策理由**:
- 简化设计，聚焦核心功能
- 符合"一个文档，多个版本"的理念

### 5. 异步摘要生成

**决策**: 版本变化摘要通过 BackgroundTasks 异步生成

**决策理由**:
- 不阻塞主流程
- LLM 调用耗时不影响用户体验
- 第一个版本不生成（无对比）

---

## 可复用的提示词模板

```
创建一个 LLM 驱动的文档编辑系统：

后端架构：
- 三层分层：LLM架构层(llm/) → 业务Prompt层(prompts/) → 服务层(services/)
- LLM 抽象：BaseLLM 抽象基类 + LLMRegistry 注册表
- Provider 模式：支持 mock、placeholder、OpenAI 兼容 API
- 单文档模式：系统只有一个文档，通过版本号管理
- 版本控制：每次修改创建版本，回退创建新版本（不覆盖）
- 差异生成：使用 difflib.unified_diff
- 异步摘要：FastAPI BackgroundTasks + LLM 生成变化说明

前端架构：
- React + TypeScript + Ant Design
- 布局：Header / Content(分屏) / Footer
- 核心组件：MarkdownEditor、ChatPanel、VersionPanel
- API 客户端：axios 封装，响应拦截器统一错误处理
```

---

## 扩展点

### 添加新的 LLM Provider

1. 创建 `llm/providers/your_provider.py`
2. 继承 `BaseLLM` 并实现 `generate()` 方法
3. 在 `LLMRegistry` 中注册

### 添加新的业务功能

1. 在 `prompts/` 添加新的 Prompt 构建函数
2. 在 `services/` 添加新的服务方法
3. 在 `api/` 添加新的路由

### 前端添加新页面

1. 在 `pages/` 添加新组件
2. 在 `App.tsx` 添加路由
