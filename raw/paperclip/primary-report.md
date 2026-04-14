# Paperclip 初级分析报告

## 项目基本信息

- **项目名称**: Paperclip
- **项目类型**: 源码项目（Monorepo）
- **核心功能**: AI Agent 企业编排系统 - 协调多个 AI Agent 执行业务目标的平台
- **项目地址**: https://github.com/paperclipai/paperclip
- **版本**: 0.3.1
- **许可证**: MIT

## 项目结构

```
paperclip (Monorepo)
├── server/               # Node.js + Express 后端
├── cli/                  # 命令行工具
├── ui/                   # React + Vite 前端
├── packages/             # 共享包
│   ├── adapters/         # 适配器实现
│   ├── db/               # Drizzle ORM + PostgreSQL
│   └── shared/           # 共享类型和工具
└── docs/                 # 文档
```

## 技术栈详细分析

### 1. 前端 (ui/)

**依赖配置文件**: `ui/package.json`

| 类别 | 库名称 | 版本 | 说明 |
|------|--------|------|------|
| **UI组件库** | radix-ui | ^1.4.3 | 无头 UI 组件库（@radix-ui/react-slot） |
| **CSS框架** | tailwindcss | ^4.0.7 | Tailwind CSS 4 |
| **状态管理** | @tanstack/react-query | ^5.90.21 | 数据请求和缓存 |
| **路由** | react-router-dom | ^7.1.5 | 页面路由管理 |
| **构建工具** | vite | ^6.1.0 | 打包/开发服务器 |
| **图标库** | lucide-react | ^0.574.0 | 图标组件 |
| **拖拽/排序** | @dnd-kit/core | ^6.3.1 | 拖拽核心库 |
| **拖拽/排序** | @dnd-kit/sortable | ^10.0.0 | 拖拽排序组件 |
| **拖拽/排序** | @dnd-kit/utilities | ^3.2.2 | 拖拽工具函数 |
| **富文本/编辑器** | lexical | 0.35.0 | 富文本编辑器核心 |
| **富文本/编辑器** | @lexical/link | 0.35.0 | Lexical 链接插件 |
| **富文本/编辑器** | @mdxeditor/editor | ^3.52.4 | Markdown 编辑器 |
| **AI 对话 UI** | @assistant-ui/react | 0.12.23 | AI 对话组件 |
| **图表/可视化** | mermaid | ^11.12.0 | 图表绘制 |
| **数据请求** | @tanstack/react-query | ^5.90.21 | HTTP 客户端/数据获取 |
| **样式工具** | class-variance-authority | ^0.7.1 | CSS 变体工具 |
| **样式工具** | clsx | ^2.1.1 | CSS 类名拼接 |
| **样式工具** | tailwind-merge | ^3.4.1 | Tailwind 合并工具 |
| **命令菜单** | cmdk | ^1.1.1 | 命令菜单组件 |
| **Markdown** | react-markdown | ^10.1.0 | Markdown 渲染 |
| **Markdown** | remark-gfm | ^4.0.1 | GitHub Flavored Markdown |

### 2. 后端 (server/)

**依赖配置文件**: `server/package.json`

| 类别 | 库名称 | 版本 | 说明 |
|------|--------|------|------|
| **API框架** | express | ^5.1.0 | HTTP 服务框架 |
| **数据库ORM** | drizzle-orm | ^0.38.4 | 数据库 ORM |
| **数据库** | embedded-postgres | ^18.1.0-beta.16 | 嵌入式 PostgreSQL |
| **认证/授权** | better-auth | 1.4.18 | 用户认证 |
| **文件上传** | multer | ^2.1.1 | 文件上传处理 |
| **S3存储** | @aws-sdk/client-s3 | ^3.888.0 | AWS S3 客户端 |
| **图片处理** | sharp | ^0.34.5 | 图片处理 |
| **WebSocket** | ws | ^8.19.0 | WebSocket 支持 |
| **日志** | pino | ^9.6.0 | 日志库 |
| **环境变量** | dotenv | ^17.0.1 | 环境变量管理 |
| **数据验证** | zod | ^3.24.2 | 数据验证 |
| **DOM处理** | jsdom | ^28.1.0 | DOM 模拟 |
| **HTML净化** | dompurify | ^3.3.2 | HTML 净化 |
| **文件监控** | chokidar | ^4.0.3 | 文件监控 |
| **端口检测** | detect-port | ^2.1.0 | 端口检测 |

### 3. CLI (cli/)

**依赖配置文件**: `cli/package.json`

| 类别 | 库名称 | 版本 | 说明 |
|------|--------|------|------|
| **CLI框架** | commander | ^13.1.0 | 命令行工具框架 |
| **CLI交互** | @clack/prompts | ^0.10.0 | 命令行交互提示 |
| **构建工具** | esbuild | ^0.27.3 | 打包/编译 |
| **样式输出** | picocolors | ^1.1.1 | 彩色终端输出 |

### 4. 根项目 (package.json)

| 类别 | 库名称 | 版本 | 说明 |
|------|--------|------|------|
| **测试框架** | vitest | ^3.0.5 | 单元测试 |
| **测试框架** | @playwright/test | ^1.58.2 | E2E 测试 |
| **包管理器** | pnpm | 9.15.4 | Monorepo 包管理 |

## 核心模块

1. **适配器系统** - 支持多种 AI Agent（Claude Code、Codex、Cursor 等）
2. **存储抽象** - 支持本地磁盘和 S3 存储
3. **Agent 管理** - CRUD + 配置版本化
4. **目标管理** - 层级目标树（company → project → issue）
5. **预算控制** - 多层次预算策略
6. **审批流程** - Agent 产出审批工作流
7. **心跳监控** - Agent 状态监控

## 关键技术决策

1. **Monorepo 组织** - server + cli + ui + packages
2. **适配器插件化** - 统一接口支持多种 Agent
3. **配置版本化** - Agent 配置变更保存历史
4. **存储抽象** - StorageProvider 接口
5. **嵌入式数据库** - 开发环境使用 embedded-postgres
