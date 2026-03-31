# 设计模式目录

15 个从 Claude Code 中提炼的可迁移设计模式。每个模式包含问题场景、解决方案、实现要点与迁移建议。

---

## 模式 1：状态机驱动的主循环（State-Machine Loop）

**问题**：复杂的多轮交互逻辑散落在回调或嵌套 if-else 中，难以理解和测试。

**方案**：将主循环建模为显式状态机，集中管理可变状态。

```typescript
type State = {
  messages: Message[]
  recoveryCount: number
  turnCount: number
  transition: Continue | undefined  // 上一次转移原因
}

async function* mainLoop(params: Params): AsyncGenerator<Event, Terminal> {
  let state: State = { /* 初始状态 */ }
  const config = snapshotConfig()  // 不可变配置
  while (true) {
    const { messages, recoveryCount } = state
    // ... 处理逻辑 ...
    // 决定转移
    if (shouldContinue) {
      state = { ...state, transition: { reason: 'tool_use' }, turnCount: state.turnCount + 1 }
      continue
    }
    return { reason: 'completed' }
  }
}
```

**要点**：
- `State` 类型是唯一的可变状态容器
- `Config` 在入口快照，循环内只读
- 每次 `continue` 构造新的 state 对象（意图明确）
- `transition` 字段让测试可以断言恢复路径

**适用场景**：Agent 循环、工作流引擎、长连接管理、任务调度器

---

## 模式 2：读写分区并发（Read-Write Partitioned Concurrency）

**问题**：并发执行所有操作不安全，串行执行太慢。

**方案**：按副作用将操作分为只读和写入两类，只读批量并发，写入串行执行。

```typescript
function partitionCalls(calls: Call[], context: Context): Batch[] {
  return calls.reduce((batches, call) => {
    const isSafe = call.tool.isConcurrencySafe(call.input)
    if (isSafe && batches.at(-1)?.isConcurrencySafe) {
      batches.at(-1)!.items.push(call)
    } else {
      batches.push({ isConcurrencySafe: isSafe, items: [call] })
    }
    return batches
  }, [])
}
```

**要点**：
- 连续只读操作合并为一个并发批次
- 写入操作切断批次，形成串行点
- 并发度可配置（`MAX_CONCURRENCY`）
- 安全性声明在工具定义侧（`isConcurrencySafe`），非调度侧

**适用场景**：数据库读写分离、文件系统操作、API 批量调用

---

## 模式 3：流式执行器（Streaming Executor）

**问题**：传统模式等待所有工具调用解析完毕才开始执行，浪费等待时间。

**方案**：边解析边执行 — 当一个工具调用的参数流式完成时立即启动执行。

```typescript
class StreamingExecutor {
  private tools: TrackedTool[] = []           // queued → executing → completed → yielded
  private siblingAbortController: AbortController

  addTool(block: ToolBlock): void {
    this.tools.push({ ...block, status: 'queued' })
    this.processQueue()
  }

  private canExecute(isSafe: boolean): boolean {
    const executing = this.tools.filter(t => t.status === 'executing')
    return executing.length === 0 || (isSafe && executing.every(t => t.isConcurrencySafe))
  }

  async *getRemainingResults(): AsyncGenerator<Result> {
    while (this.hasUnfinished()) {
      await this.processQueue()
      yield* this.getCompleted()
      if (this.needsWait()) await Promise.race([...executingPromises, progressPromise])
    }
  }

  discard(): void { this.discarded = true }  // 流式降级时抛弃未完成工具
}
```

**要点**：
- 4 态生命周期：queued → executing → completed → yielded
- 进度消息立即 yield，结果消息按序缓冲
- `discard()` 支持流式降级场景
- `siblingAbortController` 实现 Bash 错误级联取消

**适用场景**：流式管道、实时数据处理、并行任务调度

---

## 模式 4：窄依赖注入（Narrow Dependency Injection）

**问题**：全局 mock 和 `spyOn` 在大型测试套件中脆弱且耦合。

**方案**：将外部依赖收敛为一个窄类型，配以生产工厂函数。

