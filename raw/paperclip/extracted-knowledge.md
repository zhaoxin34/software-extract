# 萃取的精华

## Paperclip 系统设计

**description**: AI Agent 企业编排系统 - 协调多个 AI Agent 执行业务目标的平台

**purpose**: 解决企业需要管理、协调多个 AI Agent（OpenClaw、Claude Code、Codex、Cursor 等）进行自动化工作的需求，提供组织架构、目标管理、预算控制、审批流程等企业级功能

---

## 整体架构

```
paperclip (Monorepo)
├── server/               # Node.js + Express 后端
├── cli/                  # 命令行工具
├── ui/                   # React + Vite 前端
└── packages/             # 共享包
    ├── adapters/           # 适配器实现
    ├── db/                 # Drizzle ORM + PostgreSQL
    └── shared/             # 共享类型和工具
```

---

## 模块：前端架构

**description**: React 19 + Vite + Tailwind CSS 4 构建的 Web UI

**purpose**: 提供 Web UI 管理 Agent 团队

**tech_stack**:
  - category: UI组件库
    items:
      - radix-ui: ^1.4.3（无头 UI 组件基础）
      - @radix-ui/react-slot: ^1.2.4
  - category: CSS框架
    items:
      - tailwindcss: ^4.0.7
      - @tailwindcss/typography: ^0.5.19
  - category: 路由
    items:
      - react-router-dom: ^7.1.5
  - category: 数据请求/状态管理
    items:
      - @tanstack/react-query: ^5.90.21
  - category: 构建工具
    items:
      - vite: ^6.1.0
      - @tailwindcss/vite: ^4.0.7
  - category: 图标库
    items:
      - lucide-react: ^0.574.0
  - category: 拖拽/排序
    items:
      - @dnd-kit/core: ^6.3.1
      - @dnd-kit/sortable: ^10.0.0
      - @dnd-kit/utilities: ^3.2.2
  - category: 富文本/编辑器
    items:
      - lexical: 0.35.0
      - @lexical/link: 0.35.0
      - @mdxeditor/editor: ^3.52.4
  - category: AI 对话 UI
    items:
      - @assistant-ui/react: 0.12.23
  - category: 图表/可视化
    items:
      - mermaid: ^11.12.0
  - category: 命令菜单
    items:
      - cmdk: ^1.1.1
  - category: Markdown 处理
    items:
      - react-markdown: ^10.1.0
      - remark-gfm: ^4.0.1
  - category: 样式工具
    items:
      - class-variance-authority: ^0.7.1
      - clsx: ^2.1.1
      - tailwind-merge: ^3.4.1

**features**:

- **Feature: 页面模块**
  - description: Agent 管理界面、目标管理界面、任务看板、审批面板、成本仪表盘
  - decision: React Router 路由管理，TanStack Query 数据请求

- **Feature: 拖拽交互**
  - description: 使用 @dnd-kit 实现任务看板的拖拽排序功能
  - decision: 选择 @dnd-kit 而非 react-beautiful-dnd（已废弃）

- **Feature: 富文本编辑**
  - description: Lexical 作为核心编辑器，支持 Markdown 编辑
  - decision: Lexical 是 React 官方推荐的编辑器框架

- **Feature: AI 对话界面**
  - description: @assistant-ui/react 提供 AI 对话组件
  - decision: 集成第三方 AI UI 组件加速开发

- **Feature: 命令菜单**
  - description: cmdk 提供命令面板功能
  - decision: 轻量级命令菜单实现

- **Feature: 图表绘制**
  - description: mermaid 用于绘制架构图、流程图
  - decision: Mermaid 是常见的 Markdown 图表方案

---

## 模块：后端架构

**description**: Express.js 5 + Drizzle ORM + PostgreSQL

**purpose**: 提供 HTTP API 和业务逻辑处理

