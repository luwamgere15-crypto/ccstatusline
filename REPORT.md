# ccstatusline 实现原理分析报告

## 1. 简介
`ccstatusline` 是一个用于 Claude Code CLI 的高度可定制状态栏格式化工具。它允许用户在终端底部显示模型信息、Git 状态、Token 使用情况、会话时长等多种指标，并支持 Powerline 风格的渲染。

## 2. 核心架构
该工具基于 TypeScript 开发，使用 Node.js/Bun 运行时。其核心架构可以分为以下几个部分：

### 2.1 双模式运行
入口文件 `src/ccstatusline.ts` 通过检查 `process.stdin.isTTY` 来决定运行模式：
- **交互配置模式 (Interactive Mode)**: 当 `isTTY` 为 `true` 时，表示用户直接运行命令。此时启动基于 React 和 Ink 的 TUI (Text User Interface)，允许用户交互式地配置状态栏。
- **渲染模式 (Render Mode)**: 当 `isTTY` 为 `false` 时，表示工具正通过管道接收数据（通常由 Claude Code 调用）。此时工具读取标准输入 (stdin) 的 JSON 数据，解析后进行渲染并输出到标准输出 (stdout)。

### 2.2 模块化 Widget 系统
工具采用了高度模块化的设计，所有的状态栏元素都被定义为 Widget。
- **接口定义**: `src/types/Widget.ts` 定义了 `Widget` 接口，要求实现 `render`（渲染逻辑）、`getEditorDisplay`（编辑器显示）等方法。
- **实现**: `src/widgets/` 目录下包含了各种 Widget 的具体实现（如 `ModelWidget`, `GitBranchWidget`, `SessionClockWidget` 等）。
- **注册**: Widget 通过注册机制被加载，渲染引擎根据配置动态调用相应的 Widget 进行渲染。

## 3. 数据流与指标采集
`ccstatusline` 的数据来源主要有两个途径：

### 3.1 标准输入 (Stdin)
Claude Code 在调用状态栏工具时，会将当前状态以 JSON 格式传递给工具的标准输入。包含的数据包括：
- 模型信息 (`model`)
- Git 状态 (`git`)
- 当前工作目录
- Transcript 路径 (`transcript_path`)
- 其他元数据

### 3.2 Transcript 日志解析
为了提供比标准输入更丰富的信息（如精确的 Token 消耗、会话时长、Block 进度），工具会主动读取 Claude Code 生成的日志文件。
- **Token 统计**: `src/utils/jsonl.ts` 中的 `getTokenMetrics` 函数会读取 `transcript_path` 指向的 JSONL 文件，累加 `input_tokens` 和 `output_tokens`。
- **Block 计时**: `getBlockMetrics` 函数会扫描 Claude 配置目录下的所有 JSONL 文件，通过文件修改时间和内容分析，智能重建 5 小时的 Block 会话周期，从而计算当前 Block 的剩余时间。

## 4. 渲染引擎 (Renderer)
渲染引擎 (`src/utils/renderer.ts`) 是该工具最复杂的部分，负责将各个 Widget 的输出组装成最终的字符串。主要特性包括：

### 4.1 Powerline 渲染
在 Powerline 模式下，渲染器需要处理特殊的箭头分隔符和颜色过渡：
- **颜色推导**: 分隔符的颜色通常取决于前一个 Widget 的背景色和后一个 Widget 的背景色。
- **逻辑**:
    - **前景色**: 设置为前一个 Widget 的背景色。
    - **背景色**: 设置为后一个 Widget 的背景色。
    - **反转**: 支持反转模式，用于某些特定的视觉效果。

### 4.2 Flex 布局
支持类似 CSS Flexbox 的布局逻辑：
- **Flex Separator**: 用户可以插入 `flex-separator` 组件。
- **空间计算**: 渲染器首先计算所有固定内容的长度，然后计算终端剩余宽度。
- **自动填充**: 剩余宽度被均匀分配给所有的 `flex-separator`，从而实现左对齐、右对齐或居中对齐的效果。

### 4.3 智能截断与填充
- **宽度检测**: 自动获取终端宽度（`process.stdout.columns`），并根据配置预留空间（如为 Claude 的自动压缩提示预留 40 字符）。
- **截断**: 当内容超出可用宽度时，会自动截断并添加省略号，同时确保不会截断 ANSI 转义序列导致乱码。
- **填充**: 支持为每个 Widget 添加统一的左右 Padding，并处理相邻 Widget 合并时的 Padding 消除逻辑。

## 5. 配置管理
配置存储在 `~/.config/ccstatusline/settings.json` 中，使用 Zod (`src/types/Settings.ts`) 进行 Schema 定义和校验。配置项包括：
- `lines`: 多行状态栏配置，每行包含一组 Widget。
- `powerline`: Powerline 相关配置（主题、分隔符、Cap 等）。
- `flexMode`: 布局模式（全宽、预留空间等）。
- `colorLevel`: 颜色支持级别（16色、256色、TrueColor）。

## 6. 总结
`ccstatusline` 通过拦截 Claude Code 的状态输出，结合对日志文件的深度解析，利用强大的渲染引擎实现了高度可定制的终端状态栏。其架构清晰地分离了数据获取、配置管理和界面渲染，使得扩展新的 Widget 变得非常容易。
