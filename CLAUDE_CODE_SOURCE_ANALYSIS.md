# Claude Code 源码深度分析

> 基于 `@anthropic-ai/claude-code@2.1.88` sourcemap 还原代码分析
> 分析路径：`restored-src/src/`

---

## 目录

1. [项目整体架构](#一项目整体架构)
2. [目录结构总览](#二目录结构总览)
3. [启动流程](#三启动流程)
4. [核心 Agent 循环](#四核心-agent-循环)
5. [工具系统](#五工具系统)
6. [API 层](#六api-层)
7. [UI 渲染层](#七ui-渲染层)
8. [权限与配置系统](#八权限与配置系统)
9. [Hooks 钩子系统](#九hooks-钩子系统)
10. [MCP 集成](#十mcp-集成)
11. [Skills 技能系统](#十一skills-技能系统)
12. [多 Agent / Swarm 系统](#十二多-agent--swarm-系统)
13. [状态管理](#十三状态管理)
14. [记忆与上下文压缩](#十四记忆与上下文压缩)
15. [系统架构图](#十五系统架构图)

---

## 一、项目整体架构

Claude Code 是一个基于命令行的 AI 编程助手，核心技术栈：

| 技术 | 用途 |
|------|------|
| TypeScript + React | 主要开发语言与 UI 框架 |
| [Ink](https://github.com/vadimdemedes/ink) | 基于 React 的 TUI（终端 UI）框架 |
| Zod | Schema 验证与类型生成 |
| `@anthropic-ai/sdk` | Anthropic API 调用 |
| MCP (Model Context Protocol) | 可扩展的工具/资源协议 |

整体上，Claude Code 是一个**多 Agent 协调系统**，以 REPL 为交互入口，通过工具调用让 Claude 完成复杂的软件工程任务。

---

## 二、目录结构总览

```
restored-src/src/
├── entrypoints/          # 各类入口点（CLI、SDK、BYOC等）
├── main.tsx              # CLI 初始化主逻辑
├── query.ts              # ★ 核心 Agent 循环
├── QueryEngine.ts        # SDK 模式下的查询引擎
├── Tool.ts               # 工具接口定义（所有工具的基类契约）
├── tools.ts              # 工具注册中心
├── commands.ts           # Slash 命令注册中心
├── context.ts            # 系统上下文（git 状态、用户信息等）
├── Task.ts               # 后台任务基础类型
│
├── tools/                # ★ 工具实现（40+ 工具）
│   ├── AgentTool/        # 启动子 Agent
│   ├── BashTool/         # 执行 Shell 命令
│   ├── FileReadTool/     # 读取文件
│   ├── FileEditTool/     # 编辑文件
│   ├── FileWriteTool/    # 写入文件
│   ├── GlobTool/         # 文件模式匹配
│   ├── GrepTool/         # 内容搜索
│   ├── WebFetchTool/     # 网页抓取
│   ├── WebSearchTool/    # 网页搜索
│   ├── MCPTool/          # MCP 协议工具代理
│   ├── SkillTool/        # 执行 Skills/slash 命令
│   ├── TodoWriteTool/    # 任务列表管理
│   ├── NotebookEditTool/ # Jupyter Notebook 编辑
│   └── ...               # 其他工具
│
├── commands/             # Slash 命令实现（50+ 命令）
├── services/             # 服务层
│   ├── api/              # Anthropic API 交互
│   ├── mcp/              # MCP 客户端/服务器
│   ├── compact/          # 对话压缩
│   ├── tools/            # 工具编排（并发/串行调度）
│   └── analytics/        # 埋点分析
│
├── screens/              # 顶级屏幕组件
│   ├── REPL.tsx          # ★ 主交互界面（3000+ 行）
│   └── Doctor.tsx        # 诊断工具
│
├── components/           # React/Ink UI 组件（100+ 个）
├── hooks/                # React Hooks（UI 状态管理）
├── state/                # 全局 AppState 管理
├── bootstrap/            # 全局运行时状态
│
├── skills/               # Skills 技能系统
│   ├── bundled/          # 内置技能（commit、loop 等）
│   └── loadSkillsDir.ts  # 技能加载逻辑
│
├── memdir/               # CLAUDE.md 自动记忆系统
├── tasks/                # 后台任务（LocalAgentTask、LocalShellTask）
├── coordinator/          # 多 Agent 协调器模式
├── bridge/               # Bridge/Remote Control 功能
├── remote/               # 远程会话管理
├── migrations/           # 配置迁移脚本
├── utils/                # 工具函数（200+ 模块）
├── types/                # 公共 TypeScript 类型
├── constants/            # 常量（提示词、betas、limits 等）
├── schemas/              # Zod Schema 定义
├── plugins/              # 插件系统
├── vim/                  # Vim 模式支持
└── voice/                # 语音功能
```

---

## 三、启动流程

### 3.1 Bootstrap 入口：`entrypoints/cli.tsx`

这是用户执行 `claude` 命令后第一个执行的文件，采用**快速路径分发**策略：

```
claude 命令
    │
    ├── --version/-v          → 零模块加载，直接打印版本号
    ├── --dump-system-prompt  → 输出系统提示词后退出（调试用）
    ├── --claude-in-chrome-mcp → 启动 Chrome MCP 服务器
    ├── --computer-use-mcp    → 启动计算机使用 MCP 服务器
    ├── --daemon-worker       → 启动后台 Daemon Worker
    ├── remote-control/rc/bridge → 启动 Bridge 模式（远程控制）
    ├── daemon                → 启动长期运行的 Supervisor
    ├── ps/logs/attach/kill   → 会话管理命令
    ├── environment-runner    → BYOC 无头运行器
    ├── self-hosted-runner    → 自托管运行器
    ├── --worktree --tmux     → 快速进入 tmux worktree 模式
    └── （默认）              → 调用 main.tsx 的 main()
```

设计亮点：在快速路径（如 `--version`）中，**不加载任何重型模块**，避免启动延迟。

### 3.2 CLI 初始化：`main.tsx`

`main()` 函数的完整初始化序列：

```
main()
  │
  ├── profileCheckpoint()           # 性能检查点
  ├── startMdmRawRead()             # 并行读取 MDM 企业配置
  │   └── macOS: plutil / Windows: reg query
  ├── startKeychainPrefetch()       # 并行预取认证信息（节省约 65ms）
  │   └── OAuth token + legacy API key
  ├── enableConfigs()               # 启用配置系统（Settings 多层级合并）
  ├── initGrowthBook()              # 初始化特性标志（Feature Flags）
  ├── init()                        # 核心初始化
  │   ├── auth 认证检查
  │   ├── telemetry 遥测初始化
  │   └── permissions 权限初始化
  ├── getMcpToolsCommandsAndResources()  # 连接所有 MCP 服务器
  └── getTools()                    # 组装工具集
        │
        ├── 交互模式 → launchRepl()   → screens/REPL.tsx
        └── 非交互模式 → QueryEngine  （管道/脚本模式）
```

---

## 四、核心 Agent 循环

### 4.1 `query.ts`：Agent 心跳

这是整个系统最核心的文件，导出核心 `query()` 异步生成器函数：

```typescript
export async function* query(
  params: QueryParams,
): AsyncGenerator<StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage, Terminal>
```

### 4.2 `queryLoop()` 内部迭代逻辑

`while(true)` 无限循环，每轮迭代执行以下步骤：

```
queryLoop() 每轮迭代
    │
    ├── 1. Skill Prefetch          预取相关技能（优化响应速度）
    ├── 2. yield stream_request_start  通知 UI 新请求开始
    ├── 3. applyToolResultBudget() 限制工具结果大小
    ├── 4. snipCompactIfNeeded()   裁剪历史（软截断）
    ├── 5. microCompact()          压缩旧工具结果为摘要
    ├── 6. Context Collapse        上下文折叠优化
    ├── 7. autoCompact()           超过 token 阈值时全量压缩
    │
    ├── 8. streamQuery()           ★ 调用 Anthropic API（流式）
    │       └── 收到 token → yield 给 UI 实时渲染
    │
    ├── 9. runTools()              ★ 处理 tool_use blocks
    │       ├── 并发执行只读工具（最多 10 个并发）
    │       └── 串行执行写操作工具
    │
    ├── 10. handleStopHooks()      执行 Stop 类型 Hooks
    └── 11. Budget / Continue 判断  是否继续循环
```

**退出条件（Terminal 状态）**：
- `stop_reason: end_turn` 且无待处理工具调用
- 达到 `max_turns` 限制
- 用户中断（AbortController 信号）
- 不可恢复的 API 错误

### 4.3 `Tool.ts`：工具接口契约

所有工具必须实现的接口：

```typescript
interface Tool<TInput, TOutput> {
  name: string
  description(input: TInput, options): string      // 向 Claude 描述工具用途
  prompt(options): string | undefined              // 贡献系统提示词内容
  inputSchema: ZodSchema<TInput>                   // 输入验证 Schema

  call(args, context, canUseTool, parentMessage, onProgress): Promise<TOutput>
  checkPermissions(input, context): PermissionResult
  validateInput(input, context): ValidationResult

  isReadOnly(input): boolean           // 是否只读（影响并发执行）
  isConcurrencySafe(input): boolean    // 是否可与其他工具并发
  isDestructive(input): boolean        // 是否不可逆操作

  renderToolResultMessage(): JSX.Element  // UI 渲染工具结果
  renderToolUseMessage(): JSX.Element     // UI 渲染工具调用参数
}
```

**`ToolUseContext`** 是工具调用时的上下文对象，包含 40+ 字段：
- `abortController` - 取消控制器
- `options.tools` - 当前可用工具集
- `options.mcpClients` - MCP 客户端列表
- `getAppState() / setAppState()` - React 状态访问
- `messages` - 当前对话历史
- `setToolJSX()` - 向 UI 注入自定义 JSX

### 4.4 `toolOrchestration.ts`：工具并发引擎

```typescript
// 主入口：分批执行工具调用
async function runTools(toolCalls: ToolUse[], context: ToolUseContext): Promise<ToolResult[]>

// 将工具调用分为"可并发批次"和"串行批次"
function partitionToolCalls(calls): { concurrent: ToolUse[], serial: ToolUse[] }

// 并发执行（最多 CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY=10 个并发）
async function runToolsConcurrently(calls, context): Promise<ToolResult[]>

// 串行执行（写操作、不安全操作）
async function runToolsSerially(calls, context): Promise<ToolResult[]>
```

---

## 五、工具系统

### 5.1 工具注册中心：`tools.ts`

`getAllBaseTools()` 返回所有基础工具（按重要性排序）：

| 工具 | 用途 | 特性 |
|------|------|------|
| `AgentTool` | 启动子 Agent | 支持后台运行、隔离 worktree、远程执行 |
| `TaskOutputTool` | 获取后台任务输出 | 与 AgentTool 配合使用 |
| `BashTool` | 执行 Bash 命令 | 权限检查、沙箱隔离、自动后台化 |
| `GlobTool` | 文件模式匹配 | 快速文件查找 |
| `GrepTool` | 内容搜索 | ripgrep 驱动 |
| `FileReadTool` | 读取文件 | 支持图片、PDF、Jupyter Notebook |
| `FileEditTool` | 编辑文件 | 字符串精确替换，防止幻觉 |
| `FileWriteTool` | 写入文件 | 整体覆写 |
| `WebFetchTool` | 网页抓取 | HTML → Markdown 转换 |
| `WebSearchTool` | 网页搜索 | 多搜索引擎支持 |
| `TodoWriteTool` | 任务列表 | 结构化任务跟踪 |
| `MCPTool` | MCP 工具代理 | 动态创建，连接 MCP 服务器 |
| `SkillTool` | 执行技能 | 将 slash 命令作为工具暴露 |
| `LSPTool` | LSP 集成 | 代码诊断与符号分析 |
| `ToolSearchTool` | 工具发现 | 延迟加载（deferred tools）系统 |
| `EnterPlanModeTool` | 进入计划模式 | 只规划不执行 |
| `ExitPlanModeTool` | 退出计划模式 | - |
| `EnterWorktreeTool` | 创建 Git Worktree | 隔离工作环境 |
| `ExitWorktreeTool` | 退出 Worktree | - |
| `ConfigTool` | 配置读写 | 支持多层级配置 |
| `AskUserQuestionTool` | 向用户提问 | 支持单选/多选/预览 |
| `NotebookEditTool` | Notebook 编辑 | Jupyter cell 操作 |
| 条件工具 | `REPLTool`、`SleepTool`、`SendMessageTool` 等 | 通过 Feature Flags 控制 |

### 5.2 AgentTool 详解

AgentTool 是最复杂的工具，负责启动子 Agent（实现递归调用）。

**输入 Schema**：

```typescript
{
  description: string    // 任务描述（3-5 词）
  prompt: string         // 完整任务提示词
  subagent_type?: string // 指定专业 Agent 类型（来自 ~/.claude/agents/）
  model?: 'sonnet' | 'opus' | 'haiku'  // 模型覆盖
  run_in_background?: boolean  // 后台异步运行
  isolation?: 'worktree' | 'remote'  // 隔离模式
  cwd?: string           // 自定义工作目录
}
```

**执行路径**：

```
AgentTool.call()
    │
    ├── getDenyRuleForAgent()        检查是否被 deny 规则拦截
    ├── checkRemoteAgentEligibility() 决定是否远程运行（CCR 环境）
    ├── createSubagentContext()       克隆父级 ToolUseContext
    │   └── 共享 readFileState 缓存（优化 prompt cache 命中率）
    ├── loadAgentsDir()               加载 Agent 定义文件
    │
    ├── run_in_background=true  → runAsyncAgentLifecycle()
    │                              └── 注册到 LocalAgentTask（AppState.tasks）
    └── run_in_background=false → runAgent()
                                   └── 递归调用 query()
```

### 5.3 BashTool 详解

```typescript
// 权限检查
function bashToolHasPermission(command: string, rules: PermissionRules): PermissionResult

// 安全处理 sed -i 命令（转换为文件编辑操作）
function parseSedEditCommand(command: string): FileEdit | null

// 沙箱判断
function shouldUseSandbox(context: ToolUseContext): boolean

// 自动后台化（超过 15 秒的命令）
// 进度显示（超过 2 秒显示 spinner）
```

### 5.4 延迟加载（Deferred Tools）系统

当工具总数超过阈值（通常 20 个），部分工具标记为 `shouldDefer: true`：

- **初始系统提示**中不包含这些工具的完整 schema
- Claude 需先调用 `ToolSearch` 工具发现并激活这些工具
- 设计目的：减少 token 消耗，避免 Claude 因工具过多而困惑

```typescript
// 工具发现工具
ToolSearchTool.call({
  query: "关键词搜索" | "select:<tool_name>"  // 直接选择
})
```

---

## 六、API 层

### 6.1 `services/api/claude.ts`

与 Anthropic API 交互的核心模块（约 1000+ 行）：

```typescript
// 核心函数：流式 API 调用
async function* streamQuery(
  messages: Message[],
  tools: Tool[],
  systemPrompt: string,
  options: StreamQueryOptions,
): AsyncGenerator<StreamEvent>

// 将 Tool 对象转换为 API 格式
function toolToAPISchema(tool: Tool): BetaToolUnion

// Token 用量统计
function accumulateUsage(existing: Usage, delta: Usage): Usage
```

**Beta Headers 控制**（实验性功能通过 HTTP 头激活）：

| Header | 功能 |
|--------|------|
| `prompt-caching-2024-07-31` | 系统提示词缓存（节省 token） |
| `interleaved-thinking-2025-05-14` | 交错思考模式 |
| `tool-result-preamble-2025-05-14` | 工具结果前言 |
| `fast-mode-2025-06-05` | 快速输出模式 |
| `effort-2025-05-14` | 思考力度控制 |
| `task-budgets-2026-03-13` | 任务 token 预算 |

**Prompt Cache 管理**：
- 支持 1 小时 TTL 的长效缓存（`PROMPT_CACHING_SCOPE_BETA_HEADER`）
- 自动检测缓存断裂，必要时重建缓存标记

### 6.2 `services/api/client.ts`

支持多种 API 后端：

```typescript
// 创建 Anthropic SDK 客户端，支持：
- 直连 Anthropic API（api.anthropic.com）
- AWS Bedrock
- Google Cloud Vertex AI
- 自定义 baseURL（企业代理/内部网关）
```

---

## 七、UI 渲染层

### 7.1 `screens/REPL.tsx`

主 TUI 屏幕，约 3000+ 行，使用 Ink（基于 React 的 CLI UI 框架）。

**主要职责**：
- 渲染消息列表（`VirtualMessageList` - 虚拟化列表，优化大量消息渲染）
- 渲染输入框（`PromptInput` - 支持多行输入、历史记录、vim 模式）
- 处理全局键绑定（`GlobalKeybindingHandlers`、`CommandKeybindingHandlers`）
- 管理权限请求 UI（`PermissionRequest`、`ElicitationDialog`）
- 工具进度显示（spinner、后台任务导航）
- Swarm/teammate 视图

**关键 React Hooks**：

| Hook | 用途 |
|------|------|
| `useCanUseTool` | 工具权限检查 |
| `useAssistantHistory` | 对话历史记录管理 |
| `useCommandQueue` | 命令队列处理 |
| `useQueueProcessor` | 用户输入队列处理 |
| `useSwarmInitialization` | 多 Agent Swarm 初始化 |
| `useReplBridge` | Bridge/远程控制集成 |
| `useVimMode` | Vim 模式支持 |

### 7.2 组件体系

```
screens/REPL.tsx
    ├── components/VirtualMessageList/   虚拟化消息列表
    │   ├── AssistantMessage/            Claude 的回复（含 streaming）
    │   ├── ToolUseMessage/              工具调用展示
    │   ├── ToolResultMessage/           工具结果展示
    │   └── UserMessage/                 用户输入展示
    ├── components/PromptInput/          用户输入框
    │   ├── vim mode support
    │   └── history navigation
    ├── components/PermissionRequest/    权限请求对话框
    ├── components/ElicitationDialog/    结构化信息采集
    ├── components/TaskProgress/         后台任务进度
    └── components/StatusBar/            底部状态栏
```

---

## 八、权限与配置系统

### 8.1 权限模式（`PermissionMode`）

```typescript
type PermissionMode =
  | 'default'           // 标准：危险操作需要用户批准
  | 'acceptEdits'       // 自动接受文件编辑
  | 'bypassPermissions' // 绕过所有权限（--dangerously-skip-permissions）
  | 'plan'              // 计划模式：只规划不执行
  | 'auto'              // 自动模式：AI 分类器决定
```

### 8.2 权限检查流程

```
hasPermissionsToUseTool(tool, input, context)
    │
    ├── 1. 检查 Deny 规则      → 匹配 → 拒绝（立即返回）
    ├── 2. 检查 Always-Allow   → 匹配 → 允许（无需弹窗）
    ├── 3. 检查 Allow 规则     → 匹配 → 允许
    ├── 4. 检查权限模式
    │   ├── bypassPermissions  → 允许所有
    │   └── plan               → 拒绝所有写操作
    └── 5. 无规则匹配          → 弹出 PermissionRequest UI
```

权限规则示例（`settings.json`）：

```json
{
  "permissions": {
    "allow": ["Bash(git *)", "Read(*)", "Bash(npm run *)"],
    "deny": ["Bash(rm -rf *)", "Bash(sudo *)"],
    "defaultMode": "acceptEdits"
  }
}
```

### 8.3 Settings 多层级配置

优先级从高到低：

```
policySettings    企业 MDM 管理策略（最高优先级）
    ↓
enterprise        企业配置文件（claude.json）
    ↓
user              用户全局配置（~/.claude/settings.json）
    ↓
project           项目配置（.claude/settings.json）
    ↓
local             本地配置（.claude/settings.local.json）
```

主要配置项：

```typescript
interface Settings {
  permissions: {
    allow: string[]
    deny: string[]
    ask: string[]
    defaultMode: PermissionMode
  }
  hooks: HookConfig[]          // 事件钩子
  env: Record<string, string>  // 注入环境变量
  mcpServers: MCPServerConfig  // MCP 服务器配置
  model: string                // 默认模型
}
```

---

## 九、Hooks 钩子系统

### 9.1 支持的 Hook 事件

```typescript
const HOOK_EVENTS = [
  // 工具相关
  'PreToolUse',         // 工具调用前
  'PostToolUse',        // 工具调用成功后
  'PostToolUseFailure', // 工具调用失败后

  // 会话生命周期
  'SessionStart',       // 会话开始
  'SessionEnd',         // 会话结束

  // Agent 生命周期
  'Stop',               // Claude 停止输出
  'StopFailure',        // 停止失败
  'SubagentStart',      // 子 Agent 启动
  'SubagentStop',       // 子 Agent 停止

  // 上下文管理
  'PreCompact',         // 压缩前
  'PostCompact',        // 压缩后
  'InstructionsLoaded', // 指令加载完成
  'CwdChanged',         // 工作目录变更
  'FileChanged',        // 文件变更

  // 权限相关
  'PermissionRequest',  // 权限申请
  'PermissionDenied',   // 权限被拒

  // 任务相关
  'TaskCreated',        // 任务创建
  'TaskCompleted',      // 任务完成

  // 用户交互
  'UserPromptSubmit',   // 用户提交输入
  'Notification',       // 通知
  'Elicitation',        // 信息采集请求

  // Worktree 相关
  'WorktreeCreate',     // Worktree 创建
  'WorktreeRemove',     // Worktree 删除

  // 其他
  'ConfigChange',       // 配置变更
  'Setup',              // 初始化
  'TeammateIdle',       // 协作者空闲
]
```

### 9.2 Hook 类型

```typescript
type HookCommand =
  // 执行 Shell 命令
  | { type: 'command'; command: string; shell?: string; if?: string }

  // 向模型发送提示词
  | { type: 'prompt'; prompt: string }

  // 启动子 Agent 执行 hook 逻辑
  | { type: 'agent'; prompt: string }

  // 发起 HTTP 请求（可用于 Webhook）
  | { type: 'http'; url: string; method?: string; headers?: Record<string, string> }
```

### 9.3 Hook 配置示例

```json
{
  "hooks": [
    {
      "event": "PreToolUse",
      "matcher": "Bash(git *)",
      "config": {
        "type": "command",
        "command": "echo 'About to run git command'"
      }
    },
    {
      "event": "PostToolUse",
      "config": {
        "type": "http",
        "url": "https://my-webhook.example.com/tool-used"
      }
    },
    {
      "event": "Stop",
      "config": {
        "type": "command",
        "command": "osascript -e 'display notification \"Claude finished\"'"
      }
    }
  ]
}
```

---

## 十、MCP 集成

### 10.1 MCP 架构

MCP（Model Context Protocol）是 Anthropic 开放的标准协议，允许外部服务器向 Claude 提供工具、资源和提示词。

```
Claude Code
    │
    ├── MCP Client ──── stdio ──── 本地进程（如 filesystem-mcp）
    ├── MCP Client ──── SSE  ──── 远程 HTTP 服务器
    ├── MCP Client ──── WS   ──── WebSocket 服务器
    └── MCP Client ──── SDK  ──── In-process 模式
```

### 10.2 MCP 服务器配置

在 `~/.claude.json` 或 `.claude.json` 中配置：

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
    },
    "remote-api": {
      "type": "sse",
      "url": "https://api.example.com/mcp",
      "headers": { "Authorization": "Bearer TOKEN" },
      "oauth": { "clientId": "...", "authorizationUrl": "..." }
    }
  }
}
```

### 10.3 MCP 工具动态创建

`fetchToolsForClient()` 从 MCP 服务器获取工具列表，并动态创建 Tool 实例：

```typescript
// 以 MCPTool 为模板，覆盖以下方法：
{
  name: `mcp__${serverName}__${toolName}`,
  description: () => tool.description,
  inputSchema: zodSchemaFromJsonSchema(tool.inputSchema),
  call: async (args) => mcpClient.callTool(tool.name, args),
}
```

---

## 十一、Skills 技能系统

### 11.1 技能来源

| 来源 | 路径 | 说明 |
|------|------|------|
| `skills` | `~/.claude/skills/*.md` | 用户自定义技能（新格式） |
| `commands_DEPRECATED` | `.claude/commands/*.md` | 旧版命令格式 |
| `bundled` | `src/skills/bundled/` | 内置技能 |
| `plugin` | 插件目录 | 插件提供的技能 |
| `managed` | 企业配置 | 企业管理员推送的技能 |
| `mcp` | MCP 服务器 | MCP server 提供的 prompts |

### 11.2 技能文件格式

```markdown
---
description: 创建 git commit 的完整工作流
model: sonnet
tools: [Bash, Read, FileEdit]
hooks:
  PreToolUse:
    - type: command
      command: echo "before tool use"
---

# Commit 技能

请执行以下步骤来创建一个好的 git commit：

1. 运行 `git status` 查看变更
2. 运行 `git diff` 了解具体改动
...（发给 Claude 的完整指令）
```

### 11.3 内置技能列表

| 技能 | 文件 | 功能 |
|------|------|------|
| `/commit` | `commit.ts` | Git commit 完整工作流 |
| `/batch` | `batch.ts` | 批量操作执行 |
| `/loop` | `loop.ts` | 定时循环执行任务 |
| `/remember` | `remember.ts` | 跨会话记忆存储 |
| `/schedule-remote-agents` | `scheduleRemoteAgents.ts` | 调度远程 Agent |
| `/skillify` | `skillify.ts` | 将对话转为技能文件 |
| `/verify` | `verify.ts` | 验证工具执行结果 |
| `/claude-api` | `claudeApi.ts` | 内嵌 Claude API 调用技能 |

---

## 十二、多 Agent / Swarm 系统

### 12.1 子 Agent 执行流程

```
AgentTool.call()
    │
    ├── loadAgentsDir()
    │   └── 从 ~/.claude/agents/ 和 .claude/agents/ 加载 Agent 定义
    │
    ├── createSubagentContext()
    │   ├── 克隆父级 ToolUseContext
    │   └── 共享 readFileState（优化 prompt cache 命中率）
    │
    ├── 初始化 Agent 专属 MCP 服务器（frontmatter 中配置）
    │
    └── runAgent()
        └── 递归调用 query()  （独立的 Agent 循环）
```

### 12.2 Agent 定义文件

文件路径：`~/.claude/agents/<agent-name>.md` 或 `.claude/agents/<agent-name>.md`

```markdown
---
description: 专门用于代码审查的 Agent
tools: [Read, Glob, Grep]
model: opus
color: blue
mcpServers:
  - github-mcp
---

你是一个专业的代码审查专家，请按照以下标准进行审查...
```

### 12.3 LocalAgentTask（后台任务管理）

```typescript
interface LocalAgentTask {
  id: string
  status: 'running' | 'completed' | 'failed'
  progress: ProgressTracker  // token 用量 + 近期工具活动
  result?: AgentResult
  error?: Error
}

// 注册到 AppState.tasks，在 REPL UI 中可见和导航
```

### 12.4 内置 Agent 类型

通过 `subagent_type` 参数指定：

- `general-purpose`：通用助手（默认，拥有所有工具）
- `Explore`：代码库探索专家（只读工具）
- `Plan`：架构设计专家（只读工具 + 规划）
- `claude-code-guide`：Claude Code 使用指南专家

---

## 十三、状态管理

### 13.1 全局运行时状态：`bootstrap/state.ts`

```typescript
interface State {
  // 会话信息
  sessionId: string
  originalCwd: string
  projectRoot: string

  // 成本跟踪
  totalCostUSD: number
  totalAPIDuration: number
  modelUsage: Record<string, TokenUsage>

  // 模型配置
  mainLoopModelOverride?: string
  modelStrings: ModelStrings

  // Token 预算
  currentTurnTokenBudget?: number
  turnOutputTokens: number

  // 已注册的 SDK 回调 Hooks
  registeredHooks: HookRegistration[]
}
```

### 13.2 AppState（React 状态）：`state/AppStateStore.ts`

```typescript
interface AppState {
  // 权限上下文
  toolPermissionContext: PermissionContext

  // 后台任务列表
  tasks: Map<string, LocalAgentTask | LocalShellTask>

  // MCP 状态
  mcp: {
    servers: MCPServer[]
    commands: MCPCommand[]
    resources: MCPResource[]
  }

  // 对话历史（主线程）
  messages: Message[]

  // UI 推测状态
  speculations: Speculation[]
}
```

---

## 十四、记忆与上下文压缩

### 14.1 CLAUDE.md 记忆系统

```
项目目录
    └── .claude/
        └── CLAUDE.md         项目级指令（最高优先级）

用户主目录
    └── ~/.claude/
        ├── CLAUDE.md          全局指令
        └── memory/            自动记忆目录
            ├── MEMORY.md      主记忆文件（自动注入上下文）
            └── *.md           主题记忆文件
```

`getClaudeMds()` 扫描当前目录树，找到所有 `CLAUDE.md` 文件并注入系统提示词。

### 14.2 上下文压缩系统

当对话 token 数超过阈值时触发：

```
Token 使用量
    │
    ├── 轻量压缩（Microcompact）
    │   └── 将旧工具结果替换为摘要文本
    │       （保留结构，减少 token）
    │
    ├── 自动全量压缩（Autocompact）
    │   ├── 触发条件：超过最大 token 阈值
    │   └── 流程：
    │       ├── runCompact() 启动临时子 Agent
    │       ├── 子 Agent 对历史进行摘要
    │       └── buildPostCompactMessages() 重组为对话列表
    │
    └── 手动压缩（/compact 命令）
        └── 用户主动触发全量压缩
```

---

## 十五、系统架构图

```
┌──────────────────────────────────────────────────────────┐
│                     用户输入                              │
└───────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│           entrypoints/cli.tsx (Bootstrap)                 │
│  快速路径分发：version / mcp-server / daemon / main       │
└───────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│    main.tsx (main())  初始化序列                          │
│    MDM读取 → Keychain预取 → Settings → GrowthBook        │
│    → auth/telemetry/permissions → MCP连接 → 工具组装     │
└───────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│         screens/REPL.tsx (Ink TUI)                        │
│    用户输入 → processUserInput() → 提交到查询队列         │
└───────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│              query.ts (queryLoop)  ★核心★                │
│                                                           │
│   ┌─────────────────────────────────────────────────┐   │
│   │  while(true)                                     │   │
│   │    ├── 上下文管理（microcompact/autocompact）    │   │
│   │    ├── streamQuery() ──→ Anthropic API           │   │
│   │    │      └── 流式返回 tokens + tool_use blocks  │   │
│   │    ├── runTools() ──→ 工具并发/串行执行           │   │
│   │    ├── handleStopHooks()                        │   │
│   │    └── Terminal 检查（end_turn/中断/错误）       │   │
│   └─────────────────────────────────────────────────┘   │
└──────────────┬────────────────────────┬──────────────────┘
               │                        │
┌──────────────▼──────────┐  ┌──────────▼──────────────────┐
│  services/api/claude.ts  │  │  工具执行层                  │
│  ├── streamQuery()       │  │  ├── BashTool → exec()      │
│  ├── prompt caching      │  │  ├── FileEditTool → write() │
│  ├── beta headers        │  │  ├── AgentTool              │
│  └── token accounting    │  │  │   └── 递归 query()       │
│                          │  │  ├── MCPTool → MCP server  │
│  支持：                   │  │  └── SkillTool             │
│  - Anthropic API         │  │      └── runAgent()         │
│  - AWS Bedrock           │  └─────────────────────────────┘
│  - Google Vertex AI      │
│  - 企业代理              │
└──────────────────────────┘

权限系统：hasPermissionsToUseTool()
    ├── Deny rule match → 拒绝
    ├── Allow rule match → 允许
    └── No match → PermissionRequest UI

Hooks 系统：贯穿整个调用链
    ├── PreToolUse / PostToolUse
    ├── UserPromptSubmit
    ├── SessionStart / SessionEnd
    └── Stop / SubagentStart / ...

MCP 系统：扩展外部工具/资源
    ├── stdio（本地进程）
    ├── SSE/HTTP（远程服务器）
    └── WebSocket
```

---

## 十六、设计亮点总结

### 16.1 性能优化

| 优化策略 | 实现 |
|---------|------|
| 快速路径分发 | `cli.tsx` 中对简单命令零模块加载 |
| 并行预初始化 | MDM 读取和 Keychain 预取并行执行 |
| Prompt Cache | 系统提示词 1 小时 TTL 缓存，大幅降低 token 成本 |
| 工具并发执行 | 只读工具最多 10 个并发执行 |
| 虚拟化列表 | `VirtualMessageList` 处理大量消息不卡顿 |
| Skill Prefetch | 在 API 调用前预取相关技能 |
| readFileState 共享 | 子 Agent 共享文件读取缓存，提升 prompt cache 命中率 |

### 16.2 安全设计

| 安全特性 | 实现 |
|---------|------|
| 多层权限规则 | Allow/Deny/Ask 三级权限 + 通配符匹配 |
| 权限模式 | default/acceptEdits/bypassPermissions/plan/auto |
| 企业策略覆盖 | MDM/policySettings 优先级最高，无法被用户覆盖 |
| 沙箱隔离 | BashTool 支持沙箱模式 |
| Worktree 隔离 | Agent 可在独立 git worktree 中执行 |
| sed 安全处理 | `parseSedEditCommand()` 防止命令注入 |

### 16.3 扩展性设计

| 扩展点 | 机制 |
|-------|------|
| 工具扩展 | 实现 Tool 接口，注册到 `getAllBaseTools()` |
| 技能扩展 | 在 `~/.claude/skills/` 添加 `.md` 文件 |
| Agent 定制 | 在 `~/.claude/agents/` 定义专业 Agent |
| MCP 服务器 | 配置任意 MCP 兼容服务器 |
| Hooks 扩展 | 监听 20+ 事件，执行命令/提示词/HTTP 请求 |
| 插件系统 | `plugins/` 目录支持插件化扩展 |
| Feature Flags | GrowthBook 控制实验性功能的渐进发布 |

---

*文档生成时间：2026-04-02*
*源码版本：@anthropic-ai/claude-code@2.1.88*
