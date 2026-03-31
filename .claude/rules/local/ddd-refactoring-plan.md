# Claude Code DDD 改造设计文档

> 本文档分析 Claude Code 源码的现有架构，并提出基于 DDD（领域驱动设计）的重构方案。

---

## 一、现有架构分析

### 1.1 项目概述

Claude Code 是 Anthropic 开发的 CLI 工具，用于在终端中与 Claude 交互，执行软件工程任务。

**技术栈**：
- 运行时：Bun
- 语言：TypeScript (strict)
- 终端 UI：React + Ink
- CLI 解析：Commander.js
- Schema 验证：Zod v4
- 代码搜索：ripgrep

**规模**：~1,900 文件，512,000+ 行代码

### 1.2 现有目录结构

```
src/
├── main.tsx                 # 入口编排 (Commander.js)
├── commands.ts              # 命令注册 (~25K 行)
├── tools.ts                 # 工具注册 (~17K 行)
├── Tool.ts                  # 工具类型定义 (~29K 行)
├── QueryEngine.ts           # LLM 查询引擎 (~46K 行)
├── context.ts               # 系统/用户上下文收集
├── cost-tracker.ts          # Token 成本追踪
│
├── commands/                # 斜杠命令实现 (~50 个)
├── tools/                   # Agent 工具实现 (~40 个)
├── components/              # Ink UI 组件 (~140 个)
├── hooks/                   # React hooks
├── services/                # 外部服务集成
├── screens/                 # 全屏 UI (Doctor, REPL, Resume)
├── types/                   # TypeScript 类型定义
├── utils/                   # 工具函数
│
├── bridge/                  # IDE 和远程控制桥接
├── coordinator/             # 多 Agent 协调器
├── plugins/                 # 插件系统
├── skills/                  # 技能系统
├── keybindings/             # 快捷键配置
├── vim/                     # Vim 模式
├── voice/                   # 语音输入
├── remote/                  # 远程会话
├── server/                  # 服务器模式
├── memdir/                  # 持久化内存目录
├── tasks/                   # 任务管理
├── state/                   # 状态管理
├── migrations/              # 配置迁移
├── schemas/                 # 配置 Schema (Zod)
├── entrypoints/             # 初始化逻辑
├── ink/                     # Ink 渲染器包装
├── buddy/                   # 伴侣精灵
├── native-ts/               # 原生 TS 工具
├── outputStyles/            # 输出样式
├── query/                   # 查询管道
└── upstreamproxy/           # 代理配置
```

### 1.3 现有架构问题

#### 问题 1：巨型文件

| 文件 | 行数 | 问题 |
|-----|------|------|
| QueryEngine.ts | ~46K | 查询引擎包含过多职责 |
| Tool.ts | ~29K | 类型定义与业务逻辑混杂 |
| commands.ts | ~25K | 命令注册包含过多实现细节 |
| tools.ts | ~17K | 工具注册包含条件编译逻辑 |

#### 问题 2：职责不清

