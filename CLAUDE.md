# CLAUDE.md

> 此文件是 Claude Code 在本项目的**指令入口**，保持精简（~150 行）。
> 原则：指令，不是百科全书。Claude Code 会自动读取此文件。

## 指令加载方式

Claude Code 自动加载以下内容：

```
CLAUDE.md                    # 项目级指令（本文件）
.claude/rules/               # 规则文件目录
   ├─ common/                # 通用规则（跨项目共享）
   └─ local/                 # 本地规则（项目特定）
       ├─ ddd-design.md      # DDD 设计指南
       ├─ naming.md          # 命名约定
       ├─ framework.md       # 框架使用模板
       └─ api.md             # API 开发模板
```

## 项目概述

[项目维护者在此描述项目目的、核心业务领域和关键功能]

## 技术栈

- **语言**：[待填写]
- **框架**：[待填写]
- **数据库**：[待填写]

---

## DDD 四层架构

```
┌─────────────────────────────────────────────────────┐
│              用户界面层 (Interfaces)                 │
│   Controller · DTO · Assembler · Facade            │
├─────────────────────────────────────────────────────┤
│                应用层 (Application)                  │
│   Service · Command · Query · EventHandler         │
├─────────────────────────────────────────────────────┤
│                 领域层 (Domain)                      │
│   Entity · ValueObject · Aggregate · Repository接口 │
│   DomainService · DomainEvent                      │
├─────────────────────────────────────────────────────┤
│             基础设施层 (Infrastructure)              │
│   Repository实现 · MQ · Cache · ExternalService     │
└─────────────────────────────────────────────────────┘
```

**依赖原则**：上层依赖下层，领域层不依赖技术框架。

详细设计指南见 `.claude/rules/local/ddd-design.md`。

## 限界上下文

[待填写：定义核心限界上下文及其边界]

---

## 构建命令

```bash
# 安装依赖
[待填写]

# 开发模式
[待填写]

# 测试
[待填写]

# 代码检查
[待填写]
```

---

## 编码规范

- **命名约定**：见 `.claude/rules/local/naming.md`
- **框架模板**：见 `.claude/rules/local/framework.md`
- **API 模板**：见 `.claude/rules/local/api.md`

---

## 测试策略

| 层级 | 测试类型 | 重点 |
|-----|---------|------|
| 领域层 | 单元测试 | 业务规则、不变性约束 |
| 应用层 | 单元测试 | 用例编排、命令处理 |
| 基础设施层 | 集成测试 | 仓储实现、外部服务 |
| 接口层 | E2E 测试 | API 契约、请求响应 |

---

## 规则治理

### 规则文件约定
- **规则目录**：`.claude/rules/local/` 存放项目特定规则
- **命名规范**：`{类别}.md`
- **规则粒度**：每个规则文件专注一个主题

### 渐进式规则策略
- 初始包含 DDD 设计指南和基础模板
- 需要新规则时：创建规则文件 → 在 CLAUDE.md 中引用
- 规则文件应按需扩展，避免冗余

## 注意事项

- **DDD 改造计划**：见 `.claude/rules/local/ddd-refactoring-plan.md`
- **源码分析**：这是 Claude Code 源码快照，用于教育研究和架构分析
- **架构特点**：巨型文件（QueryEngine.ts ~46K 行）、职责混合、循环依赖
