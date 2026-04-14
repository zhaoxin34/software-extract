# 萃取的精华

## zjstatus 系统设计

**description**: Zellij 终端复用器的可配置状态栏插件

**purpose**: 为 Zellij 提供高度可定制、可主题化的状态栏，支持 Widget 扩展和 ANSI 颜色格式化

---

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

---

## 模块：Widget 系统

**description**: 可扩展的 Widget 插件系统

**purpose**: 通过统一接口支持多种状态栏组件，方便扩展新功能

**responsibilities**:
- 定义 Widget 标准接口
- 提供基础实现（datetime, mode, command 等）
- 处理点击事件

**features**:

- **Feature: Widget Trait**
  - **file**: `src/widgets/widget.rs`
  - **description**: 所有 Widget 必须实现的 trait
  - **key_methods**:
    - `process(name, state) -> String`: 处理并返回渲染字符串
    - `process_click(name, state, pos)`: 处理点击事件
  - **decision**:
    - 简单接口设计，易于实现
    - 状态通过 ZellijState 传入

- **Feature: DateTimeWidget**
  - **file**: `src/widgets/datetime.rs`
  - **description**: 日期时间显示 Widget
  - **key_config**:
    - `datetime_format`: 整体格式
    - `datetime_time_format`: 时间格式
    - `datetime_date_format`: 日期格式
    - `datetime_timezone`: 时区
  - **decision**:
    - 使用 chrono + chrono-tz 处理时区
    - 支持 strftime 格式字符串

- **Feature: CommandWidget**
  - **file**: `src/widgets/command.rs`
  - **description**: 命令执行并显示输出
  - **key_config**:
    - `command`: 要执行的命令
    - `interval`: 执行间隔（秒）
    - `render_mode`: Static/Dynamic/Raw
    - `hide_on_empty_stdout`: 空输出时隐藏
  - **Spec: 命令缓存**
    - description: 命令执行结果缓存在 ZellijState.command_results
    - 通过 interval 控制更新频率

- **Feature: ModeWidget**
  - **description**: 显示当前 Zellij 模式（Normal, Insert, Scroll 等）

- **Feature: SessionWidget**
  - **description**: 显示会话信息

- **Feature: TabsWidget**
  - **description**: 显示标签页列表

- **Feature: NotificationWidget**
  - **description**: 显示通知消息

- **Feature: PipeWidget**
  - **description**: 处理管道消息

- **Feature: SwapLayoutWidget**
  - **description**: 动态切换布局

**interfaces**:

- **Widget 处理**
  - input: name, ZellijState
  - output: 格式化后的渲染字符串
  - usage: 主渲染循环调用

- **Widget 点击**
  - input: name, ZellijState, position
  - output: 无（副作用）
  - usage: 处理用户交互

**tech_stack**:
- Rust Trait
- chrono (时间)
- chrono-tz (时区)

**key_files**:
- `src/widgets/widget.rs`: Trait 定义
- `src/widgets/datetime.rs`: DateTime Widget 示例实现

---

## 模块：渲染引擎

**description**: ANSI 颜色和样式格式化系统

**purpose**: 将格式化字符串转换为 ANSI 转义序列

**responsibilities**:
- 解析格式化字符串
- 管理颜色和样式
- 提供缓存机制

**features**:

- **Feature: FormattedPart 结构**
  - **file**: `src/render.rs`
  - **description**: 表示一段格式化文本
  - **key_fields**:
    - fg/bg/us: 前景/背景/下划线颜色
    - effects: 效果（bold, italic, reverse 等）
    - content: 文本内容
    - cache_mask/cached_content: 缓存相关
  - **decision**:
    - 使用 anstyle crate 处理 ANSI 样式
    - 支持 256 色和 RGB 颜色

- **Feature: 格式化字符串语法**
  - **description**: `#[key:value]` 格式
  - **examples**:
    - `#[fg:red]` - 红色前景
    - `#[bg:#RRGGBB]` - RGB 背景
    - `#[effects:bold,underline]` - 多重效果
  - **decision**:
    - KDL 风格，易于解析
    - 支持嵌套组合

- **Feature: 格式化缓存**
  - **description**: 使用 cached crate 缓存解析结果
  - **key_methods**:
    - `formatted_part_from_string_cached()`: 单段缓存
    - `formatted_parts_from_string_cached()`: 多段缓存
  - **decision**:
    - SizedCache 限制缓存大小（100条）
    - 按配置字符串作为缓存键

**algorithms**:

- **格式化字符串解析**
  - 使用正则 `\{[a-z_0-9]+\}` 提取变量占位符
  - `split("#[")` 分割颜色指令和文本
  - 支持变量替换：`{format}`, `{date}`, `{time}`

**tech_stack**:
- anstyle (ANSI 样式)
- cached (缓存)
- regex (正则)

**key_files**:
- `src/render.rs`: 渲染引擎核心

---

## 模块：配置系统

**description**: KDL 配置解析和全局状态管理

**purpose**: 解析用户配置，维护运行时状态

**responsibilities**:
- 解析 KDL 配置文件
- 管理 ZellijState 状态
- 定义更新事件掩码

**features**:

- **Feature: ZellijState**
  - **file**: `src/config.rs`
  - **description**: 全局运行时状态
  - **key_fields**:
    - cols: 屏幕列数
    - command_results: 命令执行结果缓存
    - pipe_results: 管道消息缓存
    - mode: 当前模式信息
    - panes: 窗格清单
    - tabs: 标签页列表
    - sessions: 会话列表
    - start_time: 启动时间
    - cache_mask: 缓存掩码
  - **decision**:
    - 集中管理所有状态
    - 跨 Widget 共享数据

