# 架构原则详解

从 Claude Code 50 万行 TypeScript 工程中提炼的 12 条核心架构原则。每条原则包含来源分析、实现模式与反模式。

---

## 原则 1：分离编排、执行与策略层

**来源**：`query.ts`（编排）、`toolOrchestration.ts`（执行）、`permissions.ts`（策略）

**核心洞察**：Claude Code 将 Agent 循环拆为三个清晰的层：

| 层 | 职责 | 核心文件 |
|---|---|---|
| 编排层 | 驱动 while(true) 循环、管理状态转移、决定何时继续/终止 | `query.ts` queryLoop |
| 执行层 | 运行工具、管理并发、收集结果 | `toolOrchestration.ts`, `toolExecution.ts` |
| 策略层 | 权限判定、规则匹配、安全分类 | `permissions.ts`, `useCanUseTool.tsx` |

**实现模式**：
- 编排层通过 `canUseTool` 回调与策略层解耦，而非直接调用权限检查
- 执行层通过 `findToolByName` + 统一 `Tool` 接口与具体工具解耦
- 策略层通过 `PermissionRule` 配置驱动，非硬编码

**反模式**：
- 在工具执行函数里内联权限检查逻辑
- 在编排循环里直接处理 API 重试细节
- 将业务规则硬编码在策略层

---

## 原则 2：Agent 循环即状态机

**来源**：`query.ts` 的 `queryLoop` 函数

**核心洞察**：主循环不是简单的 request-response，而是一个显式状态机：

```
┌──────────────────────────────────────────┐
│              queryLoop                    │
│                                          │
│  State {                                 │
│    messages, toolUseContext,              │
│    autoCompactTracking,                  │
│    maxOutputTokensRecoveryCount,         │
│    turnCount, transition                 │
│  }                                       │
│                                          │
│  while(true) {                           │
│    1. 解构 state                          │
│    2. 构建 system prompt                  │
│    3. 调用 deps.callModel()              │
│    4. 处理流式响应                         │
│    5. 执行工具 (if any)                   │
│    6. 决定: Continue(reason) | Terminal   │
│  }                                       │
└──────────────────────────────────────────┘
```

**关键设计**：
- `State` 类型集中所有跨迭代可变状态 — 不散落在局部变量中
- `QueryConfig` 在入口处快照一次，循环内不可变 — 避免配置漂移
- `Continue` 与 `Terminal` 是类型化的转移结果 — 测试可断言恢复路径
- `transition` 字段记录"上一次为什么继续" — 调试友好

**何时应用**：任何需要多轮交互、重试、恢复的核心循环

---

## 原则 3：工具并发的读写分区

**来源**：`toolOrchestration.ts` 的 `partitionToolCalls`、`StreamingToolExecutor.ts`

**核心洞察**：不是所有工具调用都需要串行。Claude Code 将工具按副作用分区：

- **只读工具**（`isConcurrencySafe=true`）：文件读取、搜索、Web 获取 → 可并发
- **写入工具**（`isConcurrencySafe=false`）：文件写入、Bash 命令 → 必须串行

**分区算法**：
1. 遍历工具调用列表
2. 连续的只读工具合并为一个并发批次
3. 遇到写入工具则切断，形成新的串行批次
4. 结果：`[{concurrent: [read, read]}, {serial: [write]}, {concurrent: [read]}]`

**流式执行进化**：`StreamingToolExecutor` 更进一步 — 在模型还在生成时就开始执行已完成解析的工具，通过状态机（queued → executing → completed → yielded）管理每个工具生命周期。

**关键细节**：
- Bash 错误触发 `siblingAbortController`，取消兄弟子进程
- 非 Bash 错误不取消兄弟（读操作互相独立）
- 进度消息立即 yield；结果消息按顺序缓冲

---

## 原则 4：依赖注入要窄、要有工厂函数

**来源**：`query/deps.ts` 的 `QueryDeps`

**核心洞察**：
```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}

export function productionDeps(): QueryDeps {
  return { callModel: queryModelWithStreaming, ... }
}
```

