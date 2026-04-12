# 架构

> 深入解析 Claude Code 的内部结构。

---

## 概览

Claude Code 是一个面向终端的原生 AI 编程助手，以**单二进制 CLI** 形式交付。架构采用流水线模型：

```
用户输入 → CLI 解析器 → 查询引擎 → LLM API → 工具执行循环 → 终端 UI
```

整个 UI 层基于 **React + Ink**（面向终端的 React）构建，因此它是一个完整的响应式 CLI 应用：组件、钩子、状态管理，以及你在 React Web 应用中熟悉的各种模式——只是渲染目标换成了终端。

---

## 核心流水线

### 1. 入口（`src/main.tsx`）

CLI 解析器使用 [Commander.js](https://github.com/tj/commander.js)（`@commander-js/extra-typings`）构建。启动时会：

- 在加载较重模块之前，并行触发预取副作用（MDM 设置、钥匙串、API 预连接）
- 解析 CLI 参数与标志
- 初始化 React/Ink 渲染器
- 交给 REPL 启动器（`src/replLauncher.tsx`）

### 2. 初始化（`src/entrypoints/`）

| 文件 | 作用 |
|------|------|
| `cli.tsx` | CLI 会话编排——从启动到 REPL 的主路径 |
| `init.ts` | 配置、遥测、OAuth、MDM 策略初始化 |
| `mcp.ts` | MCP 服务端模式入口（Claude Code 作为 MCP 服务端） |
| `sdk/` | Agent SDK——用于嵌入 Claude Code 的程序化 API |

启动时会并行初始化：读取 MDM 策略、预取钥匙串、检查功能开关，然后执行核心初始化。

### 3. 查询引擎（`src/QueryEngine.ts`，约 4.6 万行）

Claude Code 的核心。负责：

- **流式响应**：来自 Anthropic API
- **工具调用循环**：当 LLM 请求工具时执行，并将结果回传
- **思考模式**：扩展思考与预算管理
- **重试逻辑**：对瞬时失败自动退避重试
- **Token 计数**：跟踪每轮输入/输出 token 与成本
- **上下文管理**：管理对话历史与上下文窗口

### 4. 工具系统（`src/Tool.ts` + `src/tools/`）

Claude 能调用的每一项能力都是一个**工具**。每个工具自包含：

- **输入模式**（Zod 校验）
- **权限模型**（哪些需要用户批准）
- **执行逻辑**（实际实现）
- **UI 组件**（调用与结果在终端中的展示方式）

工具在 `src/tools.ts` 中注册，由查询引擎在工具调用循环中发现。

完整目录见 [工具参考](tools.md)。

### 5. 命令系统（`src/commands.ts` + `src/commands/`）

面向用户的斜杠命令（`/commit`、`/review`、`/mcp` 等），可在 REPL 中输入。共三类：

| 类型 | 说明 | 示例 |
|------|------|------|
| **PromptCommand** | 向 LLM 发送带注入工具的格式化提示 | `/review`、`/commit` |
| **LocalCommand** | 进程内执行，返回纯文本 | `/cost`、`/version` |
| **LocalJSXCommand** | 进程内执行，返回 React JSX | `/doctor`、`/install` |

命令在 `src/commands.ts` 中注册，在 REPL 中通过 `/command-name` 调用。

完整目录见 [命令参考](commands.md)。

---

## 状态管理

Claude Code 采用 **React Context + 自定义 store** 模式：

| 组件 | 位置 | 用途 |
|------|------|------|
| `AppState` | `src/state/AppStateStore.ts` | 全局可变状态对象 |
| Context 提供者 | `src/context/` | 通知、统计、FPS 等的 React Context |
| 选择器 | `src/state/` | 派生状态函数 |
| 变更观察者 | `src/state/onChangeAppState.ts` | 状态变更时的副作用 |

`AppState` 对象会传入工具上下文，使工具能访问对话历史、设置与运行时状态。

---

## UI 层

### 组件（`src/components/`，约 140 个组件）

- 使用 Ink 原语（`Box`、`Text`、`useInput()`）的函数式 React 组件
- 使用 [Chalk](https://github.com/chalk/chalk) 做终端配色
- 启用 React Compiler 以优化重渲染
- 设计系统原语位于 `src/components/design-system/`

### 屏幕（`src/screens/`）

全屏 UI 模式：

| 屏幕 | 用途 |
|------|------|
| `REPL.tsx` | 主交互式 REPL（默认屏幕） |
| `Doctor.tsx` | 环境诊断（`/doctor`） |
| `ResumeConversation.tsx` | 会话恢复（`/resume`） |

### 钩子（`src/hooks/`，约 80 个）

标准 React 钩子模式。主要类别包括：

- **权限钩子** — `useCanUseTool`、`src/hooks/toolPermission/`
- **IDE 集成** — `useIDEIntegration`、`useIdeConnectionStatus`、`useDiffInIDE`
- **输入处理** — `useTextInput`、`useVimInput`、`usePasteHandler`、`useInputBuffer`
- **会话管理** — `useSessionBackgrounding`、`useRemoteSession`、`useAssistantHistory`
- **插件/技能钩子** — `useManagePlugins`、`useSkillsChange`
- **通知钩子** — `src/hooks/notifs/`（速率限制、弃用警告等）

---

## 配置与模式

### 配置模式（`src/schemas/`）

基于 Zod v4 的各类配置模式：

- 用户设置
- 项目级设置
- 组织/企业策略
- 权限规则

### 迁移（`src/migrations/`）

处理版本间的配置格式变更——读取旧配置并转换为当前模式。

---

## 构建系统

### Bun 运行时

Claude Code 运行在 [Bun](https://bun.sh) 上（而非 Node.js）。主要影响：

- 原生 JSX/TSX 支持，无需单独转译步骤
- `bun:bundle` 功能开关用于死代码消除
- ES 模块使用 `.js` 扩展名（Bun 约定）

### 功能开关（死代码消除）

```typescript
import { feature } from 'bun:bundle'

// 未激活的功能开关内的代码会在构建时被完全剔除
if (feature('VOICE_MODE')) {
  const voiceCommand = require('./commands/voice/index.js').default
}
```

值得关注的开关：

| 开关 | 功能 |
|------|------|
| `PROACTIVE` | 主动式智能体模式（自主操作） |
| `KAIROS` | Kairos 子系统 |
| `BRIDGE_MODE` | IDE 桥接集成 |
| `DAEMON` | 后台守护进程模式 |
| `VOICE_MODE` | 语音输入/输出 |
| `AGENT_TRIGGERS` | 触发型智能体动作 |
| `MONITOR_TOOL` | 监控工具 |
| `COORDINATOR_MODE` | 多智能体协调器 |
| `WORKFLOW_SCRIPTS` | 工作流自动化脚本 |

### 懒加载

较重模块通过动态 `import()` 延迟到首次使用时再加载：

- OpenTelemetry（约 400KB）
- gRPC（约 700KB）
- 其他可选依赖

---

## 错误处理与遥测

### 遥测（`src/services/analytics/`）

- [GrowthBook](https://www.growthbook.io/)：功能开关与 A/B 测试
- [OpenTelemetry](https://opentelemetry.io/)：分布式追踪与指标
- 自定义事件：用于使用分析

### 成本追踪（`src/cost-tracker.ts`）

跟踪每轮对话的 token 用量与估算费用。可通过 `/cost` 命令查看。

### 诊断（`/doctor` 命令）

`Doctor.tsx` 屏幕会运行环境检查：API 连通性、认证、工具可用性、MCP 服务端状态等。

---

## 并发模型

Claude Code 使用 **单线程事件循环**（Bun/Node.js 模型），配合：

- 异步/await 处理 I/O
- React 并发渲染更新 UI
- Web Worker 或子进程处理 CPU 密集型任务（gRPC 等）
- 工具并发安全——每个工具声明 `isConcurrencySafe()`，表示是否可与其他工具并行运行

---

## 另见

- [工具参考](tools.md) — 全部 40 个智能体工具的完整目录
- [命令参考](commands.md) — 全部斜杠命令的完整目录
- [子系统指南](subsystems.md) — 桥接、MCP、权限、技能、插件等
- [探索指南](exploration-guide.md) — 如何浏览本代码库
