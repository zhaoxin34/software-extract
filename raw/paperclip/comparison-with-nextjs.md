# Paperclip 与 Next.js 架构对比

## 核心区别

| | Paperclip | Next.js |
|---|---|---|
| **架构** | Monorepo（前后端分离） | 前后端合一 |
| **前端** | React + Vite + react-router-dom | React + Next.js 自己的文件路由 |
| **后端** | Express（独立 server/ 目录） | API Routes / Route Handlers |
| **构建** | Vite | Turbopack |

## Paperclip 的实际结构

```
paperclip (Monorepo)
├── server/   # Node.js + Express 后端（独立服务）
├── ui/       # React + Vite 前端（独立服务）
└── packages/ # 共享包
```

## 关键区别

1. **Paperclip**：
   - 前端用 `Vite` 而非 Next.js
   - 后端是独立的 `Express` 服务，监听独立端口
   - 前后端通过 HTTP API 通信

2. **Next.js**：
   - 前端和后端在同一个 Node.js 进程
   - 使用文件系统路由（pages 或 app 目录）
   - API Routes 是框架内置的

## 总结

Paperclip 的架构更接近「传统前后端分离」模式（React/Vite 前端 + Express 后端），而不是 Next.js 的全栈合一模式。