- **Tool.ts**：混合了类型定义、权限上下文、进度状态
- **QueryEngine.ts**：混合了查询生命周期、消息处理、状态管理
- **services/**：服务层缺乏统一抽象，各服务实现风格不一致

#### 问题 3：依赖混乱

- 循环依赖：tools.ts ↔ TeamCreateTool/TeamDeleteTool
- 条件编译逻辑散布在各处（`feature('XXX')`）
- 领域逻辑与技术实现耦合

#### 问题 4：状态管理分散

- AppState、FileStateCache、PermissionContext 等状态分散
- React 状态与领域状态混合
- 缺乏统一的状态变更追踪

---

## 二、DDD 限界上下文划分

### 2.1 核心限界上下文

基于业务分析，识别以下限界上下文：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Claude Code 限界上下文地图                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   Conversation  │  │     Tool        │  │    Command      │ │
│  │   会话上下文     │  │     工具执行     │  │    命令处理     │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘ │
│           │                    │                    │          │
│           └────────────────────┼────────────────────┘          │
│                                │                               │
│  ┌─────────────────────────────┴─────────────────────────────┐ │
│  │                    Query (查询)                             │ │
│  │                    LLM 交互核心                             │ │
│  └─────────────────────────────┬─────────────────────────────┘ │
│                                │                               │
│  ┌─────────────────────────────┴─────────────────────────────┐ │
│  │                    Session (会话)                           │ │
│  │                    会话生命周期管理                          │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   Permission    │  │     Bridge      │  │     Plugin      │ │
│  │   权限控制       │  │     IDE 桥接     │  │     插件系统     │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │     Skill       │  │     Agent       │  │     Memory      │ │
│  │     技能系统     │  │     子代理       │  │     持久化记忆   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 各限界上下文职责

| 限界上下文 | 核心职责 | 关键聚合 |
|-----------|---------|---------|
| **Conversation** | 管理对话消息、上下文压缩 | Message, Context |
| **Tool** | 工具注册、执行、权限检查 | Tool, ToolExecution, Permission |
| **Command** | 斜杠命令注册与执行 | Command, CommandExecution |
| **Query** | LLM API 调用、流式响应处理 | Query, QueryResult |
| **Session** | 会话生命周期、状态持久化 | Session, SessionState |
| **Permission** | 权限规则、用户授权 | PermissionRule, Authorization |
| **Bridge** | IDE 扩展通信、远程控制 | BridgeConnection, BridgeMessage |
| **Plugin** | 插件加载、生命周期管理 | Plugin, PluginRegistry |
| **Skill** | 技能定义、执行编排 | Skill, SkillExecution |
| **Agent** | 子代理创建、协调、通信 | Agent, AgentTeam, AgentMessage |
| **Memory** | 持久化记忆、团队同步 | Memory, MemoryEntry |

---

## 三、各限界上下文详细设计

### 3.1 Query 限界上下文

**领域层**：

```
src/contexts/query/
├── domain/
│   ├── entities/
│   │   ├── Query.ts              # 查询实体
│   │   ├── QueryResult.ts        # 查询结果实体
│   │   └── QueryTurn.ts          # 查询轮次实体
│   ├── value-objects/
│   │   ├── ModelId.ts            # 模型标识值对象
│   │   ├── TokenUsage.ts         # Token 使用量值对象
│   │   ├── QueryStatus.ts        # 查询状态枚举
│   │   └── StreamingState.ts     # 流式状态值对象
│   ├── aggregates/
│   │   └── QueryAggregate.ts     # 查询聚合根
│   ├── events/
│   │   ├── QueryStartedEvent.ts
│   │   ├── QueryCompletedEvent.ts
│   │   ├── TokenUsageUpdatedEvent.ts
│   │   └── QueryErrorEvent.ts
│   ├── services/
│   │   └── QueryOrchestrator.ts  # 查询编排领域服务
│   └── repositories/
│       └── IQueryRepository.ts   # 查询仓储接口
├── application/
│   ├── services/
│   │   └── QueryApplicationService.ts
│   ├── commands/
│   │   ├── SubmitQueryCommand.ts
│   │   ├── CancelQueryCommand.ts
│   │   └── RetryQueryCommand.ts
│   ├── queries/
│   │   ├── GetQueryStatusQuery.ts
│   │   └── GetTokenUsageQuery.ts
│   └── handlers/
│       ├── SubmitQueryHandler.ts
│       └── QueryEventHandler.ts
├── infrastructure/
│   ├── repositories/
│   │   └── QueryRepository.ts    # 查询仓储实现
│   ├── adapters/
│   │   ├── AnthropicApiAdapter.ts
│   │   └── StreamingAdapter.ts
│   └── mappers/
│       └── QueryMapper.ts
└── interfaces/
    ├── QueryController.ts
    └── dto/
        ├── QueryRequestDto.ts
        └── QueryResponseDto.ts
```

**核心聚合设计**：

```typescript
// domain/aggregates/QueryAggregate.ts
export class QueryAggregate {
  private constructor(
    public readonly id: QueryId,
    private messages: Message[],
    private status: QueryStatus,
    private tokenUsage: TokenUsage,
    private model: ModelId,
    private thinkingConfig: ThinkingConfig,
    private domainEvents: DomainEvent[] = []
  ) {}

  // 工厂方法
  static create(config: QueryConfig): QueryAggregate {
    const query = new QueryAggregate(
      QueryId.generate(),
      [],
      QueryStatus.PENDING,
      TokenUsage.empty(),
      config.model,
      config.thinkingConfig
    )
    query.addDomainEvent(new QueryCreatedEvent(query.id))
    return query
  }

  // 业务行为
  submitMessage(message: UserMessage): void {
    this.ensureStatus(QueryStatus.READY)
    this.messages.push(message)
    this.addDomainEvent(new MessageSubmittedEvent(this.id, message.id))
  }

  startStreaming(): void {
    this.ensureStatus(QueryStatus.PENDING)
    this.status = QueryStatus.STREAMING
    this.addDomainEvent(new QueryStartedEvent(this.id))
  }

  handleStreamDelta(delta: StreamDelta): void {
    this.ensureStatus(QueryStatus.STREAMING)
    // 处理流式增量
  }

  complete(result: QueryResult): void {
    this.ensureStatus(QueryStatus.STREAMING)
    this.status = QueryStatus.COMPLETED
    this.tokenUsage = result.usage
    this.addDomainEvent(new QueryCompletedEvent(this.id, result))
  }

  fail(error: QueryError): void {
    this.status = QueryStatus.FAILED
    this.addDomainEvent(new QueryErrorEvent(this.id, error))
  }

  cancel(): void {
    if (this.status === QueryStatus.STREAMING) {
      this.status = QueryStatus.CANCELLED
      this.addDomainEvent(new QueryCancelledEvent(this.id))
    }
  }

  // 不变性约束
  private ensureStatus(expected: QueryStatus): void {
    if (this.status !== expected) {
      throw new InvalidQueryStateError(this.status, expected)
    }
  }
}
```

### 3.2 Tool 限界上下文

**领域层**：

```
src/contexts/tool/
├── domain/
│   ├── entities/
│   │   ├── Tool.ts                # 工具实体
│   │   ├── ToolExecution.ts       # 工具执行实体
│   │   └── ToolResult.ts          # 工具结果实体
│   ├── value-objects/
│   │   ├── ToolName.ts            # 工具名称值对象
│   │   ├── ToolSchema.ts          # 工具 Schema 值对象
│   │   ├── ExecutionStatus.ts     # 执行状态枚举
│   │   └── PermissionDecision.ts  # 权限决策值对象
│   ├── aggregates/
│   │   ├── ToolAggregate.ts       # 工具聚合根
│   │   └── ToolExecutionAggregate.ts
│   ├── events/
│   │   ├── ToolRegisteredEvent.ts
│   │   ├── ToolExecutionStartedEvent.ts
│   │   ├── ToolExecutionCompletedEvent.ts
│   │   └── PermissionRequiredEvent.ts
│   ├── services/
│   │   ├── ToolRegistry.ts        # 工具注册表领域服务
│   │   └── PermissionEvaluator.ts # 权限评估领域服务
│   └── repositories/
│       ├── IToolRepository.ts
│       └── IToolExecutionRepository.ts
├── application/
│   ├── services/
│   │   └── ToolApplicationService.ts
│   ├── commands/
│   │   ├── RegisterToolCommand.ts
│   │   ├── ExecuteToolCommand.ts
│   │   └── CancelToolExecutionCommand.ts
│   └── queries/
│       ├── GetToolByNameQuery.ts
│       └── ListAvailableToolsQuery.ts
├── infrastructure/
│   ├── repositories/
│   │   └── ToolRepository.ts
│   ├── executors/
│   │   ├── BashToolExecutor.ts
│   │   ├── FileReadToolExecutor.ts
│   │   ├── FileWriteToolExecutor.ts
│   │   └── ...                    # 各工具执行器
│   └── adapters/
│       └── McpToolAdapter.ts      # MCP 工具适配器
└── interfaces/
    ├── ToolController.ts
    └── dto/
        ├── ToolExecutionRequestDto.ts
        └── ToolExecutionResponseDto.ts
```

**工具聚合设计**：

```typescript
// domain/aggregates/ToolAggregate.ts
export class ToolAggregate {
  private constructor(
    public readonly name: ToolName,
    public readonly schema: ToolSchema,
    private permissionRules: PermissionRule[],
    private isEnabled: boolean,
    private domainEvents: DomainEvent[] = []
  ) {}

  static register(definition: ToolDefinition): ToolAggregate {
    const tool = new ToolAggregate(
      ToolName.from(definition.name),
      ToolSchema.from(definition.inputSchema),
      [],
      true
    )
    tool.addDomainEvent(new ToolRegisteredEvent(tool.name))
    return tool
  }

  canExecute(context: ToolPermissionContext): PermissionDecision {
    const evaluator = new PermissionEvaluator(this.permissionRules)
    return evaluator.evaluate(context)
  }

  execute(input: ToolInput, context: ToolUseContext): ToolExecution {
    const decision = this.canExecute(context.toolPermissionContext)

    if (decision.isDenied) {
      return ToolExecution.denied(decision.reason)
    }

    if (decision.requiresApproval) {
      this.addDomainEvent(new PermissionRequiredEvent(this.name, input))
      return ToolExecution.pendingApproval()
    }

    return ToolExecution.start(this.name, input, context)
  }

  addPermissionRule(rule: PermissionRule): void {
    this.permissionRules.push(rule)
  }

  disable(): void {
    this.isEnabled = false
  }
}

// domain/entities/ToolExecution.ts
export class ToolExecution {
  private constructor(
    public readonly id: ExecutionId,
    public readonly toolName: ToolName,
    public readonly input: ToolInput,
    private status: ExecutionStatus,
    private result?: ToolResult,
    private startedAt: Date,
    private completedAt?: Date
  ) {}

  static start(name: ToolName, input: ToolInput, context: ToolUseContext): ToolExecution {
    return new ToolExecution(
      ExecutionId.generate(),
      name,
      input,
      ExecutionStatus.RUNNING,
      undefined,
      new Date()
    )
  }

  complete(result: ToolResult): void {
    this.ensureStatus(ExecutionStatus.RUNNING)
    this.status = ExecutionStatus.COMPLETED
    this.result = result
    this.completedAt = new Date()
  }

  fail(error: ToolError): void {
    this.status = ExecutionStatus.FAILED
    this.result = ToolResult.fromError(error)
    this.completedAt = new Date()
  }
}
```

### 3.3 Session 限界上下文

**领域层**：

```
src/contexts/session/
├── domain/
│   ├── entities/
│   │   ├── Session.ts             # 会话实体
│   │   └── SessionState.ts        # 会话状态实体
│   ├── value-objects/
│   │   ├── SessionId.ts           # 会话 ID 值对象
│   │   ├── SessionStatus.ts       # 会话状态枚举
│   │   └── SessionMetadata.ts     # 会话元数据值对象
│   ├── aggregates/
│   │   └── SessionAggregate.ts    # 会话聚合根
│   ├── events/
│   │   ├── SessionCreatedEvent.ts
│   │   ├── SessionResumedEvent.ts
│   │   ├── SessionArchivedEvent.ts
│   │   └── SessionStateSavedEvent.ts
│   ├── services/
│   │   └── SessionManager.ts      # 会话管理领域服务
│   └── repositories/
│       └── ISessionRepository.ts
├── application/
│   ├── services/
│   │   └── SessionApplicationService.ts
│   ├── commands/
│   │   ├── CreateSessionCommand.ts
│   │   ├── ResumeSessionCommand.ts
│   │   ├── ArchiveSessionCommand.ts
│   │   └── SaveSessionStateCommand.ts
│   └── queries/
│       ├── GetSessionByIdQuery.ts
│       └── ListRecentSessionsQuery.ts
├── infrastructure/
│   ├── repositories/
│   │   └── SessionRepository.ts
│   ├── storage/
│   │   ├── SessionStorage.ts
│   │   └── SessionSerializer.ts
│   └── adapters/
│       └── SessionBackupAdapter.ts
└── interfaces/
    ├── SessionController.ts
    └── dto/
        ├── SessionDto.ts
        └── SessionStateDto.ts
```

### 3.4 Permission 限界上下文

**领域层**：

```
src/contexts/permission/
├── domain/
│   ├── entities/
│   │   ├── PermissionRule.ts      # 权限规则实体
│   │   └── Authorization.ts       # 授权实体
│   ├── value-objects/
│   │   ├── PermissionMode.ts      # 权限模式枚举
│   │   ├── PermissionDecision.ts  # 权限决策值对象
│   │   └── RuleScope.ts           # 规则范围值对象
│   ├── aggregates/
│   │   └── PermissionContextAggregate.ts
│   ├── events/
│   │   ├── PermissionGrantedEvent.ts
│   │   ├── PermissionDeniedEvent.ts
│   │   └── PermissionModeChangedEvent.ts
│   ├── services/
│   │   ├── RuleMatcher.ts         # 规则匹配领域服务
│   │   └── AuthorizationService.ts
│   └── repositories/
│       └── IPermissionRuleRepository.ts
├── application/
│   ├── services/
│   │   └── PermissionApplicationService.ts
│   ├── commands/
│   │   ├── GrantPermissionCommand.ts
│   │   ├── DenyPermissionCommand.ts
│   │   └── SetPermissionModeCommand.ts
│   └── queries/
│       └── GetPermissionContextQuery.ts
├── infrastructure/
│   ├── repositories/
│   │   └── PermissionRuleRepository.ts
│   ├── evaluators/
│   │   ├── RuleEvaluator.ts
│   │   └── BypassModeEvaluator.ts
│   └── adapters/
│       └── HookPermissionAdapter.ts
└── interfaces/
    ├── PermissionController.ts
    └── dto/
        ├── PermissionRequestDto.ts
        └── PermissionResponseDto.ts
```

### 3.5 Agent 限界上下文

**领域层**：

```
src/contexts/agent/
├── domain/
│   ├── entities/
│   │   ├── Agent.ts               # Agent 实体
│   │   ├── AgentTeam.ts           # Agent 团队实体
│   │   └── AgentMessage.ts        # Agent 消息实体
│   ├── value-objects/
│   │   ├── AgentId.ts             # Agent ID 值对象
│   │   ├── AgentRole.ts           # Agent 角色枚举
│   │   ├── AgentStatus.ts         # Agent 状态枚举
│   │   └── AgentCapability.ts     # Agent 能力值对象
│   ├── aggregates/
│   │   ├── AgentAggregate.ts      # Agent 聚合根
│   │   └── AgentTeamAggregate.ts  # Agent 团队聚合根
│   ├── events/
│   │   ├── AgentSpawnedEvent.ts
│   │   ├── AgentCompletedEvent.ts
│   │   ├── AgentMessageSentEvent.ts
│   │   └── TeamCreatedEvent.ts
│   ├── services/
│   │   ├── AgentCoordinator.ts    # Agent 协调领域服务
│   │   └── TaskDistributor.ts     # 任务分配领域服务
│   └── repositories/
│       ├── IAgentRepository.ts
│       └── IAgentTeamRepository.ts
├── application/
│   ├── services/
│   │   └── AgentApplicationService.ts
│   ├── commands/
│   │   ├── SpawnAgentCommand.ts
│   │   ├── SendMessageToAgentCommand.ts
│   │   ├── CreateTeamCommand.ts
│   │   └── TerminateAgentCommand.ts
│   └── queries/
│       ├── GetAgentStatusQuery.ts
│       └── ListTeamMembersQuery.ts
├── infrastructure/
│   ├── repositories/
│   │   └── AgentRepository.ts
│   ├── coordinators/
│   │   └── CoordinatorModeCoordinator.ts
│   └── adapters/
│       └── SubagentContextAdapter.ts
└── interfaces/
    ├── AgentController.ts
    └── dto/
        ├── AgentDto.ts
        └── AgentTeamDto.ts
```

---

## 四、跨上下文集成设计

### 4.1 上下文映射

```
┌─────────────────────────────────────────────────────────────────┐
│                        上下文映射                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Query ──────────────────────────────────────────► Tool         │
│    │                                                  │         │
│    │  调用工具执行                                    │         │
│    │  (ToolExecutionAggregate)                       │         │
│    │                                                  │         │
│    └──────────────────────────────────────────────────┘         │
│                                                                 │
│  Query ◄────────────────────────────────────────── Session      │
│    │                                                  │         │
│    │  恢复会话时加载消息                              │         │
│    │  (SessionAggregate.messages)                    │         │
│    │                                                  │         │
│    └──────────────────────────────────────────────────┘         │
│                                                                 │
│  Tool ────────────────────────────────────────────► Permission  │
│    │                                                  │         │
│    │  请求权限检查                                    │         │
│    │  (PermissionDecision)                           │         │
│    │                                                  │         │
│    └──────────────────────────────────────────────────┘         │
│                                                                 │
│  Agent ───────────────────────────────────────────► Query       │
│    │                                                  │         │
│    │  子代理执行查询                                  │         │
│    │  (QueryAggregate)                               │         │
│    │                                                  │         │
│    └──────────────────────────────────────────────────┘         │
│                                                                 │
│  Bridge ◄─────────────────────────────────────────► Session     │
│    │                                                  │         │
│    │  同步会话状态                                    │         │
│    │  (SessionState)                                 │         │
│    │                                                  │         │
│    └──────────────────────────────────────────────────┘         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 集成模式

#### 4.2.1 共享内核 (Shared Kernel)

**共享类型定义**：

```
src/shared-kernel/
├── types/
│   ├── ids.ts                    # 所有 ID 类型
│   ├── errors.ts                 # 领域错误类型
│   └── events.ts                 # 领域事件基类
├── value-objects/
│   ├── Timestamp.ts              # 时间戳值对象
│   └── Json.ts                   # JSON 值对象
└── DomainEvent.ts                # 领域事件基类
```

#### 4.2.2 防腐层 (Anti-Corruption Layer)

**外部服务适配**：

```typescript
// src/contexts/query/infrastructure/adapters/AnthropicApiAdapter.ts
export class AnthropicApiAdapter {
  // 将 Anthropic API 响应转换为领域模型
  toDomain(apiResponse: AnthropicMessage): QueryResult {
    return QueryResult.from({
      content: this.mapContent(apiResponse.content),
      usage: this.mapUsage(apiResponse.usage),
      stopReason: apiResponse.stop_reason
    })
  }

  // 将领域模型转换为 API 请求
  toApi(query: QueryAggregate): AnthropicRequest {
    return {
      model: query.model.value,
      messages: this.mapMessages(query.messages),
      tools: this.mapTools(query.availableTools),
      max_tokens: query.maxTokens
    }
  }
}
```

#### 4.2.3 开放主机服务 (Open Host Service)

**API 层设计**：

```typescript
// src/api/ApiController.ts
export class ApiController {
  constructor(
    private queryService: QueryApplicationService,
    private toolService: ToolApplicationService,
    private sessionService: SessionApplicationService
  ) {}

  // RESTful API 端点
  async postQuery(req: QueryRequest): Promise<QueryResponse> {
    const command = new SubmitQueryCommand(req)
    const result = await this.queryService.execute(command)
    return QueryResponse.from(result)
  }

  async getTool(name: string): Promise<ToolResponse> {
    const query = new GetToolByNameQuery(name)
    const tool = await this.toolService.query(query)
    return ToolResponse.from(tool)
  }
}
```

---

## 五、重构路线图

### 5.1 阶段一：基础设施准备 (2 周)

**目标**：建立 DDD 架构基础，不改变现有功能

**任务**：
1. 创建限界上下文目录结构
2. 提取共享内核类型
3. 建立领域事件基础设施
4. 创建基础仓储接口

**验收标准**：
- [ ] 所有限界上下文目录创建完成
- [ ] 共享类型定义提取完成
- [ ] 领域事件发布/订阅机制可用
- [ ] 现有功能不受影响

### 5.2 阶段二：Tool 上下文重构 (3 周)

**目标**：将工具系统重构为 DDD 架构

**任务**：
1. 提取 Tool 聚合和值对象
2. 实现工具仓储
3. 重构工具执行逻辑
4. 迁移权限检查逻辑

**验收标准**：
- [ ] 所有工具通过新架构注册
- [ ] 工具执行通过聚合根进行
- [ ] 权限检查通过领域服务
- [ ] 现有工具功能正常

### 5.3 阶段三：Query 上下文重构 (4 周)

**目标**：将查询引擎重构为 DDD 架构

**任务**：
1. 拆分 QueryEngine.ts 为多个领域对象
2. 实现 Query 聚合
3. 重构流式处理逻辑
4. 迁移消息处理逻辑

**验收标准**：
- [ ] QueryEngine.ts 拆分为多个小文件
- [ ] 查询生命周期通过聚合根管理
- [ ] 流式响应处理正常
- [ ] 所有查询功能正常

### 5.4 阶段四：Session 上下文重构 (2 周)

**目标**：建立会话管理 DDD 架构

**任务**：
1. 实现 Session 聚合
2. 重构会话持久化
3. 迁移会话恢复逻辑

**验收标准**：
- [ ] 会话生命周期通过聚合根管理
- [ ] 会话持久化正常
- [ ] 会话恢复功能正常

### 5.5 阶段五：其他上下文重构 (4 周)

**目标**：完成剩余限界上下文重构

**任务**：
1. Permission 上下文重构
2. Agent 上下文重构
3. Bridge 上下文重构
4. Plugin 上下文重构
5. Skill 上下文重构

### 5.6 阶段六：清理与优化 (2 周)

**目标**：清理遗留代码，优化架构

**任务**：
1. 删除遗留代码
2. 优化依赖关系
3. 完善文档
4. 性能优化

---

## 六、设计决策记录

### ADR-001: 选择限界上下文边界

**状态**：已接受

**背景**：
Claude Code 是一个复杂的 CLI 工具，包含工具执行、查询处理、会话管理等多个功能模块。

**决策**：
按照业务能力划分限界上下文，而非技术层。每个上下文包含完整的领域层、应用层、基础设施层和接口层。

**理由**：
1. 业务能力划分更符合 DDD 原则
2. 各上下文可独立演进
3. 便于团队分工
4. 降低耦合度

### ADR-002: 聚合设计原则

**状态**：已接受

**背景**：
需要确定聚合的大小和边界。

**决策**：
采用小聚合策略，每个聚合只包含必要的实体和值对象。跨聚合通过 ID 引用。

**理由**：
1. 小聚合性能更好
2. 并发冲突更少
3. 事务边界更清晰
4. 符合 DDD 最佳实践

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

---

## 七、风险与缓解措施

| 风险 | 影响 | 缓解措施 |
|-----|------|---------|
| 重构影响现有功能 | 高 | 渐进式重构，保持向后兼容 |
| 学习曲线 | 中 | 提供培训和文档 |
| 性能下降 | 中 | 性能测试，按需优化 |
| 团队抵触 | 中 | 演示收益，收集反馈 |

---

## 八、参考资源

- 《领域驱动设计》- Eric Evans
- 《实现领域驱动设计》- Vaughn Vernon
- DDD 实战课程
- 项目现有代码库分析