```typescript
type Deps = {
  callModel: typeof realCallModel
  compact: typeof realCompact
  uuid: () => string
}

function productionDeps(): Deps {
  return { callModel: realCallModel, compact: realCompact, uuid: randomUUID }
}

// 使用时
async function* process(params: Params & { deps?: Deps }) {
  const deps = params.deps ?? productionDeps()
  const result = await deps.callModel(...)
}
```

**要点**：
- `typeof fn` 保持签名自动同步
- 范围刻意窄小，先证明模式再扩展
- 工厂函数不是单例 — 每次调用创建新实例
- 测试只需传入部分 fakes，其余走默认

**适用场景**：核心业务逻辑的测试友好化

---

## 模式 5：上下文穿透袋（Context Bag）

**问题**：深层函数调用需要大量参数传递，但 DI 容器过重。

**方案**：定义一个 Context 类型，承载当前操作所需的所有运行时上下文。

```typescript
type ToolUseContext = {
  options: { tools: Tool[], commands: Command[], mcpClients: Client[] }
  abortController: AbortController
  messages: Message[]
  getAppState: () => AppState
  setAppState: (updater) => void
  queryTracking: { chainId: string, depth: number }
}
```

**要点**：
- Context 是值对象，可通过 spread 创建变体
- 包含 getAppState/setAppState — 但不直接持有状态引用
- 包含 abortController — 取消传播
- 在工具执行流中作为参数向下传递

**适用场景**：中间件链、请求处理管道、多层级函数调用

---

## 模式 6：级联取消控制器（Cascading Abort Controller）

**问题**：并发操作中一个失败后，如何优雅取消兄弟操作？

**方案**：构建 abort controller 父子树，子取消不影响父，父取消级联到所有子。

```
queryLoop.abortController (用户中断 → 全部取消)
  └── siblingAbortController (Bash 错误 → 兄弟工具取消)
       ├── tool1.abortController
       ├── tool2.abortController
       └── tool3.abortController
```

**要点**：
- `createChildAbortController(parent)` 创建子控制器
- 子 abort 可选择是否冒泡到父（权限拒绝需冒泡，兄弟错误不冒泡）
- 取消原因编码在 `signal.reason` 中（`'sibling_error'`、`'user_interrupted'`、`'streaming_fallback'`）
- 被取消的工具收到合成错误消息，不留未配对的 tool_use

**适用场景**：并发 HTTP 请求、子进程管理、微服务编排

---

## 模式 7：Withholding 模式（错误延迟暴露）

**问题**：某些错误可以内部恢复，但一旦暴露给上层消费者就无法撤回。

**方案**：在 yield 给消费者之前"扣住"可恢复的错误，先尝试内部恢复。

```typescript
function isWithheld(msg: Message): boolean {
  return msg.type === 'assistant' && msg.apiError === 'max_output_tokens'
}

// 在循环中
for await (const msg of callModel(...)) {
  if (isWithheld(msg)) {
    withheldMessage = msg
    continue  // 不 yield，尝试恢复
  }
  yield msg
}

// 恢复成功 → withheldMessage 被丢弃，消费者无感知
// 恢复失败（超过重试上限）→ yield withheldMessage
```

**要点**：
- 对 SDK 调用者透明 — 不知道内部发生了错误和恢复
- 有明确的恢复上限（`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`）
- 与 streaming fallback 配合 — tombstone 旧消息后重建

**适用场景**：API 网关的重试透明化、数据库连接池的故障转移、缓存的透明回源

---

## 模式 8：34 行极简 Store

**问题**：需要响应式状态管理，但不想引入 Redux/MobX 等重框架。

**方案**：34 行函数式 Store — getState、setState、subscribe。

```typescript
function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // bailout
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener) => { listeners.add(listener); return () => listeners.delete(listener) }
  }
}
```

**要点**：
- `Object.is` 比较避免无效通知
- onChange 回调支持中间件/日志
- 返回取消订阅函数，防止内存泄漏
- 无魔法、无代理、无装饰器

**适用场景**：CLI 应用状态、轻量级 UI 状态、配置管理

---

## 模式 9：分层权限规则链（Permission Rule Chain）

**问题**：权限逻辑在不同场景下差异巨大（交互/无头/协调器），难以统一管理。

**方案**：将权限判定建模为有序规则链，链的结果驱动后续分支。