- **Feature: UpdateEventMask**
  - **description**: 控制何时更新各 Widget
  - **values**:
    - Always (0b10000000): 总是更新
    - Mode (0b00000001): 模式变化时更新
    - Tab (0b00000011): 标签变化时更新
    - Command (0b00000100): 命令结果变化时更新
    - Session (0b00001000): 会话变化时更新
    - None (0b00000000): 不更新

- **Feature: ModuleConfig**
  - **description**: 单个模块的配置
  - **key_fields**:
    - left_parts_config / center_parts_config / right_parts_config: 三段配置
    - format_space: 段之间分隔符

**interfaces**:

- **配置解析**
  - input: KDL 格式配置字符串
  - output: BTreeMap<String, String>
  - usage: Widget 初始化

- **状态更新**
  - input: Zellij 事件
  - output: 更新 ZellijState
  - usage: 主循环

**tech_stack**:
- kdl (配置解析)
- BTreeMap (配置存储)

---

## 模块：zjframes

**description**: 条件化显示窗格框架

**purpose**: 在特定条件下自动隐藏/显示窗格框架

**responsibilities**:
- 检测条件状态
- 切换窗格框架显示

**features**:

- **Feature: FrameConfig**
  - **file**: `src/frames.rs`
  - **description**: 框架显示配置
  - **key_options**:
    - hide_frames_for_single_pane: 单窗格时隐藏
    - hide_frames_except_for_search: 搜索模式外隐藏
    - hide_frames_except_for_fullscreen: 全屏模式外隐藏
    - hide_frames_except_for_scroll: 滚动模式外隐藏

- **Feature: 条件判断**
  - `should_show_frames_for_scroll()`: 滚动模式
  - `should_show_frames_for_search()`: 搜索模式
  - `should_show_frames_for_fullscreen()`: 全屏模式
  - `should_show_frames_for_multiple_panes()`: 多窗格

- **Feature: 双二进制支持**
  - **description**: zjstatus 和 zjframes 是两个独立的 WASM 二进制
  - **decision**:
    - zjstatus: 状态栏 + 框架控制
    - zjframes: 仅框架控制（轻量）
    - 避免重复加载

**tech_stack**:
- tracing (日志)
- zellij-tile (WASM API)

---

## 关键设计决策

### 1. Widget Trait 模式

**决策**: 通过 Trait 定义统一接口

**决策理由**:
- 解耦 Widget 实现和渲染逻辑
- 方便扩展新 Widget 类型
- 测试简单（Mock Widget）

### 2. KDL 配置格式

**决策**: 使用 KDL 替代 JSON/TOML

**决策理由**:
- Zellij 原生使用 KDL
- 语法简洁，可读性好
- 支持注释

### 3. ANSI 颜色格式化

**决策**: 自定义格式化字符串语法

**决策理由**:
- `#[fg:red]` 比 `\x1b[31m` 更易读
- 统一颜色和效果定义
- 便于主题化

### 4. 缓存策略

**决策**: 使用 SizedCache 缓存格式化结果

**决策理由**:
- WASM 内存受限
- 避免重复解析
- LRU 淘汰防止内存泄漏

### 5. 双二进制分发

**决策**: zjstatus 和 zjframes 分开编译

**决策理由**:
- zjframes 更轻量
- 用户可按需选择
- 减少不必要的加载

---

## 可复用的提示词模板

```
创建一个 Zellij WASM 状态栏插件：

核心架构：
- Rust WASM 插件，使用 zellij-tile 接口
- Widget Trait 模式：每个组件实现 process() 和 process_click()
- 渲染引擎：自定义 #[fg:red] 格式化字符串 → ANSI 转义序列
- 配置系统：KDL 格式配置 + ZellijState 集中状态管理
- 缓存策略：SizedCache 按配置字符串缓存

Widget 系统设计：
- pub trait Widget { fn process(&self, name: &str, state: &ZellijState) -> String; fn process_click(...); }
- 内置 Widget：datetime, mode, command, session, tabs, notification, pipe, swap_layout
- 配置驱动：每个 Widget 从 BTreeMap<String, String> 读取配置

格式化字符串：
- #[fg:red] 设置前景色
- #[bg:#RRGGBB] 设置背景色（RGB）
- #[effects:bold,italic] 设置效果
- {format}, {date}, {time} 变量占位符

渲染缓存：
- formatted_part_from_string_cached() 缓存单段
- formatted_parts_from_string_cached() 缓存多段
- SizedCache::with_size(100) 限制大小

双二进制：
- zjstatus: 主状态栏（含框架控制）
- zjframes: 独立框架控制（轻量）
```

---

## 扩展点

### 添加新的 Widget

1. 在 `src/widgets/` 创建 `new_widget.rs`
2. 实现 `Widget` trait
3. 在 `src/widgets/mod.rs` 添加 `pub mod new_widget`
4. 在主渲染逻辑中注册

### 添加新的格式化指令

1. 在 `FormattedPart` 添加新字段
2. 在 `from_format_string()` 添加解析逻辑
3. 在 `format_string()` 添加渲染逻辑

### 添加新的条件框架控制

1. 在 `FrameConfig` 添加新选项
2. 在 `hide_frames_conditionally()` 添加条件判断
3. 在 `should_show_frames_*()` 添加判断函数
