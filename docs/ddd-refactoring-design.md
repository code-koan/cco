# Claude Code DDD 改造设计文档

> 版本：1.0
> 日期：2026-04-01
> 状态：草案

---

## 目录

1. [执行摘要](#一执行摘要)
2. [现有架构分析](#二现有架构分析)
3. [DDD 限界上下文划分](#三ddd-限界上下文划分)
4. [各限界上下文详细设计](#四各限界上下文详细设计)
5. [跨上下文集成设计](#五跨上下文集成设计)
6. [共享内核设计](#六共享内核设计)
7. [领域事件设计](#七领域事件设计)
8. [重构路线图](#八重构路线图)
9. [设计决策记录](#九设计决策记录)
10. [风险评估与缓解措施](#十风险评估与缓解措施)
11. [附录](#十一附录)

---

## 一、执行摘要

### 1.1 项目背景

Claude Code 是 Anthropic 开发的 CLI 工具，用于在终端中与 Claude 交互执行软件工程任务。本项目基于 2026-03-31 公开暴露的源码快照进行架构分析和 DDD 改造设计。

### 1.2 现有问题

| 问题类型 | 具体表现 | 影响 |
|---------|---------|------|
| 巨型文件 | QueryEngine.ts (~46K 行)、Tool.ts (~29K 行)、commands.ts (~25K 行) | 难以理解、维护、测试 |
| 职责混乱 | Tool.ts 混合类型定义、权限上下文、进度状态 | 违反单一职责原则 |
| 循环依赖 | tools.ts ↔ TeamCreateTool/TeamDeleteTool | 编译风险、测试困难 |
| 状态分散 | AppState、FileStateCache、PermissionContext 等多处定义 | 状态追踪困难 |
| 条件编译散布 | `feature('XXX')` 散布在各处 | 代码可读性差 |

### 1.3 改造目标

1. **可维护性**：将巨型文件拆分为职责清晰的小模块
2. **可测试性**：领域逻辑独立于技术实现，便于单元测试
3. **可扩展性**：通过限界上下文隔离，支持独立演进
4. **可理解性**：统一语言、清晰的聚合边界

### 1.4 预期收益

- 代码行数：单个文件控制在 800 行以内
- 测试覆盖：领域层测试覆盖率达到 90%+
- 编译时间：减少循环依赖，提升编译速度
- 团队协作：各上下文可独立开发

---

## 二、现有架构分析

### 2.1 项目概况

```
项目规模：~1,900 文件，512,000+ 行代码
技术栈：
  - 运行时：Bun
  - 语言：TypeScript (strict)
  - 终端 UI：React + Ink
  - CLI 解析：Commander.js
  - Schema 验证：Zod v4
  - 代码搜索：ripgrep
```

### 2.2 目录结构分析

```
src/
├── main.tsx                 # 入口编排 (803KB, 巨型文件)
├── commands.ts              # 命令注册 (~25K 行)
├── tools.ts                 # 工具注册 (~17K 行)
├── Tool.ts                  # 工具类型定义 (~29K 行)
├── QueryEngine.ts           # LLM 查询引擎 (~46K 行)
├── context.ts               # 系统/用户上下文收集
├── cost-tracker.ts          # Token 成本追踪
│
├── commands/                # 斜杠命令实现 (~50 个命令)
│   ├── commit.ts
│   ├── review/
│   ├── compact/
│   ├── config/
│   └── ...
│
├── tools/                   # Agent 工具实现 (~40 个工具)
│   ├── AgentTool/           # 子代理工具
│   ├── BashTool/            # Shell 命令执行
│   ├── FileReadTool/        # 文件读取
│   ├── FileWriteTool/       # 文件写入
│   ├── FileEditTool/        # 文件编辑
│   ├── GlobTool/            # 文件模式搜索
│   ├── GrepTool/            # 内容搜索
│   ├── WebFetchTool/        # URL 获取
│   ├── WebSearchTool/       # Web 搜索
│   ├── SkillTool/           # 技能执行
│   ├── MCPTool/             # MCP 工具调用
│   ├── LSPTool/             # LSP 集成
│   └── ...
│
├── components/              # Ink UI 组件 (~140 个)
│   ├── MessageSelector.tsx
│   ├── Spinner.tsx
│   └── ...
│
├── hooks/                   # React hooks
│   ├── useCanUseTool.ts
│   └── ...
│
├── services/                # 外部服务集成
│   ├── api/                 # Anthropic API 客户端
│   │   ├── claude.ts        # 核心 API 调用
│   │   ├── errors.ts        # 错误处理
│   │   └── logging.ts       # 日志记录
│   ├── mcp/                 # MCP 服务
│   │   ├── types.ts         # MCP 类型定义
│   │   └── ...
│   ├── analytics/           # 分析服务
│   ├── oauth/               # OAuth 认证
│   ├── lsp/                 # LSP 管理
│   ├── compact/             # 上下文压缩
│   └── ...
│
├── state/                   # 状态管理
│   ├── AppState.tsx         # React 状态提供者
│   ├── AppStateStore.ts     # 状态存储定义
│   └── store.ts             # 状态创建
│
├── bridge/                  # IDE 和远程控制桥接
│   ├── bridgeMain.ts        # 桥接主循环
│   ├── bridgeMessaging.ts   # 消息协议
│   ├── replBridge.ts        # REPL 会话桥接
│   └── ...
│
├── coordinator/             # 多 Agent 协调器
├── plugins/                 # 插件系统
├── skills/                  # 技能系统
├── memdir/                  # 持久化内存目录
├── tasks/                   # 任务管理
├── types/                   # TypeScript 类型定义
├── utils/                   # 工具函数
└── ...
```

### 2.3 核心模块分析

#### 2.3.1 QueryEngine.ts 分析

**文件规模**：~46,000 行

**核心职责**：
1. LLM API 调用生命周期管理
2. 流式响应处理
3. 消息状态管理
4. 工具调用循环
5. Token 使用追踪
6. 权限检查集成
7. 会话持久化

**关键类型定义**：

```typescript
// QueryEngine 配置
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  // ... 更多配置项
}

// QueryEngine 类
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private readFileState: FileStateCache
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()

  async *submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean },
  ): AsyncGenerator<SDKMessage, void, unknown> {
    // ~2000 行的实现
  }
}
```

**问题分析**：
- 单一类承担过多职责
- 状态管理分散在多个私有字段
- 与 UI 层、API 层、权限层紧耦合
- 难以单独测试各个职责

#### 2.3.2 Tool.ts 分析

**文件规模**：~29,000 行

**核心职责**：
1. 工具类型定义
2. 工具接口定义
3. 权限上下文定义
4. 进度状态类型
5. 工具构建器

**关键类型定义**：

```typescript
// 工具权限上下文
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>

// 工具使用上下文
export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools
    verbose: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
    querySource?: QuerySource
    refreshTools?: () => Tools
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  // ... 50+ 字段
}

// 工具接口
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  name: string
  aliases?: string[]
  searchHint?: string
  inputSchema: Input
  inputJSONSchema?: ToolInputJSONSchema
  outputSchema?: z.ZodType<unknown>
  maxResultSizeChars: number

  // 核心方法
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  description(input, options): Promise<string>

  // 权限方法
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>

  // 状态方法
  isEnabled(): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isConcurrencySafe(input): boolean

  // UI 方法
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode
  // ... 更多 UI 方法
}
```

**问题分析**：
- 类型定义与业务逻辑混杂
- ToolUseContext 包含 50+ 字段，职责不清
- UI 渲染方法与核心逻辑混合
- 权限检查逻辑分散

#### 2.3.3 tools.ts 分析

**文件规模**：~17,000 行

**核心职责**：
1. 工具注册表
2. 工具预设管理
3. 条件编译逻辑

**关键代码**：

```typescript
// 条件编译示例
const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null

const cronTools = feature('AGENT_TRIGGERS')
  ? [
      require('./tools/ScheduleCronTool/CronCreateTool.js').CronCreateTool,
      require('./tools/ScheduleCronTool/CronDeleteTool.js').CronDeleteTool,
      require('./tools/ScheduleCronTool/CronListTool.js').CronListTool,
    ]
  : []

// 循环依赖解决
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
const getTeamDeleteTool = () =>
  require('./tools/TeamDeleteTool/TeamDeleteTool.js').TeamDeleteTool
```

**问题分析**：
- 条件编译逻辑散布
- 循环依赖需要延迟加载
- 工具注册逻辑与配置混合

#### 2.3.4 AppState 分析

**文件**：`src/state/AppStateStore.ts`

**核心结构**：

```typescript
export type AppState = DeepImmutable<{
  // 基础设置
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting

  // UI 状态
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  footerSelection: FooterItem | null

  // 权限上下文
  toolPermissionContext: ToolPermissionContext

  // Agent 相关
  agent: string | undefined
  kairosEnabled: boolean
  agentNameRegistry: Map<string, AgentId>
  foregroundedTaskId?: string
  viewingAgentTaskId?: string

  // MCP 相关
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }

  // 插件相关
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: {...}
    needsRefresh: boolean
  }

  // 任务相关
  tasks: { [taskId: string]: TaskState }

  // Bridge 相关
  replBridgeEnabled: boolean
  replBridgeConnected: boolean
  replBridgeSessionActive: boolean
  // ...

  // 其他
  todos: { [agentId: string]: TodoList }
  notifications: {...}
  elicitation: {...}
  thinkingEnabled: boolean | undefined
  // ... 更多字段
}>
```

**问题分析**：
- AppState 包含 100+ 字段
- 领域状态与 UI 状态混合
- 缺乏清晰的状态边界
- 状态变更追踪困难

### 2.4 依赖关系分析

#### 2.4.1 模块依赖图

```
┌─────────────────────────────────────────────────────────────────┐
│                         依赖关系图                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  main.tsx                                                       │
│      │                                                          │
│      ├──► commands.ts ───────────────────► Command[]           │
│      │         │                                                │
│      │         └──► commands/* (各命令实现)                      │
│      │                                                          │
│      ├──► tools.ts ──────────────────────► Tools               │
│      │         │                                                │
│      │         ├──► Tool.ts (类型定义)                           │
│      │         │                                                │
│      │         └──► tools/* (各工具实现)                         │
│      │                   │                                      │
│      │                   └──► 循环依赖: tools.ts ↔ TeamCreateTool│
│      │                                                          │
│      ├──► QueryEngine.ts ────────────────► QueryEngine         │
│      │         │                                                │
│      │         ├──► Tool.ts                                     │
│      │         ├──► services/api/claude.ts                      │
│      │         ├──► state/AppState                              │
│      │         └──► memdir/                                     │
│      │                                                          │
│      ├──► state/AppState.tsx ────────────► AppStateProvider    │
│      │         │                                                │
│      │         └──► state/AppStateStore.ts ──► AppState        │
│      │                                                          │
│      └──► bridge/                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 2.4.2 循环依赖

```typescript
// tools.ts 中的循环依赖解决方式
// Lazy require to break circular dependency: tools.ts -> TeamCreateTool -> ... -> tools.ts
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
const getSendMessageTool = () =>
  require('./tools/SendMessageTool/SendMessageTool.js').SendMessageTool
```

**问题**：循环依赖表明模块边界不清晰，需要通过延迟加载解决。

### 2.5 技术债务清单

| 编号 | 债务类型 | 位置 | 描述 | 优先级 |
|-----|---------|------|------|--------|
| TD-001 | 巨型文件 | QueryEngine.ts | ~46K 行，职责过多 | P0 |
| TD-002 | 巨型文件 | Tool.ts | ~29K 行，类型与逻辑混杂 | P0 |
| TD-003 | 巨型文件 | commands.ts | ~25K 行，注册与实现混合 | P1 |
| TD-004 | 巨型文件 | tools.ts | ~17K 行，条件编译散布 | P1 |
| TD-005 | 循环依赖 | tools.ts ↔ TeamCreateTool | 需要延迟加载解决 | P1 |
| TD-006 | 状态分散 | AppState, FileStateCache, PermissionContext | 状态管理不统一 | P1 |
| TD-007 | 条件编译 | 各处 feature('XXX') | 代码可读性差 | P2 |
| TD-008 | 职责混乱 | ToolUseContext | 50+ 字段，职责不清 | P2 |

---

## 三、DDD 限界上下文划分

### 3.1 限界上下文识别方法

采用事件风暴（Event Storming）方法识别限界上下文：

1. **识别领域事件**：找出业务中的关键状态变更
2. **识别命令**：什么动作触发了这些事件
3. **识别聚合**：哪个对象处理命令、产生事件
4. **划分边界**：根据聚合簇确定限界上下文

### 3.2 领域事件识别

| 事件名称 | 触发命令 | 所属聚合 | 所属上下文 |
|---------|---------|---------|-----------|
| QueryStarted | SubmitQuery | Query | Query |
| QueryCompleted | CompleteQuery | Query | Query |
| QueryFailed | FailQuery | Query | Query |
| TokenUsageUpdated | UpdateUsage | Query | Query |
| ToolRegistered | RegisterTool | ToolRegistry | Tool |
| ToolExecutionStarted | ExecuteTool | ToolExecution | Tool |
| ToolExecutionCompleted | CompleteToolExecution | ToolExecution | Tool |
| PermissionRequired | RequestPermission | Permission | Permission |
| PermissionGranted | GrantPermission | Authorization | Permission |
| PermissionDenied | DenyPermission | Authorization | Permission |
| SessionCreated | CreateSession | Session | Session |
| SessionResumed | ResumeSession | Session | Session |
| SessionArchived | ArchiveSession | Session | Session |
| AgentSpawned | SpawnAgent | Agent | Agent |
| AgentCompleted | CompleteAgent | Agent | Agent |
| MessageSent | SendMessage | AgentMessage | Agent |
| BridgeConnected | ConnectBridge | BridgeConnection | Bridge |
| BridgeDisconnected | DisconnectBridge | BridgeConnection | Bridge |
| PluginLoaded | LoadPlugin | Plugin | Plugin |
| SkillExecuted | ExecuteSkill | SkillExecution | Skill |
| MemoryStored | StoreMemory | Memory | Memory |
| MemoryRetrieved | RetrieveMemory | Memory | Memory |

### 3.3 限界上下文地图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Claude Code 限界上下文地图                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        核心上下文层                               │   │
│  │                                                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │    Query    │  │    Tool     │  │   Command   │             │   │
│  │  │   查询处理   │  │   工具执行   │  │   命令处理   │             │   │
│  │  │             │  │             │  │             │             │   │
│  │  │ - Query     │  │ - Tool      │  │ - Command   │             │   │
│  │  │ - QueryTurn │  │ - ToolExec  │  │ - CmdExec   │             │   │
│  │  │ - TokenUsage│  │ - ToolResult│  │             │             │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │   │
│  │         │                │                │                    │   │
│  │         └────────────────┼────────────────┘                    │   │
│  │                          │                                     │   │
│  │  ┌───────────────────────┴───────────────────────┐             │   │
│  │  │                    Session                     │             │   │
│  │  │                   会话管理                      │             │   │
│  │  │                                               │             │   │
│  │  │ - Session                                     │             │   │
│  │  │ - SessionState                                │             │   │
│  │  │ - MessageHistory                              │             │   │
│  │  └───────────────────────────────────────────────┘             │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        支撑上下文层                               │   │
│  │                                                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │ Permission  │  │   Bridge    │  │   Plugin    │             │   │
│  │  │  权限控制    │  │  IDE 桥接   │  │  插件系统   │             │   │
│  │  │             │  │             │  │             │             │   │
│  │  │ - PermRule  │  │ - BridgeConn│  │ - Plugin    │             │   │
│  │  │ - Authz     │  │ - BridgeMsg │  │ - PluginReg │             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  │                                                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │   Skill     │  │   Agent     │  │   Memory    │             │   │
│  │  │  技能系统    │  │  子代理     │  │ 持久化记忆  │             │   │
│  │  │             │  │             │  │             │             │   │
│  │  │ - Skill     │  │ - Agent     │  │ - Memory    │             │   │
│  │  │ - SkillExec │  │ - AgentTeam │  │ - MemoryEnt │             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        基础设施层                                │   │
│  │                                                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │   │
│  │  │    API      │  │    MCP      │  │   Storage   │             │   │
│  │  │  API 客户端  │  │  MCP 服务   │  │   存储      │             │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘             │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.4 各限界上下文职责

| 上下文 | 核心职责 | 关键聚合 | 关键领域服务 |
|-------|---------|---------|-------------|
| **Query** | LLM API 调用、流式响应处理、Token 追踪 | Query, QueryTurn, TokenUsage | QueryOrchestrator |
| **Tool** | 工具注册、执行、权限检查 | Tool, ToolExecution, ToolResult | ToolRegistry, PermissionEvaluator |
| **Command** | 斜杠命令注册与执行 | Command, CommandExecution | CommandRegistry |
| **Session** | 会话生命周期、状态持久化 | Session, SessionState, MessageHistory | SessionManager |
| **Permission** | 权限规则、用户授权 | PermissionRule, Authorization | RuleMatcher, AuthorizationService |
| **Bridge** | IDE 扩展通信、远程控制 | BridgeConnection, BridgeMessage | BridgeProtocol |
| **Plugin** | 插件加载、生命周期管理 | Plugin, PluginRegistry | PluginLoader |
| **Skill** | 技能定义、执行编排 | Skill, SkillExecution | SkillExecutor |
| **Agent** | 子代理创建、协调、通信 | Agent, AgentTeam, AgentMessage | AgentCoordinator, TaskDistributor |
| **Memory** | 持久化记忆、团队同步 | Memory, MemoryEntry | MemoryRepository |

### 3.5 上下文关系矩阵

| 消费者 ↓ / 提供者 → | Query | Tool | Command | Session | Permission | Bridge | Plugin | Skill | Agent | Memory |
|---------------------|-------|------|---------|---------|------------|--------|--------|-------|-------|--------|
| **Query** | - | ✓ | ✓ | ✓ | ✓ | - | - | ✓ | ✓ | ✓ |
| **Tool** | - | - | - | - | ✓ | - | - | - | - | - |
| **Command** | ✓ | ✓ | - | ✓ | ✓ | - | ✓ | ✓ | - | ✓ |
| **Session** | ✓ | - | - | - | - | ✓ | - | - | ✓ | - |
| **Permission** | - | - | - | - | - | - | - | - | - | - |
| **Bridge** | - | - | - | ✓ | - | - | - | - | - | - |
| **Plugin** | - | ✓ | ✓ | - | - | - | - | - | - | - |
| **Skill** | ✓ | ✓ | - | - | ✓ | - | - | - | - | - |
| **Agent** | ✓ | ✓ | - | ✓ | ✓ | - | - | ✓ | - | ✓ |
| **Memory** | - | - | - | - | - | - | - | - | - | - |

---

## 四、各限界上下文详细设计

### 4.1 Query 限界上下文

#### 4.1.1 目录结构

```
src/contexts/query/
├── domain/
│   ├── entities/
│   │   ├── Query.ts                    # 查询实体
│   │   ├── QueryTurn.ts                # 查询轮次实体
│   │   └── StreamingContext.ts         # 流式上下文实体
│   │
│   ├── value-objects/
│   │   ├── QueryId.ts                  # 查询 ID 值对象
│   │   ├── ModelId.ts                  # 模型标识值对象
│   │   ├── TokenUsage.ts               # Token 使用量值对象
│   │   ├── QueryStatus.ts              # 查询状态枚举
│   │   ├── StreamingState.ts           # 流式状态值对象
│   │   ├── ThinkingConfig.ts           # 思考配置值对象
│   │   └── Budget.ts                   # 预算值对象
│   │
│   ├── aggregates/
│   │   └── QueryAggregate.ts           # 查询聚合根
│   │
│   ├── events/
│   │   ├── QueryCreatedEvent.ts
│   │   ├── QueryStartedEvent.ts
│   │   ├── QueryCompletedEvent.ts
│   │   ├── QueryFailedEvent.ts
│   │   ├── QueryCancelledEvent.ts
│   │   ├── TokenUsageUpdatedEvent.ts
│   │   ├── TurnStartedEvent.ts
│   │   ├── TurnCompletedEvent.ts
│   │   └── StreamDeltaReceivedEvent.ts
│   │
│   ├── services/
│   │   ├── QueryOrchestrator.ts        # 查询编排领域服务
│   │   ├── StreamProcessor.ts          # 流式处理领域服务
│   │   └── TokenCalculator.ts          # Token 计算领域服务
│   │
│   └── repositories/
│       └── IQueryRepository.ts         # 查询仓储接口
│
├── application/
│   ├── services/
│   │   └── QueryApplicationService.ts  # 查询应用服务
│   │
│   ├── commands/
│   │   ├── SubmitQueryCommand.ts       # 提交查询命令
│   │   ├── CancelQueryCommand.ts       # 取消查询命令
│   │   ├── RetryQueryCommand.ts        # 重试查询命令
│   │   └── CompactContextCommand.ts    # 压缩上下文命令
│   │
│   ├── queries/
│   │   ├── GetQueryStatusQuery.ts      # 获取查询状态查询
│   │   ├── GetTokenUsageQuery.ts       # 获取 Token 使用量查询
│   │   └── GetMessageHistoryQuery.ts   # 获取消息历史查询
│   │
│   └── handlers/
│       ├── SubmitQueryHandler.ts
│       ├── CancelQueryHandler.ts
│       ├── QueryEventHandler.ts
│       └── QueryDomainEventHandler.ts
│
├── infrastructure/
│   ├── repositories/
│   │   └── QueryRepository.ts          # 查询仓储实现
│   │
│   ├── adapters/
│   │   ├── AnthropicApiAdapter.ts      # Anthropic API 适配器
│   │   ├── StreamingAdapter.ts         # 流式响应适配器
│   │   └── BedrockApiAdapter.ts        # Bedrock API 适配器
│   │
│   ├── mappers/
│   │   ├── QueryMapper.ts              # 查询映射器
│   │   └── MessageMapper.ts            # 消息映射器
│   │
│   └── services/
│       └── ApiClientFactory.ts         # API 客户端工厂
│
└── interfaces/
    ├── QueryController.ts              # 查询控制器
    ├── QueryStreamController.ts        # 流式查询控制器
    └── dto/
        ├── QueryRequestDto.ts
        ├── QueryResponseDto.ts
        ├── StreamDeltaDto.ts
        └── TokenUsageDto.ts
```

#### 4.1.2 核心聚合设计

```typescript
// domain/aggregates/QueryAggregate.ts

import { QueryId } from '../value-objects/QueryId'
import { ModelId } from '../value-objects/ModelId'
import { TokenUsage } from '../value-objects/TokenUsage'
import { QueryStatus } from '../value-objects/QueryStatus'
import { ThinkingConfig } from '../value-objects/ThinkingConfig'
import { Budget } from '../value-objects/Budget'
import { QueryTurn } from '../entities/QueryTurn'
import { DomainEvent } from '../../../shared-kernel/DomainEvent'
import {
  QueryCreatedEvent,
  QueryStartedEvent,
  QueryCompletedEvent,
  QueryFailedEvent,
  QueryCancelledEvent,
  TokenUsageUpdatedEvent,
  TurnStartedEvent,
  TurnCompletedEvent,
} from '../events'

/**
 * QueryAggregate - 查询聚合根
 *
 * 职责：
 * - 管理查询生命周期
 * - 维护消息历史
 * - 追踪 Token 使用
 * - 发布领域事件
 *
 * 不变性约束：
 * - 一个查询可以有多个轮次
 * - 每个轮次有明确的开始和结束
 * - Token 使用量只能增加
 * - 状态转换必须遵循状态机
 */
export class QueryAggregate {
  private constructor(
    public readonly id: QueryId,
    private model: ModelId,
    private status: QueryStatus,
    private turns: QueryTurn[],
    private tokenUsage: TokenUsage,
    private thinkingConfig: ThinkingConfig,
    private budget: Budget,
    private domainEvents: DomainEvent[] = []
  ) {}

  // ==================== 工厂方法 ====================

  /**
   * 创建新查询
   */
  static create(config: {
    model: ModelId
    thinkingConfig: ThinkingConfig
    budget: Budget
  }): QueryAggregate {
    const query = new QueryAggregate(
      QueryId.generate(),
      config.model,
      QueryStatus.PENDING,
      [],
      TokenUsage.empty(),
      config.thinkingConfig,
      config.budget
    )

    query.addDomainEvent(new QueryCreatedEvent(query.id, query.model))

    return query
  }

  /**
   * 从持久化恢复查询
   */
  static reconstitute(data: {
    id: QueryId
    model: ModelId
    status: QueryStatus
    turns: QueryTurn[]
    tokenUsage: TokenUsage
    thinkingConfig: ThinkingConfig
    budget: Budget
  }): QueryAggregate {
    return new QueryAggregate(
      data.id,
      data.model,
      data.status,
      data.turns,
      data.tokenUsage,
      data.thinkingConfig,
      data.budget,
      [] // 恢复时不包含事件
    )
  }

  // ==================== 业务行为 ====================

  /**
   * 开始新轮次
   */
  startTurn(userMessage: UserMessage): QueryTurn {
    this.ensureStatus(QueryStatus.PENDING, QueryStatus.WAITING)

    const turn = QueryTurn.create({
      queryId: this.id,
      turnNumber: this.turns.length + 1,
      userMessage,
    })

    this.turns.push(turn)
    this.status = QueryStatus.PROCESSING

    this.addDomainEvent(new TurnStartedEvent(this.id, turn.id))

    return turn
  }

  /**
   * 开始流式处理
   */
  startStreaming(): void {
    this.ensureStatus(QueryStatus.PROCESSING)
    this.status = QueryStatus.STREAMING

    this.addDomainEvent(new QueryStartedEvent(this.id))
  }

  /**
   * 处理流式增量
   */
  handleStreamDelta(delta: StreamDelta): void {
    this.ensureStatus(QueryStatus.STREAMING)

    const currentTurn = this.getCurrentTurn()
    if (!currentTurn) {
      throw new QueryError('No active turn to handle stream delta')
    }

    currentTurn.appendDelta(delta)
  }

  /**
   * 完成当前轮次
   */
  completeTurn(result: TurnResult): void {
    this.ensureStatus(QueryStatus.STREAMING)

    const currentTurn = this.getCurrentTurn()
    if (!currentTurn) {
      throw new QueryError('No active turn to complete')
    }

    currentTurn.complete(result)

    // 更新 Token 使用量
    const newUsage = this.tokenUsage.add(result.usage)
    this.tokenUsage = newUsage

    this.addDomainEvent(new TurnCompletedEvent(this.id, currentTurn.id))
    this.addDomainEvent(new TokenUsageUpdatedEvent(this.id, newUsage))

    // 检查是否需要等待用户输入
    if (result.stopReason === 'end_turn') {
      this.status = QueryStatus.WAITING
    } else if (result.stopReason === 'tool_use') {
      this.status = QueryStatus.TOOL_PENDING
    }
  }

  /**
   * 完成查询
   */
  complete(): void {
    this.ensureStatus(QueryStatus.WAITING, QueryStatus.TOOL_PENDING)
    this.status = QueryStatus.COMPLETED

    this.addDomainEvent(new QueryCompletedEvent(this.id, this.tokenUsage))
  }

  /**
   * 查询失败
   */
  fail(error: QueryError): void {
    this.status = QueryStatus.FAILED

    this.addDomainEvent(new QueryFailedEvent(this.id, error))
  }

  /**
   * 取消查询
   */
  cancel(): void {
    if (this.status === QueryStatus.COMPLETED || this.status === QueryStatus.FAILED) {
      throw new QueryError('Cannot cancel a completed or failed query')
    }

    this.status = QueryStatus.CANCELLED

    this.addDomainEvent(new QueryCancelledEvent(this.id))
  }

  /**
   * 更新预算使用
   */
  updateBudgetUsage(cost: number): void {
    this.budget = this.budget.use(cost)

    if (this.budget.isExceeded()) {
      this.status = QueryStatus.BUDGET_EXCEEDED
    }
  }

  // ==================== 查询方法 ====================

  getCurrentTurn(): QueryTurn | null {
    return this.turns[this.turns.length - 1] ?? null
  }

  getTotalTurns(): number {
    return this.turns.length
  }

  getTokenUsage(): TokenUsage {
    return this.tokenUsage
  }

  getDomainEvents(): DomainEvent[] {
    return [...this.domainEvents]
  }

  clearDomainEvents(): void {
    this.domainEvents = []
  }

  // ==================== 私有方法 ====================

  private ensureStatus(...expected: QueryStatus[]): void {
    if (!expected.includes(this.status)) {
      throw new InvalidQueryStateError(this.status, expected)
    }
  }

  private addDomainEvent(event: DomainEvent): void {
    this.domainEvents.push(event)
  }
}
```

#### 4.1.3 值对象设计

```typescript
// domain/value-objects/QueryId.ts

import { randomUUID } from 'crypto'

export class QueryId {
  private constructor(private readonly value: string) {}

  static generate(): QueryId {
    return new QueryId(randomUUID())
  }

  static from(value: string): QueryId {
    if (!value || value.trim() === '') {
      throw new Error('QueryId cannot be empty')
    }
    return new QueryId(value)
  }

  equals(other: QueryId): boolean {
    return this.value === other.value
  }

  toString(): string {
    return this.value
  }
}

// domain/value-objects/TokenUsage.ts

export class TokenUsage {
  private constructor(
    public readonly inputTokens: number,
    public readonly outputTokens: number,
    public readonly cacheReadTokens: number,
    public readonly cacheWriteTokens: number
  ) {}

  static empty(): TokenUsage {
    return new TokenUsage(0, 0, 0, 0)
  }

  static from(data: {
    inputTokens: number
    outputTokens: number
    cacheReadTokens?: number
    cacheWriteTokens?: number
  }): TokenUsage {
    return new TokenUsage(
      data.inputTokens,
      data.outputTokens,
      data.cacheReadTokens ?? 0,
      data.cacheWriteTokens ?? 0
    )
  }

  add(other: TokenUsage): TokenUsage {
    return new TokenUsage(
      this.inputTokens + other.inputTokens,
      this.outputTokens + other.outputTokens,
      this.cacheReadTokens + other.cacheReadTokens,
      this.cacheWriteTokens + other.cacheWriteTokens
    )
  }

  getTotalTokens(): number {
    return this.inputTokens + this.outputTokens
  }

  getCacheTokens(): number {
    return this.cacheReadTokens + this.cacheWriteTokens
  }
}

// domain/value-objects/QueryStatus.ts

export enum QueryStatus {
  PENDING = 'pending',
  PROCESSING = 'processing',
  STREAMING = 'streaming',
  TOOL_PENDING = 'tool_pending',
  WAITING = 'waiting',
  COMPLETED = 'completed',
  FAILED = 'failed',
  CANCELLED = 'cancelled',
  BUDGET_EXCEEDED = 'budget_exceeded',
}
```

#### 4.1.4 领域事件设计

```typescript
// domain/events/QueryCreatedEvent.ts

import { DomainEvent } from '../../../shared-kernel/DomainEvent'
import { QueryId } from '../value-objects/QueryId'
import { ModelId } from '../value-objects/ModelId'

export class QueryCreatedEvent implements DomainEvent {
  readonly eventType = 'query.created'
  readonly occurredOn: Date

  constructor(
    public readonly queryId: QueryId,
    public readonly model: ModelId
  ) {
    this.occurredOn = new Date()
  }

  toPayload(): Record<string, unknown> {
    return {
      queryId: this.queryId.toString(),
      model: this.model.toString(),
    }
  }
}

// domain/events/QueryCompletedEvent.ts

import { DomainEvent } from '../../../shared-kernel/DomainEvent'
import { QueryId } from '../value-objects/QueryId'
import { TokenUsage } from '../value-objects/TokenUsage'

export class QueryCompletedEvent implements DomainEvent {
  readonly eventType = 'query.completed'
  readonly occurredOn: Date

  constructor(
    public readonly queryId: QueryId,
    public readonly tokenUsage: TokenUsage
  ) {
    this.occurredOn = new Date()
  }

  toPayload(): Record<string, unknown> {
    return {
      queryId: this.queryId.toString(),
      tokenUsage: {
        inputTokens: this.tokenUsage.inputTokens,
        outputTokens: this.tokenUsage.outputTokens,
        totalTokens: this.tokenUsage.getTotalTokens(),
      },
    }
  }
}
```

#### 4.1.5 仓储接口设计

```typescript
// domain/repositories/IQueryRepository.ts

import { QueryId } from '../value-objects/QueryId'
import { QueryAggregate } from '../aggregates/QueryAggregate'

export interface IQueryRepository {
  /**
   * 保存查询聚合
   */
  save(query: QueryAggregate): Promise<void>

  /**
   * 根据 ID 查找查询
   */
  findById(id: QueryId): Promise<QueryAggregate | null>

  /**
   * 删除查询
   */
  delete(id: QueryId): Promise<void>

  /**
   * 获取活跃查询
   */
  findActive(): Promise<QueryAggregate[]>
}
```

#### 4.1.6 应用服务设计

```typescript
// application/services/QueryApplicationService.ts

import { IQueryRepository } from '../../domain/repositories/IQueryRepository'
import { QueryAggregate } from '../../domain/aggregates/QueryAggregate'
import { SubmitQueryCommand } from '../commands/SubmitQueryCommand'
import { CancelQueryCommand } from '../commands/CancelQueryCommand'
import { IEventBus } from '../../../shared-kernel/IEventBus'

export class QueryApplicationService {
  constructor(
    private queryRepository: IQueryRepository,
    private apiAdapter: IApiAdapter,
    private eventBus: IEventBus
  ) {}

  /**
   * 提交查询
   */
  async submitQuery(command: SubmitQueryCommand): Promise<QueryAggregate> {
    // 1. 创建查询聚合
    const query = QueryAggregate.create({
      model: command.model,
      thinkingConfig: command.thinkingConfig,
      budget: command.budget,
    })

    // 2. 保存聚合
    await this.queryRepository.save(query)

    // 3. 发布领域事件
    await this.publishEvents(query)

    return query
  }

  /**
   * 执行查询
   */
  async *executeQuery(
    queryId: QueryId,
    prompt: string
  ): AsyncGenerator<StreamDelta, void, unknown> {
    // 1. 加载聚合
    const query = await this.queryRepository.findById(queryId)
    if (!query) {
      throw new Error(`Query not found: ${queryId}`)
    }

    // 2. 开始轮次
    query.startTurn(UserMessage.fromText(prompt))
    await this.queryRepository.save(query)
    await this.publishEvents(query)

    // 3. 调用 API
    const stream = this.apiAdapter.stream({
      model: query.model,
      messages: query.getMessages(),
      thinkingConfig: query.thinkingConfig,
    })

    // 4. 处理流式响应
    query.startStreaming()
    await this.queryRepository.save(query)

    for await (const delta of stream) {
      query.handleStreamDelta(delta)
      yield delta
    }

    // 5. 完成轮次
    query.completeTurn(/* result */)
    await this.queryRepository.save(query)
    await this.publishEvents(query)
  }

  /**
   * 取消查询
   */
  async cancelQuery(command: CancelQueryCommand): Promise<void> {
    const query = await this.queryRepository.findById(command.queryId)
    if (!query) {
      throw new Error(`Query not found: ${command.queryId}`)
    }

    query.cancel()
    await this.queryRepository.save(query)
    await this.publishEvents(query)
  }

  private async publishEvents(query: QueryAggregate): Promise<void> {
    const events = query.getDomainEvents()
    for (const event of events) {
      await this.eventBus.publish(event)
    }
    query.clearDomainEvents()
  }
}
```

### 4.2 Tool 限界上下文

#### 4.2.1 目录结构

```
src/contexts/tool/
├── domain/
│   ├── entities/
│   │   ├── Tool.ts                      # 工具实体
│   │   ├── ToolExecution.ts             # 工具执行实体
│   │   └── ToolResult.ts                # 工具结果实体
│   │
│   ├── value-objects/
│   │   ├── ToolName.ts                  # 工具名称值对象
│   │   ├── ToolSchema.ts                # 工具 Schema 值对象
│   │   ├── ExecutionId.ts               # 执行 ID 值对象
│   │   ├── ExecutionStatus.ts           # 执行状态枚举
│   │   ├── PermissionDecision.ts        # 权限决策值对象
│   │   └── ToolCategory.ts              # 工具分类枚举
│   │
│   ├── aggregates/
│   │   ├── ToolAggregate.ts             # 工具聚合根
│   │   └── ToolExecutionAggregate.ts    # 工具执行聚合根
│   │
│   ├── events/
│   │   ├── ToolRegisteredEvent.ts
│   │   ├── ToolUnregisteredEvent.ts
│   │   ├── ToolExecutionStartedEvent.ts
│   │   ├── ToolExecutionCompletedEvent.ts
│   │   ├── ToolExecutionFailedEvent.ts
│   │   └── PermissionRequiredEvent.ts
│   │
│   ├── services/
│   │   ├── ToolRegistry.ts              # 工具注册表领域服务
│   │   ├── PermissionEvaluator.ts       # 权限评估领域服务
│   │   ├── InputValidator.ts            # 输入验证领域服务
│   │   └── ToolExecutor.ts              # 工具执行领域服务
│   │
│   └── repositories/
│       ├── IToolRepository.ts           # 工具仓储接口
│       └── IToolExecutionRepository.ts  # 工具执行仓储接口
│
├── application/
│   ├── services/
│   │   └── ToolApplicationService.ts    # 工具应用服务
│   │
│   ├── commands/
│   │   ├── RegisterToolCommand.ts       # 注册工具命令
│   │   ├── UnregisterToolCommand.ts     # 注销工具命令
│   │   ├── ExecuteToolCommand.ts        # 执行工具命令
│   │   └── CancelToolExecutionCommand.ts # 取消执行命令
│   │
│   └── queries/
│       ├── GetToolByNameQuery.ts        # 按名称获取工具查询
│       ├── ListAvailableToolsQuery.ts   # 列出可用工具查询
│       └── GetExecutionStatusQuery.ts   # 获取执行状态查询
│
├── infrastructure/
│   ├── repositories/
│   │   ├── ToolRepository.ts            # 工具仓储实现
│   │   └── ToolExecutionRepository.ts   # 工具执行仓储实现
│   │
│   ├── executors/
│   │   ├── IToolExecutor.ts             # 工具执行器接口
│   │   ├── BashToolExecutor.ts          # Bash 工具执行器
│   │   ├── FileReadToolExecutor.ts      # 文件读取执行器
│   │   ├── FileWriteToolExecutor.ts     # 文件写入执行器
│   │   ├── FileEditToolExecutor.ts      # 文件编辑执行器
│   │   ├── GlobToolExecutor.ts          # Glob 工具执行器
│   │   ├── GrepToolExecutor.ts          # Grep 工具执行器
│   │   ├── WebFetchToolExecutor.ts      # Web 获取执行器
│   │   ├── WebSearchToolExecutor.ts     # Web 搜索执行器
│   │   ├── AgentToolExecutor.ts         # Agent 工具执行器
│   │   ├── SkillToolExecutor.ts         # Skill 工具执行器
│   │   ├── MCPToolExecutor.ts           # MCP 工具执行器
│   │   └── index.ts                     # 执行器注册
│   │
│   └── adapters/
│       ├── McpToolAdapter.ts            # MCP 工具适配器
│       └── PluginToolAdapter.ts         # 插件工具适配器
│
└── interfaces/
    ├── ToolController.ts                # 工具控制器
    └── dto/
        ├── ToolRegistrationDto.ts
        ├── ToolExecutionRequestDto.ts
        ├── ToolExecutionResponseDto.ts
        └── PermissionRequestDto.ts
```

#### 4.2.2 核心聚合设计

```typescript
// domain/aggregates/ToolAggregate.ts

import { ToolName } from '../value-objects/ToolName'
import { ToolSchema } from '../value-objects/ToolSchema'
import { ToolCategory } from '../value-objects/ToolCategory'
import { PermissionRule } from '../entities/PermissionRule'
import { DomainEvent } from '../../../shared-kernel/DomainEvent'
import {
  ToolRegisteredEvent,
  ToolUnregisteredEvent,
  PermissionRequiredEvent,
} from '../events'

/**
 * ToolAggregate - 工具聚合根
 *
 * 职责：
 * - 管理工具定义和 Schema
 * - 维护权限规则
 * - 控制工具启用/禁用状态
 *
 * 不变性约束：
 * - 工具名称唯一
 * - 权限规则按优先级排序
 * - 禁用的工具不能执行
 */
export class ToolAggregate {
  private constructor(
    public readonly name: ToolName,
    public readonly schema: ToolSchema,
    public readonly category: ToolCategory,
    private permissionRules: PermissionRule[],
    private enabled: boolean,
    private readonly aliases: string[],
    private domainEvents: DomainEvent[] = []
  ) {}

  // ==================== 工厂方法 ====================

  /**
   * 注册新工具
   */
  static register(config: {
    name: string
    schema: ToolSchema
    category: ToolCategory
    aliases?: string[]
  }): ToolAggregate {
    const tool = new ToolAggregate(
      ToolName.from(config.name),
      config.schema,
      config.category,
      [],
      true,
      config.aliases ?? []
    )

    tool.addDomainEvent(new ToolRegisteredEvent(tool.name))

    return tool
  }

  /**
   * 从 MCP 服务器创建工具
   */
  static fromMcpServer(mcpTool: McpToolDefinition): ToolAggregate {
    return ToolAggregate.register({
      name: `mcp__${mcpTool.serverName}__${mcpTool.toolName}`,
      schema: ToolSchema.fromMcpSchema(mcpTool.inputSchema),
      category: ToolCategory.MCP,
    })
  }

  // ==================== 业务行为 ====================

  /**
   * 检查是否可以执行
   */
  canExecute(context: ToolPermissionContext): PermissionDecision {
    // 1. 检查工具是否启用
    if (!this.enabled) {
      return PermissionDecision.denied('Tool is disabled')
    }

    // 2. 评估权限规则
    const evaluator = new PermissionEvaluator(this.permissionRules)
    return evaluator.evaluate(context)
  }

  /**
   * 执行工具
   */
  execute(
    input: ToolInput,
    context: ToolUseContext
  ): ToolExecutionAggregate {
    // 1. 验证权限
    const decision = this.canExecute(context.toolPermissionContext)

    if (decision.isDenied) {
      return ToolExecutionAggregate.denied(
        this.name,
        input,
        decision.reason
      )
    }

    if (decision.requiresApproval) {
      this.addDomainEvent(
        new PermissionRequiredEvent(this.name, input)
      )
      return ToolExecutionAggregate.pendingApproval(this.name, input)
    }

    // 2. 验证输入
    const validationResult = this.schema.validate(input)
    if (!validationResult.valid) {
      return ToolExecutionAggregate.invalidInput(
        this.name,
        input,
        validationResult.errors
      )
    }

    // 3. 创建执行
    return ToolExecutionAggregate.start(this.name, input, context)
  }

  /**
   * 添加权限规则
   */
  addPermissionRule(rule: PermissionRule): void {
    this.permissionRules.push(rule)
    this.permissionRules.sort((a, b) => b.priority - a.priority)
  }

  /**
   * 移除权限规则
   */
  removePermissionRule(ruleId: string): void {
    this.permissionRules = this.permissionRules.filter(
      r => r.id !== ruleId
    )
  }

  /**
   * 禁用工具
   */
  disable(): void {
    this.enabled = false
  }

  /**
   * 启用工具
   */
  enable(): void {
    this.enabled = true
  }

  // ==================== 查询方法 ====================

  isEnabled(): boolean {
    return this.enabled
  }

  hasAlias(name: string): boolean {
    return this.aliases.includes(name)
  }

  getPermissionRules(): PermissionRule[] {
    return [...this.permissionRules]
  }

  getDomainEvents(): DomainEvent[] {
    return [...this.domainEvents]
  }

  clearDomainEvents(): void {
    this.domainEvents = []
  }

  private addDomainEvent(event: DomainEvent): void {
    this.domainEvents.push(event)
  }
}

// domain/aggregates/ToolExecutionAggregate.ts

import { ExecutionId } from '../value-objects/ExecutionId'
import { ExecutionStatus } from '../value-objects/ExecutionStatus'
import { ToolName } from '../value-objects/ToolName'

/**
 * ToolExecutionAggregate - 工具执行聚合根
 *
 * 职责：
 * - 管理执行生命周期
 * - 追踪执行状态
 * - 存储执行结果
 *
 * 不变性约束：
 * - 执行只能开始一次
 * - 结果只能在执行完成后设置
 * - 取消只能在执行中进行
 */
export class ToolExecutionAggregate {
  private constructor(
    public readonly id: ExecutionId,
    public readonly toolName: ToolName,
    public readonly input: ToolInput,
    private status: ExecutionStatus,
    private result?: ToolResult,
    private error?: ToolError,
    private startedAt?: Date,
    private completedAt?: Date,
    private domainEvents: DomainEvent[] = []
  ) {}

  // ==================== 工厂方法 ====================

  /**
   * 开始执行
   */
  static start(
    toolName: ToolName,
    input: ToolInput,
    context: ToolUseContext
  ): ToolExecutionAggregate {
    const execution = new ToolExecutionAggregate(
      ExecutionId.generate(),
      toolName,
      input,
      ExecutionStatus.RUNNING,
      undefined,
      undefined,
      new Date()
    )

    execution.addDomainEvent(
      new ToolExecutionStartedEvent(execution.id, toolName, input)
    )

    return execution
  }

  /**
   * 权限拒绝
   */
  static denied(
    toolName: ToolName,
    input: ToolInput,
    reason: string
  ): ToolExecutionAggregate {
    return new ToolExecutionAggregate(
      ExecutionId.generate(),
      toolName,
      input,
      ExecutionStatus.DENIED,
      undefined,
      new ToolError('PERMISSION_DENIED', reason)
    )
  }

  /**
   * 等待审批
   */
  static pendingApproval(
    toolName: ToolName,
    input: ToolInput
  ): ToolExecutionAggregate {
    return new ToolExecutionAggregate(
      ExecutionId.generate(),
      toolName,
      input,
      ExecutionStatus.PENDING_APPROVAL
    )
  }

  /**
   * 输入无效
   */
  static invalidInput(
    toolName: ToolName,
    input: ToolInput,
    errors: ValidationError[]
  ): ToolExecutionAggregate {
    return new ToolExecutionAggregate(
      ExecutionId.generate(),
      toolName,
      input,
      ExecutionStatus.INVALID_INPUT,
      undefined,
      new ToolError('INVALID_INPUT', errors.map(e => e.message).join(', '))
    )
  }

  // ==================== 业务行为 ====================

  /**
   * 完成执行
   */
  complete(result: ToolResult): void {
    this.ensureStatus(ExecutionStatus.RUNNING)

    this.status = ExecutionStatus.COMPLETED
    this.result = result
    this.completedAt = new Date()

    this.addDomainEvent(
      new ToolExecutionCompletedEvent(this.id, this.toolName, result)
    )
  }

  /**
   * 执行失败
   */
  fail(error: ToolError): void {
    this.ensureStatus(ExecutionStatus.RUNNING)

    this.status = ExecutionStatus.FAILED
    this.error = error
    this.completedAt = new Date()

    this.addDomainEvent(
      new ToolExecutionFailedEvent(this.id, this.toolName, error)
    )
  }

  /**
   * 取消执行
   */
  cancel(): void {
    this.ensureStatus(ExecutionStatus.RUNNING, ExecutionStatus.PENDING_APPROVAL)

    this.status = ExecutionStatus.CANCELLED
    this.completedAt = new Date()
  }

  /**
   * 批准执行
   */
  approve(): void {
    this.ensureStatus(ExecutionStatus.PENDING_APPROVAL)

    this.status = ExecutionStatus.APPROVED
    // 执行器会检测状态并开始执行
  }

  /**
   * 拒绝执行
   */
  reject(reason: string): void {
    this.ensureStatus(ExecutionStatus.PENDING_APPROVAL)

    this.status = ExecutionStatus.REJECTED
    this.error = new ToolError('USER_REJECTED', reason)
    this.completedAt = new Date()
  }

  // ==================== 查询方法 ====================

  getStatus(): ExecutionStatus {
    return this.status
  }

  getResult(): ToolResult | undefined {
    return this.result
  }

  getError(): ToolError | undefined {
    return this.error
  }

  getDuration(): number | undefined {
    if (!this.startedAt || !this.completedAt) return undefined
    return this.completedAt.getTime() - this.startedAt.getTime()
  }

  isTerminal(): boolean {
    return [
      ExecutionStatus.COMPLETED,
      ExecutionStatus.FAILED,
      ExecutionStatus.CANCELLED,
      ExecutionStatus.REJECTED,
      ExecutionStatus.DENIED,
    ].includes(this.status)
  }

  private ensureStatus(...expected: ExecutionStatus[]): void {
    if (!expected.includes(this.status)) {
      throw new InvalidExecutionStateError(this.status, expected)
    }
  }

  private addDomainEvent(event: DomainEvent): void {
    this.domainEvents.push(event)
  }
}
```

#### 4.2.3 工具执行器接口

```typescript
// infrastructure/executors/IToolExecutor.ts

import { ToolExecutionAggregate } from '../../domain/aggregates/ToolExecutionAggregate'
import { ToolInput } from '../../domain/value-objects/ToolInput'

/**
 * 工具执行器接口
 *
 * 每个具体工具实现此接口
 */
export interface IToolExecutor {
  /**
   * 工具名称
   */
  readonly toolName: string

  /**
   * 执行工具
   */
  execute(
    input: ToolInput,
    context: ToolUseContext
  ): Promise<ToolResult>

  /**
   * 验证输入
   */
  validateInput(input: ToolInput): ValidationResult

  /**
   * 检查是否只读
   */
  isReadOnly(input: ToolInput): boolean

  /**
   * 检查是否并发安全
   */
  isConcurrencySafe(input: ToolInput): boolean

  /**
   * 检查是否破坏性操作
   */
  isDestructive?(input: ToolInput): boolean
}

// infrastructure/executors/BashToolExecutor.ts

import { IToolExecutor } from './IToolExecutor'
import { exec } from '../../../utils/Shell'
import { SandboxManager } from '../../../utils/sandbox/sandbox-adapter'

export class BashToolExecutor implements IToolExecutor {
  readonly toolName = 'Bash'

  async execute(input: ToolInput, context: ToolUseContext): Promise<ToolResult> {
    const { command, timeout, sandbox } = input

    // 1. 安全检查
    const securityCheck = this.performSecurityCheck(command)
    if (!securityCheck.safe) {
      return ToolResult.error(securityCheck.reason)
    }

    // 2. 沙箱处理
    const execOptions = sandbox
      ? await SandboxManager.getSandboxOptions(command)
      : {}

    // 3. 执行命令
    const result = await exec(command, {
      timeout: timeout ?? this.getDefaultTimeout(command),
      ...execOptions,
    })

    // 4. 返回结果
    return ToolResult.success({
      stdout: result.stdout,
      stderr: result.stderr,
      exitCode: result.exitCode,
    })
  }

  validateInput(input: ToolInput): ValidationResult {
    if (!input.command || input.command.trim() === '') {
      return { valid: false, errors: [{ message: 'Command is required' }] }
    }
    return { valid: true, errors: [] }
  }

  isReadOnly(input: ToolInput): boolean {
    // 分析命令是否只读
    const readOnlyCommands = ['ls', 'cat', 'head', 'tail', 'grep', 'find', 'wc']
    const baseCommand = input.command.trim().split(/\s+/)[0]
    return readOnlyCommands.includes(baseCommand)
  }

  isConcurrencySafe(input: ToolInput): boolean {
    // 大多数 bash 命令不是并发安全的
    return false
  }

  private performSecurityCheck(command: string): SecurityCheckResult {
    // 安全检查逻辑
    const dangerousPatterns = [
      /rm\s+-rf\s+\//,  // 删除根目录
      />\s*\/dev\/sd/,  // 覆盖磁盘
      /mkfs/,           // 格式化
    ]

    for (const pattern of dangerousPatterns) {
      if (pattern.test(command)) {
        return { safe: false, reason: 'Dangerous command detected' }
      }
    }

    return { safe: true }
  }

  private getDefaultTimeout(command: string): number {
    // 根据命令类型返回默认超时
    if (command.includes('npm install') || command.includes('bun install')) {
      return 300000 // 5 分钟
    }
    return 120000 // 2 分钟
  }
}
```

### 4.3 Session 限界上下文

#### 4.3.1 目录结构

```
src/contexts/session/
├── domain/
│   ├── entities/
│   │   ├── Session.ts                    # 会话实体
│   │   └── SessionMetadata.ts            # 会话元数据实体
│   │
│   ├── value-objects/
│   │   ├── SessionId.ts                  # 会话 ID 值对象
│   │   ├── SessionStatus.ts              # 会话状态枚举
│   │   └── SessionConfig.ts              # 会话配置值对象
│   │
│   ├── aggregates/
│   │   └── SessionAggregate.ts           # 会话聚合根
│   │
│   ├── events/
│   │   ├── SessionCreatedEvent.ts
│   │   ├── SessionResumedEvent.ts
│   │   ├── SessionArchivedEvent.ts
│   │   ├── SessionStateSavedEvent.ts
│   │   └── SessionExpiredEvent.ts
│   │
│   ├── services/
│   │   └── SessionManager.ts             # 会话管理领域服务
│   │
│   └── repositories/
│       └── ISessionRepository.ts         # 会话仓储接口
│
├── application/
│   ├── services/
│   │   └── SessionApplicationService.ts  # 会话应用服务
│   │
│   ├── commands/
│   │   ├── CreateSessionCommand.ts       # 创建会话命令
│   │   ├── ResumeSessionCommand.ts       # 恢复会话命令
│   │   ├── ArchiveSessionCommand.ts      # 归档会话命令
│   │   └── SaveSessionStateCommand.ts    # 保存状态命令
│   │
│   └── queries/
│       ├── GetSessionByIdQuery.ts        # 按 ID 获取会话查询
│       ├── ListRecentSessionsQuery.ts    # 列出最近会话查询
│       └── GetSessionStateQuery.ts       # 获取会话状态查询
│
├── infrastructure/
│   ├── repositories/
│   │   └── SessionRepository.ts          # 会话仓储实现
│   │
│   ├── storage/
│   │   ├── SessionStorage.ts             # 会话存储
│   │   ├── SessionSerializer.ts          # 会话序列化
│   │   └── SessionCompressor.ts          # 会话压缩
│   │
│   └── adapters/
│       └── SessionBackupAdapter.ts       # 会话备份适配器
│
└── interfaces/
    ├── SessionController.ts              # 会话控制器
    └── dto/
        ├── SessionDto.ts
        ├── SessionStateDto.ts
        └── SessionListDto.ts
```

#### 4.3.2 核心聚合设计

```typescript
// domain/aggregates/SessionAggregate.ts

import { SessionId } from '../value-objects/SessionId'
import { SessionStatus } from '../value-objects/SessionStatus'
import { SessionConfig } from '../value-objects/SessionConfig'
import { Message } from '../../../shared-kernel/types/Message'
import { DomainEvent } from '../../../shared-kernel/DomainEvent'
import {
  SessionCreatedEvent,
  SessionResumedEvent,
  SessionArchivedEvent,
  SessionStateSavedEvent,
} from '../events'

/**
 * SessionAggregate - 会话聚合根
 *
 * 职责：
 * - 管理会话生命周期
 * - 维护消息历史
 * - 持久化会话状态
 *
 * 不变性约束：
 * - 会话创建后状态为 ACTIVE
 * - 归档后不能再添加消息
 * - 消息只能追加，不能修改或删除
 */
export class SessionAggregate {
  private constructor(
    public readonly id: SessionId,
    private status: SessionStatus,
    private config: SessionConfig,
    private messages: Message[],
    private metadata: SessionMetadata,
    private createdAt: Date,
    private updatedAt: Date,
    private domainEvents: DomainEvent[] = []
  ) {}

  // ==================== 工厂方法 ====================

  /**
   * 创建新会话
   */
  static create(config: SessionConfig): SessionAggregate {
    const now = new Date()
    const session = new SessionAggregate(
      SessionId.generate(),
      SessionStatus.ACTIVE,
      config,
      [],
      SessionMetadata.create(),
      now,
      now
    )

    session.addDomainEvent(new SessionCreatedEvent(session.id, config))

    return session
  }

  /**
   * 从持久化恢复会话
   */
  static reconstitute(data: {
    id: SessionId
    status: SessionStatus
    config: SessionConfig
    messages: Message[]
    metadata: SessionMetadata
    createdAt: Date
    updatedAt: Date
  }): SessionAggregate {
    return new SessionAggregate(
      data.id,
      data.status,
      data.config,
      data.messages,
      data.metadata,
      data.createdAt,
      data.updatedAt
    )
  }

  // ==================== 业务行为 ====================

  /**
   * 添加消息
   */
  addMessage(message: Message): void {
    this.ensureStatus(SessionStatus.ACTIVE)

    this.messages.push(message)
    this.updatedAt = new Date()
    this.metadata = this.metadata.withMessageAdded()
  }

  /**
   * 批量添加消息
   */
  addMessages(messages: Message[]): void {
    this.ensureStatus(SessionStatus.ACTIVE)

    this.messages.push(...messages)
    this.updatedAt = new Date()
    this.metadata = this.metadata.withMessagesAdded(messages.length)
  }

  /**
   * 恢复会话
   */
  resume(): void {
    this.ensureStatus(SessionStatus.ACTIVE)

    this.addDomainEvent(new SessionResumedEvent(this.id))
  }

  /**
   * 归档会话
   */
  archive(): void {
    this.status = SessionStatus.ARCHIVED
    this.updatedAt = new Date()

    this.addDomainEvent(new SessionArchivedEvent(this.id))
  }

  /**
   * 保存状态快照
   */
  saveSnapshot(): SessionSnapshot {
    const snapshot = new SessionSnapshot(
      this.id,
      [...this.messages],
      this.metadata,
      this.updatedAt
    )

    this.addDomainEvent(new SessionStateSavedEvent(this.id))

    return snapshot
  }

  /**
   * 压缩历史消息
   */
  compactHistory(compactor: ICompactor): void {
    this.ensureStatus(SessionStatus.ACTIVE)

    const compactedMessages = compactor.compact(this.messages)
    this.messages = compactedMessages
    this.updatedAt = new Date()
    this.metadata = this.metadata.withCompactionApplied()
  }

  // ==================== 查询方法 ====================

  getMessages(): Message[] {
    return [...this.messages]
  }

  getMessageCount(): number {
    return this.messages.length
  }

  getLastMessage(): Message | undefined {
    return this.messages[this.messages.length - 1]
  }

  getDuration(): number {
    return this.updatedAt.getTime() - this.createdAt.getTime()
  }

  isActive(): boolean {
    return this.status === SessionStatus.ACTIVE
  }

  getDomainEvents(): DomainEvent[] {
    return [...this.domainEvents]
  }

  clearDomainEvents(): void {
    this.domainEvents = []
  }

  private ensureStatus(expected: SessionStatus): void {
    if (this.status !== expected) {
      throw new InvalidSessionStateError(this.status, expected)
    }
  }

  private addDomainEvent(event: DomainEvent): void {
    this.domainEvents.push(event)
  }
}
```

### 4.4 Permission 限界上下文

#### 4.4.1 目录结构

```
src/contexts/permission/
├── domain/
│   ├── entities/
│   │   ├── PermissionRule.ts             # 权限规则实体
│   │   └── Authorization.ts              # 授权实体
│   │
│   ├── value-objects/
│   │   ├── PermissionMode.ts             # 权限模式枚举
│   │   ├── PermissionDecision.ts         # 权限决策值对象
│   │   ├── RuleScope.ts                  # 规则范围值对象
│   │   ├── RulePriority.ts               # 规则优先级值对象
│   │   └── PermissionSource.ts           # 权限来源枚举
│   │
│   ├── aggregates/
│   │   └── PermissionContextAggregate.ts # 权限上下文聚合根
│   │
│   ├── events/
│   │   ├── PermissionGrantedEvent.ts
│   │   ├── PermissionDeniedEvent.ts
│   │   ├── PermissionModeChangedEvent.ts
│   │   └── RuleAddedEvent.ts
│   │
│   ├── services/
│   │   ├── RuleMatcher.ts                # 规则匹配领域服务
│   │   ├── AuthorizationService.ts       # 授权领域服务
│   │   └── PermissionInheritance.ts      # 权限继承领域服务
│   │
│   └── repositories/
│       └── IPermissionRuleRepository.ts  # 权限规则仓储接口
│
├── application/
│   ├── services/
│   │   └── PermissionApplicationService.ts
│   │
│   ├── commands/
│   │   ├── GrantPermissionCommand.ts
│   │   ├── DenyPermissionCommand.ts
│   │   ├── SetPermissionModeCommand.ts
│   │   └── AddPermissionRuleCommand.ts
│   │
│   └── queries/
│       ├── GetPermissionContextQuery.ts
│       └── CheckPermissionQuery.ts
│
├── infrastructure/
│   ├── repositories/
│   │   └── PermissionRuleRepository.ts
│   │
│   ├── evaluators/
│   │   ├── RuleEvaluator.ts
│   │   ├── BypassModeEvaluator.ts
│   │   └── AutoModeEvaluator.ts
│   │
│   └── adapters/
│       └── HookPermissionAdapter.ts
│
└── interfaces/
    ├── PermissionController.ts
    └── dto/
        ├── PermissionRequestDto.ts
        ├── PermissionResponseDto.ts
        └── PermissionRuleDto.ts
```

#### 4.4.2 核心聚合设计

```typescript
// domain/aggregates/PermissionContextAggregate.ts

import { PermissionMode } from '../value-objects/PermissionMode'
import { PermissionDecision } from '../value-objects/PermissionDecision'
import { PermissionRule } from '../entities/PermissionRule'
import { DomainEvent } from '../../../shared-kernel/DomainEvent'
import {
  PermissionGrantedEvent,
  PermissionDeniedEvent,
  PermissionModeChangedEvent,
} from '../events'

/**
 * PermissionContextAggregate - 权限上下文聚合根
 *
 * 职责：
 * - 管理权限模式
 * - 维护权限规则
 * - 评估权限请求
 *
 * 不变性约束：
 * - 规则按优先级排序
 * - Bypass 模式跳过所有规则
 * - 决策必须可追溯
 */
export class PermissionContextAggregate {
  private constructor(
    private mode: PermissionMode,
    private alwaysAllowRules: PermissionRule[],
    private alwaysDenyRules: PermissionRule[],
    private alwaysAskRules: PermissionRule[],
    private decisionHistory: PermissionDecision[],
    private domainEvents: DomainEvent[] = []
  ) {}

  // ==================== 工厂方法 ====================

  /**
   * 创建默认权限上下文
   */
  static createDefault(): PermissionContextAggregate {
    return new PermissionContextAggregate(
      PermissionMode.DEFAULT,
      [],
      [],
      [],
      []
    )
  }

  /**
   * 创建 Bypass 模式上下文
   */
  static createBypass(): PermissionContextAggregate {
    return new PermissionContextAggregate(
      PermissionMode.BYPASS,
      [],
      [],
      [],
      []
    )
  }

  // ==================== 业务行为 ====================

  /**
   * 检查权限
   */
  checkPermission(request: PermissionRequest): PermissionDecision {
    // 1. Bypass 模式直接允许
    if (this.mode === PermissionMode.BYPASS) {
      const decision = PermissionDecision.allowed('Bypass mode')
      this.recordDecision(decision)
      return decision
    }

    // 2. 检查拒绝规则
    const denyRule = this.matchRule(this.alwaysDenyRules, request)
    if (denyRule) {
      const decision = PermissionDecision.denied(denyRule.reason)
      this.recordDecision(decision)
      this.addDomainEvent(new PermissionDeniedEvent(request, denyRule))
      return decision
    }

    // 3. 检查允许规则
    const allowRule = this.matchRule(this.alwaysAllowRules, request)
    if (allowRule) {
      const decision = PermissionDecision.allowed(allowRule.reason)
      this.recordDecision(decision)
      this.addDomainEvent(new PermissionGrantedEvent(request, allowRule))
      return decision
    }

    // 4. 检查询问规则
    const askRule = this.matchRule(this.alwaysAskRules, request)
    if (askRule) {
      const decision = PermissionDecision.needsApproval(askRule.reason)
      this.recordDecision(decision)
      return decision
    }

    // 5. 默认行为
    return this.getDefaultDecision(request)
  }

  /**
   * 设置权限模式
   */
  setMode(newMode: PermissionMode): void {
    const oldMode = this.mode
    this.mode = newMode

    this.addDomainEvent(new PermissionModeChangedEvent(oldMode, newMode))
  }

  /**
   * 添加允许规则
   */
  addAllowRule(rule: PermissionRule): void {
    this.alwaysAllowRules.push(rule)
    this.sortRules(this.alwaysAllowRules)
  }

  /**
   * 添加拒绝规则
   */
  addDenyRule(rule: PermissionRule): void {
    this.alwaysDenyRules.push(rule)
    this.sortRules(this.alwaysDenyRules)
  }

  /**
   * 添加询问规则
   */
  addAskRule(rule: PermissionRule): void {
    this.alwaysAskRules.push(rule)
    this.sortRules(this.alwaysAskRules)
  }

  // ==================== 查询方法 ====================

  getMode(): PermissionMode {
    return this.mode
  }

  getDecisionHistory(): PermissionDecision[] {
    return [...this.decisionHistory]
  }

  // ==================== 私有方法 ====================

  private matchRule(
    rules: PermissionRule[],
    request: PermissionRequest
  ): PermissionRule | null {
    const matcher = new RuleMatcher(rules)
    return matcher.match(request)
  }

  private recordDecision(decision: PermissionDecision): void {
    this.decisionHistory.push(decision)
    // 保留最近 100 条决策记录
    if (this.decisionHistory.length > 100) {
      this.decisionHistory = this.decisionHistory.slice(-100)
    }
  }

  private getDefaultDecision(request: PermissionRequest): PermissionDecision {
    switch (this.mode) {
      case PermissionMode.AUTO:
        return PermissionDecision.allowed('Auto mode default')
      case PermissionMode.PLAN:
        return PermissionDecision.needsApproval('Plan mode requires approval')
      default:
        return PermissionDecision.needsApproval('Default behavior')
    }
  }

  private sortRules(rules: PermissionRule[]): void {
    rules.sort((a, b) => b.priority - a.priority)
  }

  private addDomainEvent(event: DomainEvent): void {
    this.domainEvents.push(event)
  }
}
```

### 4.5 Agent 限界上下文

#### 4.5.1 目录结构

```
src/contexts/agent/
├── domain/
│   ├── entities/
│   │   ├── Agent.ts                      # Agent 实体
│   │   ├── AgentTeam.ts                  # Agent 团队实体
│   │   └── AgentMessage.ts               # Agent 消息实体
│   │
│   ├── value-objects/
│   │   ├── AgentId.ts                    # Agent ID 值对象
│   │   ├── AgentRole.ts                  # Agent 角色枚举
│   │   ├── AgentStatus.ts                # Agent 状态枚举
│   │   ├── AgentCapability.ts            # Agent 能力值对象
│   │   └── AgentConfig.ts                # Agent 配置值对象
│   │
│   ├── aggregates/
│   │   ├── AgentAggregate.ts             # Agent 聚合根
│   │   └── AgentTeamAggregate.ts         # Agent 团队聚合根
│   │
│   ├── events/
│   │   ├── AgentSpawnedEvent.ts
│   │   ├── AgentCompletedEvent.ts
│   │   ├── AgentFailedEvent.ts
│   │   ├── AgentMessageSentEvent.ts
│   │   ├── TeamCreatedEvent.ts
│   │   ├── TeamMemberAddedEvent.ts
│   │   └── TeamDissolvedEvent.ts
│   │
│   ├── services/
│   │   ├── AgentCoordinator.ts           # Agent 协调领域服务
│   │   ├── TaskDistributor.ts            # 任务分配领域服务
│   │   └── AgentCommunication.ts         # Agent 通信领域服务
│   │
│   └── repositories/
│       ├── IAgentRepository.ts           # Agent 仓储接口
│       └── IAgentTeamRepository.ts       # Agent 团队仓储接口
│
├── application/
│   ├── services/
│   │   └── AgentApplicationService.ts
│   │
│   ├── commands/
│   │   ├── SpawnAgentCommand.ts
│   │   ├── SendMessageToAgentCommand.ts
│   │   ├── CreateTeamCommand.ts
│   │   ├── AddTeamMemberCommand.ts
│   │   └── TerminateAgentCommand.ts
│   │
│   └── queries/
│       ├── GetAgentStatusQuery.ts
│       ├── ListTeamMembersQuery.ts
│       └── GetAgentMessagesQuery.ts
│
├── infrastructure/
│   ├── repositories/
│   │   ├── AgentRepository.ts
│   │   └── AgentTeamRepository.ts
│   │
│   ├── coordinators/
│   │   └── CoordinatorModeCoordinator.ts
│   │
│   └── adapters/
│       ├── SubagentContextAdapter.ts
│       └── AgentQueryAdapter.ts
│
└── interfaces/
    ├── AgentController.ts
    └── dto/
        ├── AgentDto.ts
        ├── AgentTeamDto.ts
        └── AgentMessageDto.ts
```

#### 4.5.2 核心聚合设计

```typescript
// domain/aggregates/AgentAggregate.ts

import { AgentId } from '../value-objects/AgentId'
import { AgentRole } from '../value-objects/AgentRole'
import { AgentStatus } from '../value-objects/AgentStatus'
import { AgentConfig } from '../value-objects/AgentConfig'
import { AgentMessage } from '../entities/AgentMessage'
import { DomainEvent } from '../../../shared-kernel/DomainEvent'
import {
  AgentSpawnedEvent,
  AgentCompletedEvent,
  AgentFailedEvent,
  AgentMessageSentEvent,
} from '../events'

/**
 * AgentAggregate - Agent 聚合根
 *
 * 职责：
 * - 管理 Agent 生命周期
 * - 维护 Agent 状态
 * - 处理 Agent 间通信
 *
 * 不变性约束：
 * - Agent 只能被终止一次
 * - 消息只能追加
 * - 状态转换必须遵循状态机
 */
export class AgentAggregate {
  private constructor(
    public readonly id: AgentId,
    public readonly role: AgentRole,
    private config: AgentConfig,
    private status: AgentStatus,
    private messages: AgentMessage[],
    private parentAgentId?: AgentId,
    private teamId?: string,
    private createdAt: Date = new Date(),
    private completedAt?: Date,
    private domainEvents: DomainEvent[] = []
  ) {}

  // ==================== 工厂方法 ====================

  /**
   * 创建子 Agent
   */
  static spawn(config: {
    role: AgentRole
    config: AgentConfig
    parentAgentId?: AgentId
    teamId?: string
  }): AgentAggregate {
    const agent = new AgentAggregate(
      AgentId.generate(),
      config.role,
      config.config,
      AgentStatus.SPAWNING,
      [],
      config.parentAgentId,
      config.teamId
    )

    agent.addDomainEvent(new AgentSpawnedEvent(agent.id, config.role))

    return agent
  }

  // ==================== 业务行为 ====================

  /**
   * 开始运行
   */
  start(): void {
    this.ensureStatus(AgentStatus.SPAWNING)
    this.status = AgentStatus.RUNNING
  }

  /**
   * 发送消息
   */
  sendMessage(message: AgentMessage): void {
    this.ensureStatus(AgentStatus.RUNNING)

    this.messages.push(message)
    this.addDomainEvent(new AgentMessageSentEvent(this.id, message))
  }

  /**
   * 接收消息
   */
  receiveMessage(message: AgentMessage): void {
    this.ensureStatus(AgentStatus.RUNNING)

    this.messages.push(message)
  }

  /**
   * 完成任务
   */
  complete(result: AgentResult): void {
    this.ensureStatus(AgentStatus.RUNNING)

    this.status = AgentStatus.COMPLETED
    this.completedAt = new Date()

    this.addDomainEvent(new AgentCompletedEvent(this.id, result))
  }

  /**
   * 任务失败
   */
  fail(error: AgentError): void {
    this.ensureStatus(AgentStatus.RUNNING)

    this.status = AgentStatus.FAILED
    this.completedAt = new Date()

    this.addDomainEvent(new AgentFailedEvent(this.id, error))
  }

  /**
   * 终止 Agent
   */
  terminate(): void {
    if (this.status === AgentStatus.COMPLETED || this.status === AgentStatus.FAILED) {
      return // 已完成或失败，无需终止
    }

    this.status = AgentStatus.TERMINATED
    this.completedAt = new Date()
  }

  // ==================== 查询方法 ====================

  getStatus(): AgentStatus {
    return this.status
  }

  getMessages(): AgentMessage[] {
    return [...this.messages]
  }

  isRunning(): boolean {
    return this.status === AgentStatus.RUNNING
  }

  isTerminal(): boolean {
    return [
      AgentStatus.COMPLETED,
      AgentStatus.FAILED,
      AgentStatus.TERMINATED,
    ].includes(this.status)
  }

  getDuration(): number | undefined {
    if (!this.completedAt) return undefined
    return this.completedAt.getTime() - this.createdAt.getTime()
  }

  private ensureStatus(...expected: AgentStatus[]): void {
    if (!expected.includes(this.status)) {
      throw new InvalidAgentStateError(this.status, expected)
    }
  }

  private addDomainEvent(event: DomainEvent): void {
    this.domainEvents.push(event)
  }
}

// domain/aggregates/AgentTeamAggregate.ts

/**
 * AgentTeamAggregate - Agent 团队聚合根
 *
 * 职责：
 * - 管理团队成员
 * - 协调任务分配
 * - 处理团队通信
 */
export class AgentTeamAggregate {
  private constructor(
    public readonly id: string,
    private name: string,
    private leaderId: AgentId,
    private members: Map<AgentId, AgentRole>,
    private status: 'active' | 'dissolved',
    private domainEvents: DomainEvent[] = []
  ) {}

  static create(name: string, leaderId: AgentId): AgentTeamAggregate {
    const team = new AgentTeamAggregate(
      TeamId.generate(),
      name,
      leaderId,
      new Map([[leaderId, AgentRole.LEADER]]),
      'active'
    )

    team.addDomainEvent(new TeamCreatedEvent(team.id, name, leaderId))

    return team
  }

  addMember(agentId: AgentId, role: AgentRole): void {
    if (this.status === 'dissolved') {
      throw new Error('Cannot add member to dissolved team')
    }

    this.members.set(agentId, role)
    this.addDomainEvent(new TeamMemberAddedEvent(this.id, agentId, role))
  }

  removeMember(agentId: AgentId): void {
    if (agentId.equals(this.leaderId)) {
      throw new Error('Cannot remove team leader')
    }

    this.members.delete(agentId)
  }

  dissolve(): void {
    this.status = 'dissolved'
    this.addDomainEvent(new TeamDissolvedEvent(this.id))
  }

  getMembers(): Array<{ id: AgentId; role: AgentRole }> {
    return Array.from(this.members.entries()).map(([id, role]) => ({ id, role }))
  }

  private addDomainEvent(event: DomainEvent): void {
    this.domainEvents.push(event)
  }
}
```

---

## 五、跨上下文集成设计

### 5.1 上下文映射

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           上下文映射图                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐                                    ┌─────────────┐    │
│  │    Query    │ ──────────────────────────────────►│    Tool     │    │
│  │             │   调用工具执行                      │             │    │
│  │             │   (ToolExecutionAggregate)         │             │    │
│  └─────────────┘                                    └─────────────┘    │
│         │                                                  │           │
│         │                                                  │           │
│         │                                                  ▼           │
│         │                                          ┌─────────────┐    │
│         │                                          │ Permission  │    │
│         │                                          │             │    │
│         │                                          │ 权限检查    │    │
│         │                                          └─────────────┘    │
│         │                                                  ▲           │
│         │                                                  │           │
│         ▼                                                  │           │
│  ┌─────────────┐                                    ┌─────────────┐    │
│  │   Session   │ ◄──────────────────────────────────│   Bridge    │    │
│  │             │   同步会话状态                      │             │    │
│  │             │   (SessionState)                   │             │    │
│  └─────────────┘                                    └─────────────┘    │
│         │                                                                  │
│         │                                                                  │
│         ▼                                                                  │
│  ┌─────────────┐                                    ┌─────────────┐    │
│  │   Memory    │                                    │   Agent     │    │
│  │             │                                    │             │    │
│  │ 持久化记忆  │                                    │ 子代理管理  │    │
│  └─────────────┘                                    └─────────────┘    │
│                                                             │           │
│                                                             │           │
│                                                             ▼           │
│                                                      ┌─────────────┐    │
│                                                      │    Query    │    │
│                                                      │             │    │
│                                                      │ 子代理查询  │    │
│                                                      └─────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 集成模式

#### 5.2.1 共享内核 (Shared Kernel)

```
src/shared-kernel/
├── types/
│   ├── ids.ts                    # 所有 ID 类型
│   │   ├── QueryId
│   │   ├── SessionId
│   │   ├── AgentId
│   │   ├── ToolName
│   │   └── ExecutionId
│   │
│   ├── errors.ts                 # 领域错误类型
│   │   ├── DomainError
│   │   ├── ValidationError
│   │   ├── StateError
│   │   └── PermissionError
│   │
│   ├── events.ts                 # 领域事件基类
│   │   └── DomainEvent
│   │
│   └── message.ts                # 消息类型
│       └── Message
│
├── value-objects/
│   ├── Timestamp.ts              # 时间戳值对象
│   ├── Json.ts                   # JSON 值对象
│   └── Result.ts                 # 结果类型
│
├── interfaces/
│   ├── IRepository.ts            # 仓储接口
│   ├── IEventBus.ts              # 事件总线接口
│   └── IUnitOfWork.ts            # 工作单元接口
│
└── DomainEvent.ts                # 领域事件基类
```

#### 5.2.2 防腐层 (Anti-Corruption Layer)

```typescript
// src/contexts/query/infrastructure/adapters/AnthropicApiAdapter.ts

import { QueryAggregate } from '../../domain/aggregates/QueryAggregate'
import { TokenUsage } from '../../domain/value-objects/TokenUsage'
import { StreamDelta } from '../../domain/value-objects/StreamDelta'

/**
 * AnthropicApiAdapter - Anthropic API 防腐层
 *
 * 职责：
 * - 将 Anthropic API 响应转换为领域模型
 * - 将领域模型转换为 API 请求
 * - 隔离外部 API 变化
 */
export class AnthropicApiAdapter implements IApiAdapter {
  /**
   * 将领域模型转换为 API 请求
   */
  toApiRequest(query: QueryAggregate): AnthropicRequest {
    return {
      model: query.model.toString(),
      messages: this.mapMessages(query.getMessages()),
      tools: this.mapTools(query.getAvailableTools()),
      max_tokens: query.getMaxTokens(),
      thinking: this.mapThinkingConfig(query.getThinkingConfig()),
    }
  }

  /**
   * 将 API 响应转换为领域模型
   */
  toDomain(apiResponse: AnthropicMessage): QueryResult {
    return QueryResult.from({
      content: this.mapContent(apiResponse.content),
      usage: this.mapUsage(apiResponse.usage),
      stopReason: apiResponse.stop_reason,
    })
  }

  /**
   * 将流式增量转换为领域模型
   */
  toStreamDelta(event: AnthropicStreamEvent): StreamDelta | null {
    switch (event.type) {
      case 'content_block_delta':
        return StreamDelta.text(event.delta.text)

      case 'content_block_start':
        return StreamDelta.blockStart(event.index, event.content_block)

      case 'content_block_stop':
        return StreamDelta.blockStop(event.index)

      case 'message_delta':
        return StreamDelta.usage(
          this.mapUsage(event.usage)
        )

      case 'message_start':
        return StreamDelta.messageStart({
          id: event.message.id,
          model: event.message.model,
        })

      case 'message_stop':
        return StreamDelta.messageStop()

      default:
        return null
    }
  }

  private mapMessages(messages: Message[]): AnthropicMessageParam[] {
    return messages.map(msg => {
      switch (msg.type) {
        case 'user':
          return {
            role: 'user',
            content: this.mapUserContent(msg.content),
          }
        case 'assistant':
          return {
            role: 'assistant',
            content: this.mapAssistantContent(msg.content),
          }
        default:
          throw new Error(`Unknown message type: ${msg.type}`)
      }
    })
  }

  private mapUsage(usage: AnthropicUsage): TokenUsage {
    return TokenUsage.from({
      inputTokens: usage.input_tokens,
      outputTokens: usage.output_tokens,
      cacheReadTokens: usage.cache_read_input_tokens,
      cacheWriteTokens: usage.cache_creation_input_tokens,
    })
  }
}
```

#### 5.2.3 开放主机服务 (Open Host Service)

```typescript
// src/api/ApiController.ts

import { QueryApplicationService } from '../contexts/query/application/services/QueryApplicationService'
import { ToolApplicationService } from '../contexts/tool/application/services/ToolApplicationService'
import { SessionApplicationService } from '../contexts/session/application/services/SessionApplicationService'

/**
 * ApiController - API 控制器
 *
 * 职责：
 * - 暴露统一的 API 接口
 * - 处理请求/响应转换
 * - 协调多个应用服务
 */
export class ApiController {
  constructor(
    private queryService: QueryApplicationService,
    private toolService: ToolApplicationService,
    private sessionService: SessionApplicationService
  ) {}

  /**
   * 提交查询
   */
  async postQuery(req: QueryRequest): Promise<QueryResponse> {
    const command = new SubmitQueryCommand({
      model: ModelId.from(req.model),
      prompt: req.prompt,
      thinkingConfig: ThinkingConfig.from(req.thinking),
      budget: Budget.from(req.budget),
    })

    const result = await this.queryService.submitQuery(command)

    return QueryResponse.from(result)
  }

  /**
   * 执行工具
   */
  async executeTool(req: ToolExecutionRequest): Promise<ToolExecutionResponse> {
    const command = new ExecuteToolCommand({
      toolName: ToolName.from(req.toolName),
      input: req.input,
    })

    const result = await this.toolService.executeTool(command)

    return ToolExecutionResponse.from(result)
  }

  /**
   * 创建会话
   */
  async createSession(req: CreateSessionRequest): Promise<SessionResponse> {
    const command = new CreateSessionCommand({
      config: SessionConfig.from(req.config),
    })

    const result = await this.sessionService.createSession(command)

    return SessionResponse.from(result)
  }
}
```

### 5.3 事件驱动集成

```typescript
// src/infrastructure/events/EventBus.ts

import { DomainEvent } from '../../shared-kernel/DomainEvent'

/**
 * IEventBus - 事件总线接口
 */
export interface IEventBus {
  publish(event: DomainEvent): Promise<void>
  subscribe<T extends DomainEvent>(
    eventType: string,
    handler: (event: T) => Promise<void>
  ): void
}

/**
 * InMemoryEventBus - 内存事件总线实现
 */
export class InMemoryEventBus implements IEventBus {
  private handlers: Map<string, Array<(event: DomainEvent) => Promise<void>>> = new Map()

  async publish(event: DomainEvent): Promise<void> {
    const handlers = this.handlers.get(event.eventType) ?? []

    await Promise.all(handlers.map(handler => handler(event)))
  }

  subscribe<T extends DomainEvent>(
    eventType: string,
    handler: (event: T) => Promise<void>
  ): void {
    const existing = this.handlers.get(eventType) ?? []
    this.handlers.set(eventType, [...existing, handler as any])
  }
}

// src/contexts/tool/application/handlers/ToolDomainEventHandler.ts

import { QueryCompletedEvent } from '../../../query/domain/events/QueryCompletedEvent'
import { IEventHandler } from '../../../../shared-kernel/IEventHandler'

/**
 * 监听查询完成事件，更新工具使用统计
 */
export class ToolDomainEventHandler implements IEventHandler<QueryCompletedEvent> {
  constructor(private toolRepository: IToolRepository) {}

  async handle(event: QueryCompletedEvent): Promise<void> {
    // 更新工具使用统计
    const tools = await this.toolRepository.findAll()

    for (const tool of tools) {
      // 更新统计信息
    }
  }
}
```

---

## 六、共享内核设计

### 6.1 类型定义

```typescript
// src/shared-kernel/types/ids.ts

import { randomUUID } from 'crypto'

/**
 * 基础 ID 类型
 */
export abstract class EntityId {
  protected constructor(protected readonly value: string) {}

  static generate(): EntityId {
    throw new Error('Subclass must implement generate()')
  }

  equals(other: EntityId): boolean {
    return this.value === other.value && this.constructor === other.constructor
  }

  toString(): string {
    return this.value
  }
}

/**
 * 查询 ID
 */
export class QueryId extends EntityId {
  static generate(): QueryId {
    return new QueryId(randomUUID())
  }

  static from(value: string): QueryId {
    return new QueryId(value)
  }
}

/**
 * 会话 ID
 */
export class SessionId extends EntityId {
  static generate(): SessionId {
    return new SessionId(randomUUID())
  }

  static from(value: string): SessionId {
    return new SessionId(value)
  }
}

/**
 * Agent ID
 */
export class AgentId extends EntityId {
  static generate(): AgentId {
    return new AgentId(`agent_${randomUUID()}`)
  }

  static from(value: string): AgentId {
    return new AgentId(value)
  }
}

// src/shared-kernel/types/errors.ts

/**
 * 领域错误基类
 */
export abstract class DomainError extends Error {
  constructor(
    message: string,
    public readonly code: string
  ) {
    super(message)
    this.name = this.constructor.name
  }
}

/**
 * 验证错误
 */
export class ValidationError extends DomainError {
  constructor(
    message: string,
    public readonly field?: string,
    public readonly value?: unknown
  ) {
    super(message, 'VALIDATION_ERROR')
  }
}

/**
 * 状态错误
 */
export class StateError extends DomainError {
  constructor(
    message: string,
    public readonly currentState: string,
    public readonly expectedStates: string[]
  ) {
    super(message, 'STATE_ERROR')
  }
}

/**
 * 权限错误
 */
export class PermissionError extends DomainError {
  constructor(message: string, public readonly reason: string) {
    super(message, 'PERMISSION_ERROR')
  }
}
```

### 6.2 领域事件基类

```typescript
// src/shared-kernel/DomainEvent.ts

/**
 * 领域事件接口
 */
export interface DomainEvent {
  readonly eventType: string
  readonly occurredOn: Date
  toPayload(): Record<string, unknown>
}

/**
 * 领域事件基类
 */
export abstract class BaseDomainEvent implements DomainEvent {
  abstract readonly eventType: string
  readonly occurredOn: Date

  constructor() {
    this.occurredOn = new Date()
  }

  abstract toPayload(): Record<string, unknown>
}
```

### 6.3 仓储接口

```typescript
// src/shared-kernel/interfaces/IRepository.ts

/**
 * 通用仓储接口
 */
export interface IRepository<T, Id> {
  save(entity: T): Promise<void>
  findById(id: Id): Promise<T | null>
  delete(id: Id): Promise<void>
}

/**
 * 支持查询的仓储接口
 */
export interface IQueryableRepository<T, Id> extends IRepository<T, Id> {
  findAll(): Promise<T[]>
  findBy(criteria: Partial<T>): Promise<T[]>
}
```

---

## 七、领域事件设计

### 7.1 事件分类

| 分类 | 前缀 | 示例 | 用途 |
|-----|------|------|------|
| 创建 | `*.created` | `query.created`, `session.created` | 实体创建通知 |
| 更新 | `*.updated` | `token_usage.updated` | 状态变更通知 |
| 完成 | `*.completed` | `query.completed`, `tool.execution.completed` | 流程完成通知 |
| 失败 | `*.failed` | `query.failed`, `agent.failed` | 错误处理 |
| 状态变更 | `*.status_changed` | `permission.mode_changed` | 状态机转换 |

### 7.2 事件定义规范

```typescript
// 事件命名规范
// 格式: {聚合名}.{过去式动词}
// 示例:
// - query.created
// - query.started
// - query.completed
// - query.failed
// - query.cancelled

// 事件载荷规范
interface EventPayload {
  // 必须包含聚合 ID
  aggregateId: string

  // 可选的相关 ID
  relatedId?: string

  // 事件特定数据
  [key: string]: unknown
}
```

### 7.3 事件处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                       事件处理流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 聚合执行业务行为                                             │
│     │                                                           │
│     ▼                                                           │
│  2. 聚合创建领域事件                                             │
│     │   - new QueryCompletedEvent(queryId, usage)              │
│     │   - addDomainEvent(event)                                │
│     │                                                           │
│     ▼                                                           │
│  3. 应用服务保存聚合                                             │
│     │   - await repository.save(query)                         │
│     │                                                           │
│     ▼                                                           │
│  4. 应用服务发布事件                                             │
│     │   - const events = query.getDomainEvents()               │
│     │   - for (event of events) await eventBus.publish(event) │
│     │   - query.clearDomainEvents()                            │
│     │                                                           │
│     ▼                                                           │
│  5. 事件总线分发事件                                             │
│     │   - 查找订阅者                                            │
│     │   - 并行调用处理器                                        │
│     │                                                           │
│     ▼                                                           │
│  6. 事件处理器处理事件                                           │
│         - 更新读模型                                            │
│         - 发送通知                                              │
│         - 触发其他业务流程                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、重构路线图

### 8.1 阶段概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        重构路线图                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  阶段一: 基础设施准备 (2 周)                                     │
│  ├── 创建限界上下文目录结构                                      │
│  ├── 提取共享内核类型                                            │
│  ├── 建立领域事件基础设施                                        │
│  └── 创建基础仓储接口                                            │
│                                                                 │
│  阶段二: Tool 上下文重构 (3 周)                                   │
│  ├── 提取 Tool 聚合和值对象                                      │
│  ├── 实现工具仓储                                                │
│  ├── 重构工具执行逻辑                                            │
│  └── 迁移权限检查逻辑                                            │
│                                                                 │
│  阶段三: Query 上下文重构 (4 周)                                  │
│  ├── 拆分 QueryEngine.ts 为多个领域对象                          │
│  ├── 实现 Query 聚合                                             │
│  ├── 重构流式处理逻辑                                            │
│  └── 迁移消息处理逻辑                                            │
│                                                                 │
│  阶段四: Session 上下文重构 (2 周)                                │
│  ├── 实现 Session 聚合                                           │
│  ├── 重构会话持久化                                              │
│  └── 迁移会话恢复逻辑                                            │
│                                                                 │
│  阶段五: 其他上下文重构 (4 周)                                    │
│  ├── Permission 上下文重构                                       │
│  ├── Agent 上下文重构                                            │
│  ├── Bridge 上下文重构                                           │
│  └── Plugin/Skill 上下文重构                                     │
│                                                                 │
│  阶段六: 清理与优化 (2 周)                                        │
│  ├── 删除遗留代码                                                │
│  ├── 优化依赖关系                                                │
│  ├── 完善文档                                                    │
│  └── 性能优化                                                    │
│                                                                 │
│  总计: 17 周                                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 详细任务分解

#### 阶段一：基础设施准备 (2 周)

| 任务 | 描述 | 预估时间 | 验收标准 |
|-----|------|---------|---------|
| 1.1 创建目录结构 | 创建各限界上下文目录 | 1 天 | 目录结构符合 DDD 规范 |
| 1.2 提取共享内核 | 提取 ID 类型、错误类型、事件基类 | 2 天 | 共享类型可被所有上下文使用 |
| 1.3 实现事件总线 | 实现内存事件总线 | 2 天 | 事件发布/订阅正常工作 |
| 1.4 创建仓储接口 | 定义通用仓储接口 | 1 天 | 所有上下文可使用统一接口 |
| 1.5 配置构建工具 | 更新 tsconfig、构建脚本 | 1 天 | 构建正常通过 |
| 1.6 编写测试框架 | 创建测试基础设施 | 2 天 | 领域对象测试可运行 |

#### 阶段二：Tool 上下文重构 (3 周)

| 任务 | 描述 | 预估时间 | 验收标准 |
|-----|------|---------|---------|
| 2.1 提取值对象 | ToolName, ToolSchema, ExecutionStatus 等 | 2 天 | 值对象不可变、可比较 |
| 2.2 实现 Tool 聚合 | ToolAggregate, ToolExecutionAggregate | 3 天 | 聚合行为正确、事件发布正常 |
| 2.3 实现工具仓储 | ToolRepository, ToolExecutionRepository | 2 天 | 仓储 CRUD 正常 |
| 2.4 重构工具执行器 | 各工具执行器实现 IToolExecutor | 5 天 | 所有工具通过新架构执行 |
| 2.5 迁移权限检查 | PermissionEvaluator, RuleMatcher | 3 天 | 权限检查逻辑正确 |
| 2.6 集成测试 | 端到端工具执行测试 | 2 天 | 所有工具功能正常 |

#### 阶段三：Query 上下文重构 (4 周)

| 任务 | 描述 | 预估时间 | 验收标准 |
|-----|------|---------|---------|
| 3.1 分析 QueryEngine | 分析职责边界、依赖关系 | 2 天 | 职责划分清晰 |
| 3.2 提取值对象 | QueryId, ModelId, TokenUsage 等 | 2 天 | 值对象完整 |
| 3.3 实现 Query 聚合 | QueryAggregate, QueryTurn | 4 天 | 聚合行为正确 |
| 3.4 实现流式处理 | StreamProcessor, StreamingAdapter | 4 天 | 流式响应处理正常 |
| 3.5 实现 API 适配器 | AnthropicApiAdapter, BedrockApiAdapter | 3 天 | API 调用正常 |
| 3.6 迁移消息处理 | MessageMapper, QueryOrchestrator | 3 天 | 消息处理正确 |
| 3.7 集成测试 | 端到端查询测试 | 3 天 | 查询功能正常 |

#### 阶段四：Session 上下文重构 (2 周)

| 任务 | 描述 | 预估时间 | 验收标准 |
|-----|------|---------|---------|
| 4.1 实现 Session 聚合 | SessionAggregate | 2 天 | 聚合行为正确 |
| 4.2 实现会话存储 | SessionStorage, SessionSerializer | 3 天 | 持久化正常 |
| 4.3 实现会话恢复 | ResumeSessionCommand | 2 天 | 恢复功能正常 |
| 4.4 实现会话压缩 | SessionCompactor | 2 天 | 压缩功能正常 |
| 4.5 集成测试 | 端到端会话测试 | 2 天 | 会话功能正常 |

#### 阶段五：其他上下文重构 (4 周)

| 任务 | 描述 | 预估时间 | 验收标准 |
|-----|------|---------|---------|
| 5.1 Permission 重构 | PermissionContextAggregate | 3 天 | 权限功能正常 |
| 5.2 Agent 重构 | AgentAggregate, AgentTeamAggregate | 5 天 | Agent 功能正常 |
| 5.3 Bridge 重构 | BridgeConnection, BridgeProtocol | 4 天 | Bridge 功能正常 |
| 5.4 Plugin 重构 | Plugin, PluginRegistry | 3 天 | 插件功能正常 |
| 5.5 Skill 重构 | Skill, SkillExecution | 3 天 | 技能功能正常 |
| 5.6 Memory 重构 | Memory, MemoryRepository | 2 天 | 记忆功能正常 |
| 5.7 集成测试 | 端到端集成测试 | 3 天 | 所有功能正常 |

#### 阶段六：清理与优化 (2 周)

| 任务 | 描述 | 预估时间 | 验收标准 |
|-----|------|---------|---------|
| 6.1 删除遗留代码 | 删除旧文件、清理导入 | 3 天 | 无遗留代码 |
| 6.2 优化依赖关系 | 解决循环依赖、优化导入 | 2 天 | 无循环依赖 |
| 6.3 完善文档 | API 文档、架构文档 | 3 天 | 文档完整 |
| 6.4 性能优化 | 性能分析、优化热点 | 3 天 | 性能达标 |
| 6.5 最终测试 | 全面回归测试 | 2 天 | 所有测试通过 |

---

## 九、设计决策记录

### ADR-001: 选择限界上下文边界

**状态**：已接受

**背景**：
Claude Code 是一个复杂的 CLI 工具，包含工具执行、查询处理、会话管理等多个功能模块。需要确定如何划分限界上下文边界。

**决策**：
按照业务能力划分限界上下文，而非技术层。每个上下文包含完整的领域层、应用层、基础设施层和接口层。

**理由**：
1. 业务能力划分更符合 DDD 原则
2. 各上下文可独立演进
3. 便于团队分工
4. 降低耦合度

**替代方案**：
- 按技术层划分（UI 层、业务层、数据层）：会导致层间耦合，不符合 DDD 原则
- 按模块划分（保持现有结构）：无法解决职责混乱问题

**影响**：
- 需要重新组织目录结构
- 需要建立上下文间的集成机制
- 初期工作量较大，但长期收益明显

---

### ADR-002: 聚合设计原则

**状态**：已接受

**背景**：
需要确定聚合的大小和边界，以平衡一致性和性能。

**决策**：
采用小聚合策略，每个聚合只包含必要的实体和值对象。跨聚合通过 ID 引用。

**理由**：
1. 小聚合性能更好
2. 并发冲突更少
3. 事务边界更清晰
4. 符合 DDD 最佳实践

**聚合设计规则**：
1. 聚合内强一致性，聚合间最终一致性
2. 外部只能通过聚合根访问聚合内对象
3. 一个事务只修改一个聚合实例
4. 聚合间通过 ID 引用，不直接持有对象引用

**示例**：
```typescript
// 正确：通过 ID 引用
class QueryAggregate {
  private sessionSessionId: SessionId  // ID 引用
}

// 错误：直接持有对象引用
class QueryAggregate {
  private session: SessionAggregate  // 对象引用
}
```

---

### ADR-003: 状态管理策略

**状态**：已接受

**背景**：
现有 React 状态与领域状态混合，需要统一管理策略。

**决策**：
- 领域状态：通过聚合根管理
- UI 状态：继续使用 React 状态
- 同步：通过领域事件同步

**理由**：
1. 关注点分离
2. 领域逻辑独立于 UI
3. 便于测试
4. 状态变更可追踪

**实现方式**：
```typescript
// 领域状态变更
query.startTurn(userMessage)

// 发布事件
eventBus.publish(new TurnStartedEvent(query.id, turn.id))

// UI 订阅事件更新状态
eventBus.subscribe('turn.started', (event) => {
  setAppState(prev => ({
    ...prev,
    currentTurn: event.turnId,
  }))
})
```

---

### ADR-004: 条件编译处理

**状态**：已接受

**背景**：
现有代码中 `feature('XXX')` 散布在各处，影响可读性。

**决策**：
将条件编译逻辑集中到 Feature Flag 服务，通过依赖注入控制行为。

**理由**：
1. 集中管理特性开关
2. 便于测试（可模拟开关状态）
3. 减少代码散布

**实现方式**：
```typescript
// 特性开关服务
interface IFeatureFlagService {
  isEnabled(feature: string): boolean
}

// 通过依赖注入使用
class ToolApplicationService {
  constructor(private featureFlags: IFeatureFlagService) {}

  getAvailableTools(): Tools {
    const tools = [BashTool, FileReadTool]

    if (this.featureFlags.isEnabled('AGENT_TRIGGERS')) {
      tools.push(...cronTools)
    }

    return tools
  }
}
```

---

## 十、风险评估与缓解措施

### 10.1 风险矩阵

| 风险 | 概率 | 影响 | 风险等级 | 缓解措施 |
|-----|------|------|---------|---------|
| 重构影响现有功能 | 高 | 高 | **高** | 渐进式重构、保持向后兼容、全面测试 |
| 学习曲线 | 中 | 中 | **中** | 提供培训、编写文档、代码审查 |
| 性能下降 | 中 | 中 | **中** | 性能基准测试、按需优化 |
| 团队抵触 | 中 | 中 | **中** | 演示收益、收集反馈、渐进式推进 |
| 时间超期 | 高 | 中 | **中** | 预留缓冲时间、分阶段交付 |
| 依赖冲突 | 低 | 高 | **中** | 依赖分析、版本锁定、渐进式迁移 |

### 10.2 缓解措施详情

#### 风险 1：重构影响现有功能

**缓解措施**：
1. **渐进式重构**：每次只重构一个上下文，保持其他部分不变
2. **向后兼容**：保留旧接口，新接口并行运行
3. **全面测试**：每个阶段完成后运行完整测试套件
4. **回滚机制**：每个阶段独立分支，可随时回滚

#### 风险 2：学习曲线

**缓解措施**：
1. **培训计划**：组织 DDD 培训课程
2. **文档编写**：编写详细的架构文档和代码示例
3. **代码审查**：重构代码必须经过审查
4. **结对编程**：新手与有经验者结对

#### 风险 3：性能下降

**缓解措施**：
1. **性能基准**：建立性能基准测试
2. **持续监控**：每个阶段完成后进行性能测试
3. **优化策略**：识别热点、按需优化
4. **缓存策略**：合理使用缓存

---

## 十一、附录

### 11.1 术语表

| 术语 | 定义 |
|-----|------|
| 限界上下文 (Bounded Context) | 明确的边界，边界内的领域模型保持一致 |
| 聚合 (Aggregate) | 一组相关对象的集合，作为数据修改的单元 |
| 聚合根 (Aggregate Root) | 聚合的入口点，对外代表整个聚合 |
| 值对象 (Value Object) | 没有唯一标识，通过属性值判断相等 |
| 实体 (Entity) | 有唯一标识，通过 ID 判断相等 |
| 领域事件 (Domain Event) | 表达领域内发生的业务事件 |
| 仓储 (Repository) | 封装数据访问的抽象 |
| 防腐层 (Anti-Corruption Layer) | 隔离外部系统的适配层 |

### 11.2 参考资料

1. 《领域驱动设计》- Eric Evans
2. 《实现领域驱动设计》- Vaughn Vernon
3. 《领域驱动设计精粹》- Vaughn Vernon
4. DDD 实战课程
5. Claude Code 源码分析

### 11.3 变更历史

| 版本 | 日期 | 作者 | 变更内容 |
|-----|------|------|---------|
| 1.0 | 2026-04-01 | Claude | 初始版本 |

---

**文档结束**