```
输入: (tool, input, permissionContext)
  │
  ├─ DenyRules    → match? → DENY (不可覆盖)
  ├─ AskRules     → match? → ASK  (需确认)
  ├─ tool.check() → deny?  → DENY/ASK
  ├─ Sandbox      → safe?  → ALLOW (沙箱内自动放行)
  ├─ Classifier   → risky? → DENY
  └─ Mode         → bypass? → ALLOW
  │
  结果: allow | deny(reason) | ask(options)
```

**要点**：
- 规则来源多样：管理员策略、用户配置、项目配置
- `PermissionDecision<T>` 是类型化的判定结果
- 拒绝追踪：`DenialTrackingState` 记录拒绝次数，超阈值切换到 prompting 模式
- 分析友好：每个判定路径都有 analytics event

**适用场景**：RBAC、API 权限网关、资源访问控制

---

## 模式 10：构建时与运行时 Feature Flag 双轨

**问题**：灰度发布需要运行时 flag，但不想为未启用功能付出 bundle 开销。

**方案**：两套 flag 系统并行 — 构建时 tree-shaking + 运行时 statsig。

```typescript
// 构建时（bun:bundle）— 未启用的代码完全消除
const voiceModule = feature('VOICE_MODE')
  ? require('./voice/index.js') as typeof import('./voice/index.js')
  : null

// 运行时（statsig/growthbook）— A/B 测试和灰度
const config = {
  gates: {
    streamingToolExecution: checkStatsigGate('streaming_tool_execution'),
  }
}
```

**要点**：
- 构建时 flag 用于大功能模块的按需打包
- 运行时 flag 在 query 入口快照一次，循环内不变
- 条件 require + `as typeof import(...)` 保持类型安全
- 两套系统互补，不冲突

**适用场景**：产品灰度发布、平台化 SDK 的模块裁剪

---

## 模式 11：并行预取与延迟加载（Parallel Prefetch & Lazy Load）

**问题**：启动慢 — 串行初始化 + 加载不需要的重模块。

**方案**：关键路径预取 + 非关键路径延迟加载。

```typescript
// 文件顶部副作用 — 在 import 重模块之前触发
startMdmRawRead()         // ~50ms IO
startKeychainPrefetch()   // ~100ms IO

// 后续 import 期间上述 IO 已在进行
import { Commander } from 'commander'
import { React } from 'react'

// 重模块延迟加载
async function initTelemetry() {
  const otel = await import('@opentelemetry/sdk-node')  // ~400KB
  // ...
}

// init() 用 memoize 保证单次
const init = memoize(async () => { /* ... */ })
```

**要点**：
- MDM + Keychain 在模块加载前并行预取
- OpenTelemetry (~400KB) + gRPC (~700KB) 按需 import()
- React/Ink 仅交互模式加载
- `memoize()` 确保 init 不重复

**适用场景**：CLI 启动优化、Web 应用首屏加载

---

## 模式 12：Shell Hook 生命周期协议

**问题**：需要在 Agent 生命周期各点注入自定义逻辑，但不能要求用户写 TypeScript。

**方案**：定义 JSON-in/JSON-out 的 shell 命令协议。

```
Hook 注册（settings）:
  hooks:
    PreToolUse:
      - matcher: { tool_name: "Bash", command_pattern: "rm *" }
        command: "python /path/to/safety_check.py"

执行协议:
  stdin  ← JSON { tool_name, input, session_id, ... }
  stdout → JSON { decision: "block" | "allow", message?: "..." }
  exit_code: 0=success, non-zero=error
```

**要点**：
- 语言无关 — Python、Bash、Go 均可
- 同步模式阻塞执行、异步模式后台运行
- Hook 可注入 `additionalContext` 到对话
- 支持 `blocking`（阻止继续）决策
- SSRF 防护限制 Hook 网络访问

**适用场景**：CI/CD 管道钩子、Git hooks、审批工作流

---

## 模式 13：合成错误消息（Synthetic Error Message）

**问题**：工具调用必须配对 tool_result，但工具可能被取消/超时/降级。

**方案**：为每种异常场景生成合成的 tool_result 错误消息。