**tech_stack**:
  - category: API框架
    items:
      - express: ^5.1.0
  - category: 数据库ORM
    items:
      - drizzle-orm: ^0.38.4
  - category: 数据库
    items:
      - embedded-postgres: ^18.1.0-beta.16（开发/测试）
      - PostgreSQL: 生产环境
  - category: 认证/授权
    items:
      - better-auth: 1.4.18
  - category: S3存储
    items:
      - @aws-sdk/client-s3: ^3.888.0
  - category: 图片处理
    items:
      - sharp: ^0.34.5
  - category: 文件上传
    items:
      - multer: ^2.1.1
  - category: WebSocket
    items:
      - ws: ^8.19.0
  - category: 日志
    items:
      - pino: ^9.6.0
      - pino-http: ^10.4.0
      - pino-pretty: ^13.1.3
  - category: 数据验证
    items:
      - zod: ^3.24.2
      - ajv: ^8.18.0
      - ajv-formats: ^3.0.1
  - category: 环境变量
    items:
      - dotenv: ^17.0.1
  - category: DOM处理
    items:
      - jsdom: ^28.1.0
      - dompurify: ^3.3.2
  - category: 文件监控
    items:
      - chokidar: ^4.0.3
  - category: 端口检测
    items:
      - detect-port: ^2.1.0

**features**:

- **Feature: RESTful API**
  - description: Fastify 风格的路由模块化设计
  - key_routes:
    - agents.ts: Agent CRUD
    - goals.ts: 目标管理
    - issues.ts: 任务管理
    - approvals.ts: 审批
    - budgets.ts: 预算
    - costs.ts: 成本查询
    - companies.ts: 公司管理
    - projects.ts: 项目管理
    - dashboard.ts: 仪表盘数据
    - activity.ts: 活动日志
    - authz.ts: 权限控制

- **Feature: 认证系统**
  - description: better-auth 实现用户认证
  - decision: 选用 better-auth 而非 Passport（更现代）

- **Feature: 存储服务**
  - description: 抽象存储接口，支持本地磁盘和 S3
  - decision: 工厂模式创建存储实例

---

## 模块：CLI 架构

**description**: Commander.js 构建的命令行工具

**purpose**: 提供本地开发和管理能力

**tech_stack**:
  - category: CLI框架
    items:
      - commander: ^13.1.0
      - @clack/prompts: ^0.10.0
  - category: 构建工具
    items:
      - esbuild: ^0.27.3
  - category: 样式输出
    items:
      - picocolors: ^1.1.1

**features**:

- **Feature: 命令结构**
  - description: paperclipai CLI 主命令
  - subcommands: dev, dev:watch, dev:once, dev:list, dev:stop, dev:server, dev:ui

- **Feature: 开发服务**
  - description: 管理本地开发服务器
  - decision: tsx 运行 TypeScript

---

## 模块：适配器架构

**description**: 抽象多类型 Agent 接入的适配器系统

**purpose**: 支持多种 AI Agent（OpenClaw、Claude Code、Codex、Cursor 等）的灵活接入

**responsibilities**:
- 定义统一的适配器接口
- 管理适配器注册表
- 支持外部适配器插件加载

**features**:

- **Feature: ServerAdapterModule 适配器接口**
  - description: 所有适配器必须实现的统一接口
  - key_methods:
    - execute(): 执行 Agent 任务
    - listSkills(): 列出可用技能
    - syncSkills(): 同步技能
    - testEnvironment(): 测试环境配置
    - sessionCodec(): 会话编解码
    - models(): 可用模型列表
  - decision:
    - 适配器类型独立实现，无需共享基类
    - 通过统一接口保证插件化

- **Feature: AdapterRegistry 适配器注册表**
  - file: `server/src/adapters/registry.ts`
  - description: 集中管理所有内置和外部适配器

- **Feature: PluginLoader 外部适配器加载器**
  - file: `server/src/adapters/plugin-loader.ts`
  - description: 从 `~/.paperclip/adapter-plugins.json` 加载外部适配器

---

## 模块：数据库层

**description**: Drizzle ORM + PostgreSQL 数据持久化

**purpose**: 提供类型安全的数据访问

