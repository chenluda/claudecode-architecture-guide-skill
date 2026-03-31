# Claude Code Architecture Design Guide

> An AI Agent Skill distilled from **Claude Code** (~1,900 files, 510K+ lines of TypeScript) — Anthropic's agentic CLI for software engineering. Use it to build new systems from scratch or retrofit architecture patterns into existing ones.

[English](#overview) | [中文](#概览)

---

## Overview

This is a **Agent Skill** that packages the architectural essence of Claude Code into a reusable, structured guide. Instead of copying features, it transfers the underlying **engineering principles, design patterns, and evolution strategies** that made a 510K-line codebase maintainable.

### What You Get

| Document | What's Inside |
|----------|---------------|
| [SKILL.md](SKILL.md) | Entry point — 12 decision principles, workflow, output template |
| [architecture-principles.md](references/architecture-principles.md) | 12 technical architecture principles with source code analysis |
| [pattern-catalog.md](references/pattern-catalog.md) | 18 transferable design patterns with code examples |
| [review-checklists.md](references/review-checklists.md) | 6 architecture review checklists (87 items total) |
| [from-zero-blueprint.md](references/from-zero-blueprint.md) | Build from scratch — 5-phase path with code skeletons |
| [migration-playbook.md](references/migration-playbook.md) | Retrofit existing systems — 3 paths by project stage |

### Two Paths

- **Building from scratch?** → Follow the [From-Zero Blueprint](references/from-zero-blueprint.md) (Phase 1–5)
- **Improving an existing system?** → Pick a path in the [Migration Playbook](references/migration-playbook.md)

---

## 12 Architecture Principles

Technical architecture principles extracted from Claude Code (detailed in [architecture-principles.md](references/architecture-principles.md)):

| # | Principle | Core Idea |
|---|-----------|-----------|
| 1 | Separate orchestration, execution, and policy | `query.ts` (orchestration) / `toolOrchestration.ts` (execution) / `permissions.ts` (policy) — no cross-imports |
| 2 | Agent loop as explicit state machine | Typed `State` for mutable data, immutable `Config` snapshot at entry, typed `Continue` / `Terminal` transitions |
| 3 | Read-write partitioned concurrency | Read-only tools run concurrently; write tools run serially; `isConcurrencySafe` declared per tool |
| 4 | Narrow dependency injection with factory | 4-field `QueryDeps` type + `productionDeps()` factory; `typeof fn` for auto type-sync |
| 5 | Minimal state management | 34-line Store; 4-tier state hierarchy (process → app → query → tool) |
| 6 | Feature-flag driven incremental rollout | Build-time `feature()` for dead-code elimination + runtime statsig for A/B testing |
| 7 | Parallel prefetch & lazy loading | Fire I/O before heavy imports; `import()` non-critical modules on demand; `memoize(init)` |
| 8 | AsyncGenerator-centric streaming | Backpressure-friendly `yield*` composition throughout the entire pipeline |
| 9 | Multi-layer permission model | Layered deny→ask→check→sandbox→classifier→mode decision chain with typed results |
| 10 | Graceful degradation & recovery chains | max_output_tokens recovery, model fallback, streaming fallback, reactive compact |
| 11 | Full-lifecycle hook system | 15+ hook events from SessionStart to SessionEnd; JSON-in/JSON-out, language-agnostic |
| 12 | Progressive-trust plugin architecture | builtin > marketplace > git > session — tiered capabilities with SSRF protection |

SKILL.md also provides a complementary set of **decision rules** (prefer X over Y) for quick architectural judgment calls.

---

## 18 Design Patterns

Patterns from the [catalog](references/pattern-catalog.md), each with problem/solution/code:

| # | Pattern | Core Idea |
|---|---------|-----------|
| 1 | State-Machine Loop | Model the main loop as an explicit state machine with typed transitions |
| 2 | Read-Write Partitioned Concurrency | Read-only ops run concurrently; writes run serially |
| 3 | Streaming Executor | Start executing tools while the model is still generating |
| 4 | Narrow Dependency Injection | 4-field `Deps` type + `productionDeps()` factory |
| 5 | Context Bag | A single typed context object threaded through deep call stacks |
| 6 | Cascading Abort Controller | Parent-child abort tree for graceful concurrent cancellation |
| 7 | Withholding (Deferred Error Exposure) | Hold back recoverable errors; retry transparently before surfacing |
| 8 | 34-Line Minimal Store | `getState` / `setState` / `subscribe` — no framework needed |
| 9 | Permission Rule Chain | Layered deny→ask→check→sandbox→classifier→mode decision chain |
| 10 | Dual-Track Feature Flags | Build-time tree-shaking + runtime statsig/growthbook |
| 11 | Parallel Prefetch & Lazy Load | Fire I/O before heavy imports; `import()` non-critical modules on demand |
| 12 | Shell Hook Lifecycle Protocol | Language-agnostic JSON-in/JSON-out hook execution |
| 13 | Synthetic Error Message | Generate paired error results for cancelled/timed-out operations |
| 14 | Context Budget Management | Layered truncation → micro-compact → auto-compact → collapse |
| 15 | Memoized Single-Init Guarantee | `memoize(async () => { ... })` for idempotent initialization |
| 16 | Dual-Channel Message Model | Separate API/transcript chain from metadata/progress — save tokens, keep UI responsive |
| 17 | Layered Prompt with Cache Scope | Split system prompt into global/org/dynamic blocks for multi-level prompt caching |
| 18 | VCR Recording & Replay | Record real streaming API calls as fixtures; dehydrate/hydrate for cross-env stability |

---

## Quick Start

### For Cursor Users

Copy this skill into your project or personal skills directory:

```bash
# Project-level (shared with team)
cp -r claudecode-architecture-design-guide /path/to/your-project/.cursor/skills/

# Personal (available across all projects)
cp -r claudecode-architecture-design-guide ~/.cursor/skills/
```

The skill will automatically appear in Cursor's available skills. The agent reads it when your query matches the description triggers (architecture design, technical strategy, permission models, etc.).

### For Claude Code Users

```bash
cp -r claudecode-architecture-design-guide ~/.claude/skills/
```

### Manual Use

Just read [SKILL.md](SKILL.md) and follow the workflow. The references are standalone Markdown files — no tooling required.

---

## From-Zero Blueprint at a Glance

If you're building a new system, the [blueprint](references/from-zero-blueprint.md) walks you through 5 phases:

```
Phase 1: Minimal Viable Skeleton
         ├── 3-layer directory (core / execution / policy)
         ├── State-machine main loop
         ├── Unified tool interface
         ├── Minimal store (34 lines)
         ├── Permission check point (even if allow-all)
         └── Custom error types

Phase 2: Reliability & Observability
         ├── Error recovery chains
         ├── Graceful shutdown
         ├── Performance checkpoints
         ├── Narrow dependency injection
         └── Synthetic error messages

Phase 3: Concurrency & Performance
         ├── Read-write partitioned concurrency
         ├── Cascading abort controllers
         ├── Startup path parallelization
         └── Lazy loading for heavy modules

Phase 4: Extensibility
         ├── Hook lifecycle protocol
         ├── Plugin directory convention
         ├── Tiered trust model
         └── Dual-track feature flags

Phase 5: Production Readiness
         ├── Resource budget management
         ├── Config schema + migrations
         └── Diagnostic tooling
```

Each phase has concrete code skeletons and a **completion checklist**.

---

## Architecture Review Checklists

The [review checklists](references/review-checklists.md) cover 6 areas:

- **A. Core Architecture** — layering, state management, error handling, concurrency
- **B. Security & Permissions** — permission model, trust boundaries, sandboxing
- **C. Performance** — startup, runtime, memory management
- **D. Extensibility** — tool/command system, hooks/plugins, configuration
- **E. Observability** — telemetry, diagnostics
- **F. Agent/LLM Specific** — conversation management, model calls, sub-agent coordination

---

## Source

This skill was distilled from [instructkr/claude-code](https://github.com/instructkr/claude-code) repository, analyzed for educational and architecture research purposes. The principles and patterns are general-purpose and applicable to any complex system — not limited to AI applications.

---

## 概览

这是一个从 **Claude Code**（Anthropic 的 Agent CLI，约 1900 文件、51 万行 TypeScript）中提炼的 **Agent Skill**。它将 Claude Code 的架构精华打包为可复用的结构化指南。

### 你将获得

| 文档 | 内容说明 |
|------|----------|
| [SKILL.md](SKILL.md) | 入口 — 12 条决策原则、工作流、输出模板 |
| [architecture-principles.md](references/architecture-principles.md) | 12 条技术架构原则及源码分析 |
| [pattern-catalog.md](references/pattern-catalog.md) | 18 个可迁移设计模式及代码示例 |
| [review-checklists.md](references/review-checklists.md) | 6 大类架构评审清单（共 87 项） |
| [from-zero-blueprint.md](references/from-zero-blueprint.md) | 从零构建 — 5 阶段路径及代码骨架 |
| [migration-playbook.md](references/migration-playbook.md) | 改造现有系统 — 按项目阶段分 3 条路径 |

### 两条路径

- **从零构建？** → 按 [从零构建蓝图](references/from-zero-blueprint.md)（Phase 1–5）推进
- **改造现有系统？** → 在 [迁移手册](references/migration-playbook.md) 中选择对应路径

---

## 12 条架构原则

从 Claude Code 中提取的技术架构原则（详见 [architecture-principles.md](references/architecture-principles.md)）：

| # | 原则 | 核心思想 |
|---|------|----------|
| 1 | 编排、执行、策略分离 | `query.ts`（编排）/ `toolOrchestration.ts`（执行）/ `permissions.ts`（策略）— 无交叉引用 |
| 2 | Agent 循环作为显式状态机 | 可变数据用类型化 `State`，入口处不可变 `Config` 快照，类型化 `Continue` / `Terminal` 转换 |
| 3 | 读写分区并发 | 只读工具并发执行；写工具串行执行；每个工具声明 `isConcurrencySafe` |
| 4 | 窄依赖注入 + 工厂 | 4 字段 `QueryDeps` 类型 + `productionDeps()` 工厂；`typeof fn` 自动类型同步 |
| 5 | 最小化状态管理 | 34 行 Store；4 层状态层级（process → app → query → tool） |
| 6 | 功能标志驱动渐进式发布 | 构建时 `feature()` 实现死代码消除 + 运行时 statsig 实现 A/B 测试 |
| 7 | 并行预取与懒加载 | 在重导入前触发 I/O；按需 `import()` 非关键模块；`memoize(init)` |
| 8 | 以 AsyncGenerator 为中心的流式处理 | 全管道支持背压友好的 `yield*` 组合 |
| 9 | 多层权限模型 | 分层 deny→ask→check→sandbox→classifier→mode 决策链，结果类型化 |
| 10 | 优雅降级与恢复链 | max_output_tokens 恢复、模型降级、流式降级、响应式压缩 |
| 11 | 全生命周期钩子系统 | 从 SessionStart 到 SessionEnd 的 15+ 钩子事件；JSON 输入/输出，语言无关 |
| 12 | 渐进式信任插件架构 | builtin > marketplace > git > session — 分层能力 + SSRF 防护 |

SKILL.md 还提供一套互补的**决策规则**（优先 X 而非 Y），用于快速架构判断。

---

## 18 个设计模式

来自[模式目录](references/pattern-catalog.md)，每个都包含问题/解决方案/代码：

| # | 模式 | 核心思想 |
|---|------|----------|
| 1 | 状态机循环 | 将主循环建模为具有类型化转换的显式状态机 |
| 2 | 读写分区并发 | 只读操作并发执行；写操作串行执行 |
| 3 | 流式执行器 | 模型仍在生成时就开始执行工具 |
| 4 | 窄依赖注入 | 4 字段 `Deps` 类型 + `productionDeps()` 工厂 |
| 5 | 上下文包 | 单个类型化上下文对象贯穿深层调用栈 |
| 6 | 级联中止控制器 | 父子中止树实现优雅并发取消 |
| 7 | 保留（延迟错误暴露）| 暂存可恢复错误；透明重试后再暴露 |
| 8 | 34 行最小化 Store | `getState` / `setState` / `subscribe` — 无需框架 |
| 9 | 权限规则链 | 分层 deny→ask→check→sandbox→classifier→mode 决策链 |
| 10 | 双轨功能标志 | 构建时 tree-shaking + 运行时 statsig/growthbook |
| 11 | 并行预取与懒加载 | 在重导入前触发 I/O；按需 `import()` 非关键模块 |
| 12 | Shell 钩子生命周期协议 | 语言无关的 JSON 输入/输出钩子执行 |
| 13 | 合成错误消息 | 为已取消/超时操作生成成对错误结果 |
| 14 | 上下文预算管理 | 分层截断 → 微压缩 → 自动压缩 → 折叠 |
| 15 | 记忆化单次初始化保证 | `memoize(async () => { ... })` 实现幂等初始化 |
| 16 | 双通道消息模型 | 分离 API/transcript 链与元数据/进度 — 省 token，保持 UI 响应 |
| 17 | 分层 Prompt 缓存作用域 | 将 system prompt 拆为 global/org/dynamic 块，实现多级 prompt 缓存 |
| 18 | VCR 录制/回放测试 | 录制真实流式 API 调用为 fixture；dehydrate/hydrate 保证跨环境稳定 |

---

## 快速开始

### Cursor 用户

将此 skill 复制到你的项目或个人 skills 目录：

```bash
# 项目级（与团队共享）
cp -r claudecode-architecture-design-guide /path/to/your-project/.cursor/skills/

# 个人级（跨项目可用）
cp -r claudecode-architecture-design-guide ~/.cursor/skills/
```

该 skill 将自动出现在 Cursor 的可用 skills 中。当你的查询匹配描述触发词（架构设计、技术策略、权限模型等）时，agent 会读取它。

### Claude Code 用户

```bash
cp -r claudecode-architecture-design-guide ~/.claude/skills/
```

### 手动使用

直接阅读 [SKILL.md](SKILL.md) 并按工作流执行。references 是独立的 Markdown 文件 — 无需任何工具。

---

## 从零构建蓝图概览

如果你正在构建新系统，[蓝图](references/from-zero-blueprint.md) 将引导你完成 5 个阶段：

```
Phase 1: 最小可行骨架
         ├── 3 层目录（core / execution / policy）
         ├── 状态机主循环
         ├── 统一工具接口
         ├── 最小化 store（34 行）
         ├── 权限检查点（即使是全允许）
         └── 自定义错误类型

Phase 2: 可靠性与可观测性
         ├── 错误恢复链
         ├── 优雅关闭
         ├── 性能检查点
         ├── 窄依赖注入
         └── 合成错误消息

Phase 3: 并发与性能
         ├── 读写分区并发
         ├── 级联中止控制器
         ├── 启动路径并行化
         └── 重模块懒加载

Phase 4: 可扩展性
         ├── 钩子生命周期协议
         ├── 插件目录约定
         ├── 分层信任模型
         └── 双轨功能标志

Phase 5: 生产就绪
         ├── 资源预算管理
         ├── 配置 schema + 迁移
         └── 诊断工具
```

每个阶段都有具体的代码骨架和**完成检查清单**。

---

## 架构评审清单

[评审清单](references/review-checklists.md) 涵盖 6 大领域：

- **A. 核心架构** — 分层、状态管理、错误处理、并发
- **B. 安全与权限** — 权限模型、信任边界、沙箱
- **C. 性能** — 启动、运行时、内存管理
- **D. 可扩展性** — 工具/命令系统、钩子/插件、配置
- **E. 可观测性** — 遥测、诊断
- **F. Agent/LLM 专用** — 对话管理、模型调用、子 agent 协调

---

## 来源

本 skill 从 [instructkr/claude-code](https://github.com/instructkr/claude-code) 仓库中提炼，用于教育和架构研究目的。这些原则和模式是通用的，适用于任何复杂系统 — 不限于 AI 应用。

---

### 安装

```bash
# Cursor 用户 — 项目级
cp -r claudecode-architecture-design-guide /path/to/your-project/.cursor/skills/

# Cursor 用户 — 个人级
cp -r claudecode-architecture-design-guide ~/.cursor/skills/

# Claude Code 用户
cp -r claudecode-architecture-design-guide ~/.claude/skills/
```

---

## License

MIT