```typescript
function createSyntheticError(
  toolUseId: string,
  reason: 'sibling_error' | 'user_interrupted' | 'streaming_fallback',
): Message {
  const msg = reason === 'user_interrupted' ? REJECT_MESSAGE : `Cancelled: ${reason}`
  return createUserMessage({
    content: [{ type: 'tool_result', content: msg, is_error: true, tool_use_id: toolUseId }],
  })
}
```

**要点**：
- 保证 tool_use / tool_result 总是配对 — 不破坏 API 协议
- 不同原因有不同措辞 — UI 显示更友好
- 已产出错误的工具不会重复收到兄弟合成错误
- 与 tombstone 消息配合处理流式降级

**适用场景**：RPC 超时处理、消息队列的死信生成、工作流的补偿事务

---

## 模式 14：上下文预算管理（Context Budget Management）

**问题**：LLM 上下文窗口有限，长对话会溢出。

**方案**：多层次的上下文预算管理策略。

| 策略 | 触发条件 | 效果 |
|------|---------|------|
| Tool Result Budget | 单次结果过大 | 截断/替换为摘要 |
| Micro Compact | 上下文接近阈值 | 压缩旧消息 |
| Auto Compact | 累积 token 超限 | 自动总结历史 |
| Reactive Compact | prompt_too_long 错误 | 响应式压缩 |
| Context Collapse | 极端情况 | 激进折叠 |
| Snip | 历史过长 | 裁剪最老部分 |

**要点**：
- 分层触发，从轻到重
- `AutoCompactTrackingState` 跟踪压缩状态
- 压缩后维护 `taskBudgetRemaining` 告知服务端
- Memory prefetch 使用 `using` 声明自动清理

**适用场景**：LLM 应用的上下文管理、缓存逐出策略、日志轮转

---

## 模式 15：Memoize 单次初始化保证

**问题**：初始化函数被多处调用，需保证只执行一次且结果复用。

**方案**：用 `memoize()` 包裹初始化函数。

```typescript
import memoize from 'lodash-es/memoize.js'

export const init = memoize(async (): Promise<void> => {
  enableConfigs()
  applySafeConfigEnvironmentVariables()
  setupGracefulShutdown()
  await setupTelemetry()
  // ...
})

// 多处调用 init() 安全 — 只执行一次
await init()  // 执行
await init()  // 返回缓存的 Promise
```

**要点**：
- 异步函数 memoize 后返回同一个 Promise — 并发调用安全
- 副作用集中在一个函数中，便于追踪
- 与 `profileCheckpoint()` 配合测量各阶段耗时
- 配合优雅关停：`setupGracefulShutdown()` 确保 flush

**适用场景**：应用初始化、数据库连接池、配置加载

---

## 高级模式

以下 3 个模式在更专业的场景下具有高迁移价值。

---

## 模式 16：双通道消息模型（Dual-Channel Message Model）

**问题**：对话系统的消息既要送入模型 API（transcript 链），又要携带 UI 进度、会话元数据等辅助信息。如果混在一起，进度消息会污染模型上下文、增加 token 开销，元数据会干扰 parent 链的完整性。

**方案**：将消息分为两条通道——**API/transcript 通道**只含模型需要的消息，**元数据/进度通道**承载其余一切。

```typescript
// 通道 1：进入模型链的消息（有 parentUuid 链关系）
type ChainMessage = UserMessage | AssistantMessage | SystemMessage | ToolUseSummaryMessage

// 通道 2：仅用于 UI/持久化的消息（不进模型链）
type MetaMessage = ProgressMessage | SummaryMessage | TagMessage | AiTitleMessage

// 判定逻辑
function isChainParticipant(msg: Message): boolean {
  if (msg.type === 'progress') return false
  if (msg.type === 'system' && msg.subtype === 'virtual') return false
  return true
}

// 持久化时分离存储
function recordTranscript(msg: Message) {
  if (isChainParticipant(msg)) {
    appendToTranscript(msg)          // JSONL 主链
  } else {
    appendToMetadata(msg)            // 元数据附属条目
  }
}
```

**要点**：
- Progress 消息在 UI 层立即展示，但永远不进 transcript 链 — 不浪费模型 token
- 元数据条目（title、tag、summary）与 transcript 同文件但通过 `type` 区分 — 一个 JSONL 文件，两条逻辑通道
- 恢复会话时仅重建 chain 消息的 parentUuid 链，元数据条目独立处理
- Legacy 兼容：旧版 progress 消息有专门的桥接逻辑，不破坏新链