**设计要点**：
- 范围刻意窄小（仅 4 个依赖）— "先证明模式再扩展"
- 使用 `typeof fn` 自动同步类型签名
- `productionDeps()` 工厂函数 — 测试注入 fakes，生产走真实实现
- 不用 DI 容器 — 函数式组合足够

---

## 原则 5：最小化状态管理

**来源**：`state/store.ts`（34 行）、`bootstrap/state.ts`

**核心洞察**：在 1900+ 文件的大型项目中，状态管理仅用 34 行代码：

```typescript
function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater) => { /* Object.is bailout + notify */ },
    subscribe: (listener) => { listeners.add(listener); return () => listeners.delete(listener) }
  }
}
```

**层次化状态**：
- **进程级**（`bootstrap/state.ts`）：会话 ID、费用、模型 — getter/setter + 显式注释"不要滥用全局状态"
- **应用级**（`AppStateStore`）：设置、权限上下文、MCP、插件
- **查询级**（`State` in queryLoop）：消息、工具上下文、恢复计数
- **工具级**（`ToolUseContext`）：选项、abort controller、进行中的工具 ID

**原则**：每层状态的可变范围 ≤ 其生命周期。不跨层持有可变引用。

---

## 原则 6：Feature Flag 驱动的渐进式上线

**来源**：`bun:bundle` 的 `feature()` 宏、`query/config.ts`

**核心洞察**：两种 Feature Flag 机制服务不同目的：

| 机制 | 时机 | 目的 | 示例 |
|---|---|---|---|
| `feature('X')` (bun:bundle) | **构建时** | 死代码消除 | `VOICE_MODE`, `PROACTIVE`, `REACTIVE_COMPACT` |
| statsig/growthbook | **运行时** | A/B 测试、灰度 | `streamingToolExecution`, `fastModeEnabled` |

**构建时 Flag 的妙用**：
```typescript
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? require('./services/compact/reactiveCompact.js')
  : null
```
- 未启用时整个模块从 bundle 中消除
- 不增加运行时开销
- 条件 require 保持类型安全（`as typeof import(...)`)

**运行时 Flag 的快照**：`buildQueryConfig()` 在 query 入口处快照一次，避免循环内 flag 值变化导致行为不一致。

---

## 原则 7：启动路径并行化与延迟加载

**来源**：`main.tsx` 文件顶部、`entrypoints/init.ts`

**核心洞察**：Claude Code 的启动优化体现了"时间即用户体验"的工程思维：

**并行预取**（主模块加载前的副作用）：
```typescript
// main.tsx — 在所有其他 import 之前触发
startMdmRawRead()        // MDM 配置预读
startKeychainPrefetch()  // macOS Keychain 预取
```

**延迟加载**（按需引入重模块）：
- OpenTelemetry（~400KB）— 仅在遥测初始化时 `import()`
- gRPC exporter（~700KB）— 在 instrumentation 内部懒加载
- React + Ink — 仅交互模式下 `import('./components/App.js')`
- Analytics/GrowthBook — `Promise.all()` 异步初始化

**init() 的 memoize 保证**：整个初始化函数用 `memoize()` 包裹，确保只执行一次。

---

## 原则 8：流式架构以 AsyncGenerator 为核心

**来源**：`query.ts`、`toolOrchestration.ts`、`StreamingToolExecutor.ts`

**核心洞察**：整个数据流基于 AsyncGenerator，实现了背压友好的流式处理：

```
query() → yield* queryLoop()
  → yield { type: 'stream_request_start' }
  → yield* deps.callModel()          // 模型流式响应
  → yield* runTools()                 // 工具执行结果
  → yield* streamingToolExecutor      // 流式工具执行
```

**优势**：
- 天然支持背压（消费者控制拉取速度）
- `yield*` 委托使流组合变得简洁
- 可以在任意点 `return` 终止整个链
- `using` 声明确保资源清理（如 memory prefetch）

**Withholding 模式**：对于可恢复的错误（如 `max_output_tokens`），先 withhold 不 yield 给上游，尝试内部恢复。恢复成功则错误对消费者透明。

