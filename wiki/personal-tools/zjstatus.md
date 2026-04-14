# zjstatus

> Sources: zjstatus project, 2024
> Raw: [zjstatus extracted-knowledge](../../raw/zjstatus/extracted-knowledge.md); [zjstatus primary-report](../../raw/zjstatus/primary-report.md)

## Overview

zjstatus 是 **Zellij 终端复用器的可配置状态栏插件**，核心特点是 **Widget Trait 模式**（可扩展的 Widget 插件系统）、**ANSI 颜色格式化引擎**（自定义 `#[fg:red]` 格式化字符串）、**KDL 配置解析**、以及**双二进制分发**（zjstatus + zjframes）。

## 整体架构

```
zjstatus (Rust WASM Plugin)
├── src/
│   ├── lib.rs              # 模块导出
│   ├── render.rs           # 渲染引擎（ANSI 格式化 + 缓存）
│   ├── config.rs           # 配置解析 + ZellijState 状态管理
│   ├── border.rs           # 边框配置
│   ├── frames.rs           # zjframes：窗格框架条件控制
│   ├── pipe.rs             # 管道通信
│   └── widgets/            # Widget 模块（Trait + 8种实现）
│       ├── widget.rs        # Widget trait 定义
│       ├── datetime.rs      # 日期时间
│       ├── mode.rs         # 模式显示
│       ├── command.rs      # 命令执行
│       ├── session.rs      # 会话信息
│       ├── tabs.rs         # 标签页
│       ├── notification.rs # 通知
│       ├── pipe.rs         # 管道
│       └── swap_layout.rs  # 布局切换
└── Cargo.toml
```

## 模块：Widget 系统

### Widget Trait

```rust
pub trait Widget {
    fn process(&self, name: &str, state: &ZellijState) -> String;
    fn process_click(&self, name: &str, state: &ZellijState, pos: usize);
}
```

**内置 Widget**：datetime, mode, command, session, tabs, notification, pipe, swap_layout

### 配置驱动

每个 Widget 从 `BTreeMap<String, String>` 读取配置，灵活且统一。

## 模块：渲染引擎

### FormattedPart 结构

表示一段格式化文本：
- `fg/bg/us`: 前景/背景/下划线颜色
- `effects`: 效果（bold, italic, reverse 等）
- `content`: 文本内容
- `cache_mask/cached_content`: 缓存相关

### 格式化字符串语法

```
#[fg:red]        红色前景
#[bg:#RRGGBB]    RGB 背景
#[effects:bold,underline]  多重效果
{format}, {date}, {time}   变量占位符
```

### 缓存策略

使用 `SizedCache` 缓存格式化结果，限制大小（100条），按配置字符串作为缓存键。

## 模块：配置系统

### ZellijState

全局运行时状态，集中管理所有状态，跨 Widget 共享数据：
- `cols`: 屏幕列数
- `command_results`: 命令执行结果缓存
- `pipe_results`: 管道消息缓存
- `mode`: 当前模式信息
- `panes`: 窗格清单
- `tabs`: 标签页列表
- `sessions`: 会话列表

### UpdateEventMask

控制何时更新各 Widget：
- `Always (0b10000000)`: 总是更新
- `Mode (0b00000001)`: 模式变化时更新
- `Tab (0b00000011)`: 标签变化时更新
- `Command (0b00000100)`: 命令结果变化时更新

## 模块：zjframes

条件化显示窗格框架，在特定条件下自动隐藏/显示窗格框架。

**关键选项**：
- `hide_frames_for_single_pane`: 单窗格时隐藏
- `hide_frames_except_for_search`: 搜索模式外隐藏
- `hide_frames_except_for_fullscreen`: 全屏模式外隐藏

**双二进制**：
- `zjstatus`: 主状态栏（含框架控制）
- `zjframes`: 仅框架控制（轻量）

## 关键设计决策

### 1. Widget Trait 模式

通过 Trait 定义统一接口，解耦 Widget 实现和渲染逻辑，方便扩展新 Widget 类型。

### 2. KDL 配置格式

使用 KDL 替代 JSON/TOML，语法简洁，可读性好，支持注释。

### 3. ANSI 颜色格式化

自定义 `#[fg:red]` 格式化字符串比 `\x1b[31m` 更易读，统一颜色和效果定义。

### 4. 缓存策略

使用 SizedCache 缓存格式化结果，WASM 内存受限，LRU 淘汰防止内存泄漏。

### 5. 双二进制分发

zjstatus 和 zjframes 分开编译，用户可按需选择，减少不必要的加载。