**适用场景**：聊天系统、协作编辑器、事件溯源架构、流式日志系统

---

## 模式 17：分层 System Prompt 与缓存作用域（Layered Prompt with Cache Scope）

**问题**：大型 system prompt 每次请求都全量发送，浪费 token 和延迟。但 prompt 中有些部分是全局不变的（工具定义），有些是组织级别的（策略），有些是每次请求动态变化的（上下文）。

**方案**：将 system prompt 拆成多个块，每个块标注缓存作用域，API 层按作用域决定是否复用缓存。

```typescript
type CacheScope = 'global' | 'org' | null  // null = 不缓存

type PromptBlock = {
  content: string
  cacheScope: CacheScope
}

function splitSystemPrompt(prompt: string, context: Context): PromptBlock[] {
  return [
    { content: TOOL_DEFINITIONS,      cacheScope: 'global' },  // 全局不变
    { content: ORG_POLICIES,          cacheScope: 'org' },     // 组织级缓存
    { content: DYNAMIC_BOUNDARY,      cacheScope: null },      // 动态边界标记
    { content: buildUserContext(context), cacheScope: null },   // 每次变化
  ]
}

// 特殊情况：MCP 工具存在时，工具定义不再全局稳定
if (hasMcpTools) {
  toolBlock.cacheScope = null  // 降级为不缓存
}
```

**要点**：
- 静态块（工具定义 ~数千 token）标记 `global`，跨会话复用 prompt cache
- 组织策略块标记 `org`，同组织内复用
- 动态边界之后的内容不缓存 — 每次请求都变化
- 当外部工具（MCP）存在时，全局块降级为不缓存 — 因为工具列表不再稳定
- 工具列表排序保持稳定 — 也是为 prompt cache 服务

**适用场景**：LLM 应用的 prompt 优化、CDN 多级缓存策略、模板引擎的局部缓存

---

## 模式 18：VCR 录制/回放测试（VCR Recording & Replay）

**问题**：流式 API 调用（如 LLM streaming）难以写确定性测试 — 结果不可复现、依赖外部服务、运行慢。传统 mock 无法捕捉真实的流式行为和边界情况。

**方案**：录制真实 API 交互为 fixture 文件，回放时返回录制的响应。通过输入标准化和路径脱敏确保 fixture 跨环境稳定。

```typescript
// 录制模式
async function* withStreamingVCR<T>(
  generator: AsyncGenerator<T>,
  fixtureKey: string,
): AsyncGenerator<T> {
  if (isPlaybackMode()) {
    yield* loadFixture<T>(fixtureKey)
    return
  }
  const recorded: T[] = []
  for await (const item of generator) {
    recorded.push(item)
    yield item
  }
  if (isRecordMode()) {
    saveFixture(fixtureKey, recorded)
  }
}

// 输入标准化 — 确保同样的逻辑请求映射到同一个 fixture
function normalizeForFixture(input: unknown): string {
  const dehydrated = dehydrate(input)    // 替换动态路径、时间戳、随机数
  return sha256(JSON.stringify(dehydrated))
}

// 脱敏
function dehydrate(input: unknown): unknown {
  return replaceAll(input, {
    [process.cwd()]: '<CWD>',
    [os.homedir()]: '<HOME>',
    [/\d{10,}/g]: '<TIMESTAMP>',
  })
}

// 回放时生成确定性 UUID，避免与 session 去重逻辑冲突
function hydrateMessage(msg: Message): Message {
  return { ...msg, uuid: deterministicUUID(msg) }
}
```

**要点**：
- 流式与非流式统一的录制/回放接口（`withVCR` / `withStreamingVCR`）
- dehydrate/hydrate 确保 fixture 跨平台稳定（路径、时间戳脱敏）
- SHA hash 作为 fixture 文件名 — 输入相同则命中同一 fixture
- 确定性 UUID 避免回放时 session resume 的去重误判
- 三种模式：record（录制）、playback（回放）、passthrough（真实调用）

**适用场景**：LLM API 测试、HTTP 交互测试（类似 Ruby VCR / Python vcrpy）、流式管道的集成测试
