# 初级分析报告：zjstatus

## 项目概述

- **项目名称**: zjstatus & zjframes - Zellij 终端复用器的状态栏插件
- **项目类型**: Rust WASM 插件
- **技术栈**:
  - Rust (Edition 2024)
  - zellij-tile (WASM 插件接口)
  - chrono (时间处理)
  - regex (正则匹配)
  - kdl (配置解析)
  - cached (缓存)

## 项目结构

```
zjstatus/
├── src/
│   ├── lib.rs           # 库入口，模块导出
│   ├── render.rs        # 渲染引擎（ANSI 颜色格式化）
│   ├── config.rs        # 配置解析和状态管理
│   ├── border.rs        # 边框配置
│   ├── frames.rs        # zjframes 窗格框架控制
│   ├── pipe.rs          # 管道通信
│   ├── widgets/         # Widget 模块
│   │   ├── widget.rs    # Widget trait 定义
│   │   ├── mod.rs
│   │   ├── datetime.rs  # 日期时间 Widget
│   │   ├── mode.rs     # 模式 Widget
│   │   ├── command.rs   # 命令 Widget
│   │   ├── session.rs   # 会话 Widget
│   │   ├── tabs.rs      # 标签 Widget
│   │   ├── notification.rs # 通知 Widget
│   │   ├── pipe.rs      # 管道 Widget
│   │   └── swap_layout.rs # 布局切换 Widget
│   └── bin/
│       └── main.rs      # 双二进制入口（zjstatus + zjframes）
├── Cargo.toml
├── examples/           # 配置示例
└── assets/             # 截图等资源
```

## 核心发现

### 1. 产品定位

**背景**: Zellij 终端复用器的状态栏插件

**核心理念**: "A configurable and themable statusbar for zellij"

**核心功能**:
- 可配置的状态栏显示
- 支持多个 Widget（datetime, mode, command, session, tabs 等）
- ANSI 颜色格式化
- KDL 配置文件解析
- zjframes: 条件化显示窗格框架

### 2. 核心架构

**Widget 系统**:
- `Widget` trait 定义了统一接口
- `process()`: 处理并返回渲染字符串
- `process_click()`: 处理点击事件
- 支持的 Widget: datetime, mode, command, session, tabs, notification, pipe, swap_layout

**渲染引擎**:
- `FormattedPart` 结构：管理前景色、背景色、样式
- ANSI 颜色支持（256色、RGB）
- 格式化字符串解析：`#[fg:red]` 这种格式
- 缓存机制：`cached` crate

**配置系统**:
- KDL 格式配置文件
- `ZellijState`: 全局状态（列数、命令结果、管道结果、模式、窗格等）
- `UpdateEventMask`: 更新事件掩码

### 3. 关键设计

**双二进制**:
- `zjstatus`: 主状态栏插件
- `zjframes`: 独立的窗格框架控制插件

**Widget Trait**:
```rust
pub trait Widget {
    fn process(&self, name: &str, state: &ZellijState) -> String;
    fn process_click(&self, name: &str, state: &ZellijState, pos: usize);
}
```

**格式化字符串**:
- `#[fg:red]` 设置前景色
- `#[bg:blue]` 设置背景色
- `#[effects:bold]` 设置效果
- 支持嵌套组合

**缓存策略**:
- `SizedCache` 缓存格式化结果
- 按配置字符串作为缓存键

### 4. 依赖特点

- `zellij-tile`: WASM 插件接口
- `chrono` + `chrono-tz`: 时区时间处理
- `kdl`: KDL 配置文件解析
- `cached`: 结果缓存
- `anstyle`: ANSI 样式

## 项目价值评估

- **设计完整性**: ★★★★☆（Widget 模式清晰，渲染系统完善）
- **代码实现**: ★★★★☆（Rust 代码质量高）
- **可复用性**: ★★★★☆（Widget trait、渲染引擎可复用）
- **文档完整性**: ★★★☆☆（README 清晰，配置示例丰富）