---

## 原则 9：多层权限模型

**来源**：`permissions.ts`、`useCanUseTool.tsx`、`ToolPermissionContext`

**核心洞察**：权限不是简单的 allow/deny，而是分层决策链：

```
1. Deny Rules    → 直接拒绝（管理员策略、文件保护）
2. Ask Rules     → 需要用户确认（自定义规则）
3. Tool.checkPermissions() → 工具自身的权限检查
4. Sandbox check → Bash 命令的沙箱安全检查
5. Classifier    → AI 分类器的安全评估
6. Mode-specific → bypass/plan/auto 模式特化逻辑
```

**三种交互模式**：
- **Interactive**：弹窗询问用户
- **Coordinator**：通过协调器代理决策
- **Headless**：`shouldAvoidPermissionPrompts` 直接拒绝

**关键设计**：`CanUseToolFn` 是一个统一接口，无论底层走哪条路径，上层代码只关心 `allow | deny | ask` 结果。

---

## 原则 10：优雅降级与恢复链

**来源**：`query.ts` 的多处恢复逻辑

**核心洞察**：系统为每种失败场景设计了恢复链，而非简单报错：

| 失败场景 | 恢复策略 |
|----------|----------|
| `max_output_tokens` | 恢复计数器 +1，重新进入循环（上限 3 次） |
| `prompt_too_long` | 触发 reactive compact 或 context collapse |
| 模型不可用 | `FallbackTriggeredError` 切换到 fallbackModel |
| 流式中断 | Tombstone 旧消息 + `streamingToolExecutor.discard()` + 重建 |
| 工具执行错误 | 转为 tool_result error message，继续对话 |
| Bash 兄弟错误 | siblingAbortController 级联取消同批工具 |

**Withholding 模式的精妙**：`isWithheldMaxOutputTokens` 函数让恢复循环对 SDK 调用者透明 — 外部不知道内部发生了错误和恢复。

---

## 原则 11：Hook 系统的生命周期覆盖

**来源**：`utils/hooks.ts`、`services/tools/toolHooks.ts`

**核心洞察**：Claude Code 的 Hook 系统覆盖了完整的生命周期：

```
SessionStart → Setup → InstructionsLoaded
  → UserPromptSubmit
    → PreToolUse → [Tool Execution] → PostToolUse / PostToolUseFailure
    → PermissionRequest / PermissionDenied
  → Stop / StopFailure
  → PreCompact → [Compact] → PostCompact
  → SubagentStart → SubagentStop
  → TaskCreated → TaskCompleted
SessionEnd
```

**Hook 执行模型**：
- 用户通过 settings 配置 shell 命令
- 支持同步（阻塞）和异步（后台）两种模式
- 输入通过 JSON stdin 传入，输出通过 JSON stdout 返回
- 支持 `blocking`（阻止继续）、`additionalContext`（注入上下文）等控制能力
- 语言无关 — 任何能读写 JSON 的程序都可以做 Hook

---

## 原则 12：插件架构的渐进式信任

**来源**：`utils/plugins/pluginLoader.ts`

**核心洞察**：插件系统不是全开放的，而是分级信任：

| 来源 | 信任级别 | 约束 |
|------|---------|------|
| 内置插件 | 完全信任 | 与核心代码同等权限 |
| Marketplace 官方插件 | 受控信任 | 经过审核、版本锁定 |
| Git 仓库插件 | 有限信任 | 需用户确认、策略检查 |
| Session-only 插件 | 临时信任 | 仅当前会话有效 |

**目录约定**：
```
my-plugin/
├── plugin.json      # 清单：元数据、依赖声明
├── commands/        # 自定义斜杠命令
├── agents/          # 自定义 AI 代理
└── hooks/
    └── hooks.json   # Hook 定义
```

**安全机制**：
- Marketplace 策略白名单/黑名单
- SSRF 防护（`ssrfGuard.ts`）
- 配置变量替换（不暴露原始密钥）
- 启用/禁用状态管理
