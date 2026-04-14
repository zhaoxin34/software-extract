# Paperclip

> Sources: Paperclip AI, 2024
> Raw: [paperclip extracted-knowledge](../../raw/paperclip/extracted-knowledge.md); [paperclip primary-report](../../raw/paperclip/primary-report.md)

## Overview

Paperclip 是一个**AI Agent 企业编排系统**，用于协调多个 AI Agent（OpenClaw、Claude Code、Codex、Cursor 等）执行业务目标。核心特点是**适配器插件化架构**、**Monorepo 组织**、**配置版本化**、**多层次预算控制**、以及**心脏跳动监控**。

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

## 模块：前端架构

**技术栈**：React 19 + Vite + Tailwind CSS 4

| 类别 | 库 | 说明 |
|------|-----|------|
| UI组件 | Radix UI | 无头 UI 组件基础 |
| 富文本编辑 | Lexical + @mdxeditor/editor | React 官方推荐编辑器 |
| AI对话 | @assistant-ui/react | AI 对话组件 |
| 拖拽 | @dnd-kit | 替代 react-beautiful-dnd |
| 图表 | Mermaid | Markdown 图表方案 |
| 命令菜单 | cmdk | 轻量级命令面板 |
| 数据请求 | TanStack Query 5 | HTTP 客户端/缓存 |

## 模块：后端架构

**技术栈**：Express.js 5 + Drizzle ORM + PostgreSQL

| 类别 | 库 | 说明 |
|------|-----|------|
| 认证 | better-auth | 更现代的认证方案 |
| 存储 | AWS S3 SDK + Sharp | S3 存储 + 图片处理 |
| 验证 | Zod + AJV | 数据验证 |
| 日志 | Pino | 高性能日志库 |
| WebSocket | ws | WebSocket 支持 |

## 模块：适配器架构

这是 Paperclip 最核心的设计 — **适配器插件化架构**。

### ServerAdapterModule 接口

```typescript
interface ServerAdapterModule {
    execute(): Promise<void>;              // 执行 Agent 任务
    listSkills(): Promise<string[]>;      // 列出可用技能
    syncSkills(): Promise<void>;           // 同步技能
    testEnvironment(): Promise<boolean>;    // 测试环境配置
    sessionCodec(): SessionCodec;           // 会话编解码
    models(): Model[];                      // 可用模型列表
}
```

**关键设计**：适配器类型独立实现，无需共享基类，通过统一接口保证插件化。

### 适配器注册表

`AdapterRegistry` 集中管理所有内置和外部适配器，`PluginLoader` 从 `~/.paperclip/adapter-plugins.json` 加载外部适配器。

## 模块：数据库层

核心表结构：
- `agents`: Agent 定义
- `agent_config_revisions`: 配置版本历史
- `goals`: 层级目标（company → project → issue）
- `issues`: 任务/工单
- `approvals`: 审批记录
- `budget_policies`: 预算策略（lifetime/monthly）
- `budget_incidents`: 预算事件
- `cost_events`: 成本事件
- `heartbeat_runs`: 心跳运行
- `activity_log`: 活动日志

## 关键设计决策

### 1. 适配器插件化架构

通过统一接口支持多种 Agent 类型，新 Agent 类型可通过插件扩展。

### 2. 配置版本化

Agent 配置变更不覆盖，保存版本历史，支持审计追溯和配置回滚。

### 3. 多层次预算控制

预算策略支持 company/project/agent 三层范围，灵活的成本控制粒度。

### 4. 心脏跳动监控

通过心跳机制监控 Agent 状态，实时了解 Agent 工作状态，记录每次运行的输入输出。

### 5. Monorepo 组织

server + cli + ui + packages 在同一个 repo，便于代码共享、统一版本管理。