**key_tables**:
- agents: Agent 定义
- agent_config_revisions: 配置版本历史
- agent_runtime_state: 运行时状态
- goals: 目标
- issues: 任务/工单
- approvals: 审批记录
- budget_policies: 预算策略
- budget_incidents: 预算事件
- cost_events: 成本事件
- heartbeat_runs: 心跳运行
- activity_log: 活动日志
- companies: 公司/组织
- projects: 项目
- documents: 文档

---

## 关键设计决策

### 1. 适配器插件化架构
**决策**: 通过统一接口支持多种 Agent 类型
**理由**: 企业需要灵活的 Agent 选择，新 Agent 类型可通过插件扩展

### 2. 配置版本化
**决策**: Agent 配置变更不覆盖，保存版本历史
**理由**: 支持审计追溯、配置回滚、类 Git 的设计哲学

### 3. 多层次预算控制
**决策**: 预算策略支持 company/project/agent 三层范围
**理由**: 灵活的成本控制粒度，支持月度预算和终身预算

### 4. 存储抽象
**决策**: 统一存储接口，支持本地磁盘和 S3
**理由**: 开发环境使用本地存储，生产环境使用 S3

### 5. 心脏跳动监控
**决策**: 通过心跳机制监控 Agent 状态
**理由**: 实时了解 Agent 工作状态，记录每次运行的输入输出

### 6. Monorepo 组织
**决策**: server + cli + ui + packages 在同一个 repo
**理由**: 便于代码共享、统一版本管理、便于协作开发

---

## 可复用的提示词模板

```
创建一个 AI Agent 企业编排系统：

架构：
- Monorepo: server（后端）+ cli（命令行）+ ui（前端）+ packages（共享包）
- 后端：Node.js + Express 5 + Drizzle ORM + PostgreSQL
- 前端：React 19 + Vite + Tailwind CSS 4
- 共享包：shared（类型）+ db（数据库）+ adapters（适配器）

前端技术栈（详细）：
- UI组件：Radix UI（@radix-ui/react-slot）
- CSS：Tailwind CSS 4 + @tailwindcss/typography
- 路由：React Router 7
- 数据请求：TanStack Query 5
- 图标：Lucide React
- 拖拽：@dnd-kit（core + sortable + utilities）
- 编辑器：Lexical + @mdxeditor/editor
- AI对话：@assistant-ui/react
- 图表：Mermaid
- 命令菜单：cmdk
- 样式工具：class-variance-authority + clsx + tailwind-merge

后端技术栈（详细）：
- 框架：Express 5
- ORM：Drizzle ORM
- 认证：better-auth
- 存储：AWS S3 SDK + Sharp
- 验证：Zod + AJV
- 日志：Pino
- WebSocket：ws

适配器架构：
- 定义 ServerAdapterModule 接口（execute, listSkills, syncSkills, testEnvironment, sessionCodec, models）
- 适配器注册表管理所有内置和外部适配器
- 外部适配器通过 JSON 配置动态加载

服务层：
- AgentService：CRUD + 配置版本化（每次变更保存快照）
- GoalService：层级目标管理（company → project → issue）
- BudgetService：多层次预算（lifetime/monthly）+ 状态监控（ok/warning/hard_stop）
- ApprovalService：Agent 产出审批工作流
- HeartbeatService：Agent 状态监控和心跳记录
- CostService：成本追踪和聚合

数据库：
- Drizzle ORM Schema 定义所有表
- 核心表：agents, goals, issues, approvals, budget_policies, cost_events, heartbeat_runs
- 支持嵌入式 PostgreSQL 用于测试

API 设计：
- Express RESTful API
- 路由模块化（agents, goals, issues, approvals, budgets, costs, dashboard）
- better-auth 认证
```

---

## 扩展点

### 添加新的 Agent 适配器
1. 在 `packages/adapters/` 创建新适配器包
2. 实现 `ServerAdapterModule` 接口
3. 在 `server/src/adapters/registry.ts` 注册

### 添加新的存储后端
1. 实现 `StorageProvider` 接口
2. 在 `storage/provider-registry.ts` 添加创建逻辑

### 添加新的业务服务
1. 在 `server/src/services/` 创建服务文件
2. 使用 Drizzle ORM 进行数据访问
3. 在路由中调用服务
