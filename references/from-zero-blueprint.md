# 从零构建蓝图

基于 Claude Code 架构精华，为全新项目提供从第一行代码到生产就绪的构建路径。不是"迁移旧系统"，而是"一开始就把骨架搭对"。

---

## 核心理念

Claude Code 在 51 万行代码的规模下依然保持清晰的架构，核心秘诀不是事后治理，而是**从第一天就选对骨架**。以下蓝图提炼了这套骨架的搭建顺序。

---

## Phase 1：最小可运行骨架

**目标**：用最少代码搭出能跑通核心流程的骨架，同时预留正确的扩展点。

### 1.1 定义分层结构

从第一天就拆出三层，哪怕每层暂时只有一个文件：

```
src/
├── core/              # 编排层 — 主循环、状态机
│   ├── loop.ts        # 主循环（状态机驱动）
│   ├── state.ts       # State 类型定义
│   └── config.ts      # 不可变配置快照
├── execution/         # 执行层 — 工具/命令执行
│   ├── registry.ts    # 工具注册表
│   ├── executor.ts    # 串行/并发执行器
│   └── tools/         # 每个工具一个目录
├── policy/            # 策略层 — 权限、规则
│   ├── permissions.ts # 权限规则链
│   └── rules.ts      # 规则类型与匹配
├── services/          # 外部集成 — API、存储、协议
├── state/             # 状态管理
│   └── store.ts       # 极简 Store（34 行）
├── types/             # 类型定义
└── main.ts            # 入口
```

**为什么这么拆**：Claude Code 在 `query.ts`（编排）/ `toolOrchestration.ts`（执行）/ `permissions.ts`（策略）之间没有互相 import，全部通过接口和回调解耦。从第一天就遵守这个边界，后续就不需要大重构。

### 1.2 搭建主循环状态机

这是系统的心脏。不管做 Agent、工作流引擎、还是任务调度器，先把主循环写对：

```typescript
// core/state.ts
type State = {
  messages: Message[]
  context: Context
  recoveryCount: number
  turnCount: number
  transition: { reason: string } | undefined
}

// core/config.ts — 入口快照，循环内只读
type Config = {
  sessionId: string
  gates: Record<string, boolean>
}
function buildConfig(): Config { /* 快照当前环境 */ }

// core/loop.ts
async function* mainLoop(params: Params): AsyncGenerator<Event, Terminal> {
  let state: State = buildInitialState(params)
  const config = buildConfig()
  const deps = params.deps ?? productionDeps()

  while (true) {
    const { messages, context } = state
    // 1. 准备输入
    // 2. 调用核心服务 (deps.process)
    // 3. 处理结果
    // 4. 决策：continue 或 return
    if (shouldContinue(result)) {
      state = { ...state, transition: { reason: result.reason }, turnCount: state.turnCount + 1 }
      continue
    }
    return { reason: 'completed', result }
  }
}
```

**关键决策**：
- `State` 是唯一的可变容器 — 不要散落的 `let` 变量
- `Config` 在入口快照 — 循环内不要重读配置
- `deps` 可注入 — 第一天就获得可测试性
- AsyncGenerator — 第一天就是流式的

### 1.3 定义统一工具接口

每个可执行单元（工具、命令、动作）遵循同一个接口：

```typescript
// execution/registry.ts
type Tool<TInput = unknown, TOutput = unknown> = {
  name: string
  inputSchema: ZodSchema<TInput>
  isConcurrencySafe: (input: TInput) => boolean
  checkPermissions: (input: TInput) => PermissionResult
  execute: (input: TInput, context: Context) => AsyncGenerator<Progress | TOutput>
}

// 注册
const tools: Tool[] = []
function registerTool(tool: Tool) { tools.push(tool) }
function findTool(name: string) { return tools.find(t => t.name === name) }
```

**为什么从第一天就要这样**：Claude Code 有 40+ 工具全部遵循同一个 `Tool` 类型。如果一开始每个工具有自己的接口，后续统一的成本是 O(n)。

### 1.4 实现极简 Store

不引入任何状态管理框架，34 行搞定：

```typescript
// state/store.ts
function createStore<T>(initialState: T): Store<T> {
  let state = initialState
  const listeners = new Set<() => void>()
  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return
      state = next
      for (const l of listeners) l()
    },
    subscribe: (l: () => void) => { listeners.add(l); return () => listeners.delete(l) },
  }
}
```

### 1.5 最简权限检查

不需要完整的规则链，但每个外部操作必须经过一个检查函数：

```typescript
// policy/permissions.ts
type Decision = { type: 'allow' } | { type: 'deny'; reason: string } | { type: 'ask'; prompt: string }

function checkPermission(tool: Tool, input: unknown): Decision {
  // Phase 1: 简单的 allow all / deny list
  // 后续可扩展为完整规则链
  return { type: 'allow' }
}
```

**关键**：即使现在是 allow all，也要把 `checkPermission` 调用嵌入执行路径。Claude Code 的 `CanUseToolFn` 是必传参数而非可选 — 这保证了不会有工具绕过权限。

### 1.6 自定义错误类型

从第一天区分错误种类：

```typescript
// types/errors.ts
class AppError extends Error { constructor(message: string) { super(message); this.name = 'AppError' } }
class AbortError extends AppError {}
class ConfigError extends AppError {}
class ExternalServiceError extends AppError { constructor(message: string, public retryable: boolean) { super(message) } }
```

### Phase 1 完成标准

- [ ] 主循环能跑通：输入 → 处理 → 输出
- [ ] 至少 2 个工具通过统一接口注册和执行
- [ ] 权限检查在执行路径中（即使是 allow all）
- [ ] State 是显式类型、Config 在入口快照
- [ ] 核心循环可注入 fake deps 做单元测试

---

## Phase 2：可靠性与可观测

**目标**：让系统在出错时不崩溃、在出问题时能诊断。

### 2.1 错误恢复链

为每种预期失败定义恢复策略，而不是直接 throw：

```typescript
// core/loop.ts — 在主循环中
for await (const event of deps.process(input)) {
  if (isRecoverableError(event)) {
    if (state.recoveryCount < MAX_RECOVERY) {
      state = { ...state, recoveryCount: state.recoveryCount + 1, transition: { reason: 'recovery' } }
      continue  // 重试
    }
    // 超限才真正报错
  }
  yield event
}
```

**Claude Code 的做法**：`maxOutputTokensRecoveryCount` 最多恢复 3 次；`FallbackTriggeredError` 切换备用模型；`reactiveCompact` 在 prompt_too_long 时压缩上下文。每种错误都有"先试恢复"的路径。

### 2.2 优雅关停

在系统的第一个外部交互之前注册清理函数：

```typescript
// main.ts
const cleanupFns: (() => Promise<void>)[] = []
function registerCleanup(fn: () => Promise<void>) { cleanupFns.push(fn) }

process.on('SIGINT', async () => {
  for (const fn of cleanupFns.reverse()) await fn()
  process.exit(0)
})
process.on('SIGTERM', async () => { /* 同上 */ })
```

### 2.3 性能 Checkpoint

在关键路径上埋入计时点：

```typescript
const checkpoints: Record<string, number> = {}
function checkpoint(name: string) { checkpoints[name] = performance.now() }

// 使用
checkpoint('init_start')
await init()
checkpoint('init_done')
checkpoint('first_tool_start')
```

### 2.4 窄依赖注入

将主循环的外部依赖收敛为一个小类型：

```typescript
// core/deps.ts
type Deps = {
  process: typeof realProcess
  compact: typeof realCompact
  uuid: () => string
}
function productionDeps(): Deps {
  return { process: realProcess, compact: realCompact, uuid: crypto.randomUUID }
}
```

### 2.5 合成错误消息

当操作被取消/超时时，生成合成的结果消息而非留下未配对的请求：

```typescript
function createSyntheticError(operationId: string, reason: string): Result {
  return { id: operationId, error: true, message: `Cancelled: ${reason}` }
}
```

### Phase 2 完成标准

- [ ] 至少 1 种错误有透明恢复路径（消费者无感知）
- [ ] SIGINT/SIGTERM 时清理资源、flush 数据
- [ ] 关键路径有 ≥5 个性能 checkpoint
- [ ] 核心循环的 deps 可完全替换为 fakes
- [ ] 操作取消时不留未配对的请求/响应

---

## Phase 3：并发与性能

**目标**：让系统快起来，而不是在串行等待中浪费时间。

### 3.1 读写分区并发

```typescript
// execution/executor.ts
function partitionCalls(calls: Call[]): Batch[] {
  return calls.reduce((batches, call) => {
    const safe = call.tool.isConcurrencySafe(call.input)
    const last = batches.at(-1)
    if (safe && last?.isConcurrencySafe) {
      last.items.push(call)
    } else {
      batches.push({ isConcurrencySafe: safe, items: [call] })
    }
    return batches
  }, [] as Batch[])
}

async function* executeBatches(batches: Batch[]): AsyncGenerator<Result> {
  for (const batch of batches) {
    if (batch.isConcurrencySafe) {
      yield* runConcurrently(batch.items, MAX_CONCURRENCY)
    } else {
      yield* runSerially(batch.items)
    }
  }
}
```

### 3.2 级联取消控制器

```typescript
function createChildAbort(parent: AbortController): AbortController {
  const child = new AbortController()
  parent.signal.addEventListener('abort', () => child.abort(parent.signal.reason), { once: true })
  return child
}

// 用法：每个并发操作拿一个子 controller
const taskAbort = createChildAbort(batchAbort)
// Bash 错误 → abort batchAbort → 所有兄弟子 controller 级联取消
```

### 3.3 启动路径并行化

```typescript
// main.ts — 在 import 重模块之前触发 I/O
const configPromise = loadConfig()     // 不 await，先飞
const authPromise = prefetchAuth()     // 不 await，先飞

// 这些 import 需要时间，期间上面的 I/O 在并行跑
import { HeavyFramework } from 'heavy-framework'

// 现在 await
const config = await configPromise
const auth = await authPromise
```

### 3.4 重模块延迟加载

```typescript
// 不要在顶部 import 非核心重模块
// import { OpenTelemetry } from '@opentelemetry/sdk-node'  // ❌ 400KB

// 需要时才加载
async function initTelemetry() {
  const { NodeSDK } = await import('@opentelemetry/sdk-node')  // ✅ 按需
}
```

### Phase 3 完成标准

- [ ] 只读操作可并发执行，写入操作串行
- [ ] 并发操作有级联取消机制
- [ ] 启动路径 I/O 并行预取
- [ ] 非核心重模块延迟加载
- [ ] 有并发度上限配置

---

## Phase 4：扩展能力

**目标**：让系统可以被外部扩展，而不需要改核心代码。

### 4.1 Hook 生命周期协议

定义标准的生命周期事件和 Hook 执行协议：

```typescript
type HookEvent =
  | 'session:start' | 'session:end'
  | 'pre:execute' | 'post:execute' | 'execute:error'
  | 'pre:compact' | 'post:compact'

type HookResult = {
  decision?: 'block' | 'allow' | 'modify'
  message?: string
  context?: Record<string, unknown>  // 注入额外上下文
}

// Hook 执行器
async function executeHooks(event: HookEvent, input: unknown): Promise<HookResult[]> {
  const matchers = getRegisteredHooks(event)
  return Promise.all(matchers.map(h => runHook(h, input)))
}
```

**Claude Code 的选择**：Hook 是 shell 命令，通过 JSON stdin/stdout 通信 — 语言无关，任何能读写 JSON 的程序都能做 Hook。考虑你的场景选择：
- **进程内回调**：最简单，但限制语言
- **Shell 命令**：语言无关，但有进程开销
- **HTTP**：跨网络，但有延迟

### 4.2 插件目录约定

```
my-plugin/
├── manifest.json     # 名称、版本、描述、依赖
├── tools/            # 自定义工具
│   └── my-tool.ts
├── commands/         # 自定义命令
│   └── my-cmd.md
└── hooks/
    └── hooks.json    # Hook 配置
```

### 4.3 分级信任

```typescript
type TrustLevel = 'builtin' | 'verified' | 'community' | 'session'

const TRUST_CAPABILITIES: Record<TrustLevel, string[]> = {
  builtin:   ['all'],
  verified:  ['read', 'write', 'network'],
  community: ['read', 'write'],
  session:   ['read'],
}
```

### 4.4 Feature Flag 双轨

```typescript
// 构建时 — 完全消除未启用代码
const advancedModule = BUILD_FLAG_ADVANCED
  ? require('./advanced')
  : null

// 运行时 — 灰度/A/B 测试
const config = buildConfig()
if (config.gates.newFeature) {
  // 新路径
} else {
  // 旧路径
}
```

### Phase 4 完成标准

- [ ] ≥5 个生命周期事件可注入 Hook
- [ ] 第三方可通过目录约定添加工具/命令
- [ ] 插件有信任分级
- [ ] 支持运行时 Feature Flag

---

## Phase 5：生产就绪

**目标**：上线前的最后一轮加固。

### 5.1 上下文/资源预算管理

```typescript
type BudgetStrategy =
  | { type: 'truncate'; maxSize: number }           // 截断
  | { type: 'summarize'; trigger: number }           // 摘要
  | { type: 'evict'; policy: 'lru' | 'oldest' }     // 逐出

function applyBudget(items: Item[], strategy: BudgetStrategy): Item[] {
  // 分层触发，从轻到重
}
```

### 5.2 配置 Schema + 迁移

```typescript
// schemas/config.ts
const ConfigSchema = z.object({
  version: z.number(),
  maxRetries: z.number().default(3),
  features: z.record(z.boolean()).default({}),
})

// migrations/
const migrations: Migration[] = [
  { from: 1, to: 2, migrate: (old) => ({ ...old, features: {} }) },
]
function migrateConfig(raw: unknown): Config {
  let config = raw as any
  for (const m of migrations) {
    if (config.version === m.from) config = m.migrate(config)
  }
  return ConfigSchema.parse(config)
}
```

### 5.3 诊断工具

```typescript
async function doctor(): Promise<DiagnosticReport> {
  return {
    environment: checkEnvironment(),
    connectivity: await checkConnectivity(),
    permissions: checkPermissions(),
    config: validateConfig(),
  }
}
```

### Phase 5 完成标准

- [ ] 资源有预算管理（不等溢出才处理）
- [ ] 配置有 schema 验证和版本迁移
- [ ] 有自诊断工具
- [ ] 全部 Phase 1-4 的检查项仍然通过

---

## 决策树：每个阶段该不该做

在每个 Phase 开始前，问自己：

```
当前 Phase 的前一个 Phase 完成标准是否全部通过？
├── 否 → 先补完上一个 Phase
└── 是 → 当前系统是否有用户在用？
    ├── 否 → 跳过 Phase 5，继续迭代 Phase 1-4
    └── 是 → 用户量级？
        ├── <100 → Phase 3 可精简（并发/性能暂缓）
        └── >100 → 按顺序推进
```

---

## 反模式速查

| 你可能的冲动 | 为什么不应该 | 应该怎么做 |
|-------------|-------------|-----------|
| 第一天就引入 Redux/MobX | 34 行 Store 足够到 10 万行级别 | 用极简 Store |
| 第一天就建插件系统 | 核心价值还没验证 | 留空占位 `initPlugins() {}` |
| 每个工具自己定义接口 | 后续统一的成本 O(n) | 从第一天用统一 Tool 类型 |
| 全局变量存状态 | 蔓延不可控 | 分层 State + getter/setter |
| 串行初始化 | 浪费启动时间 | I/O 先飞，重模块后加载 |
| 只测 happy path | 生产环境 80% 是异常路径 | 每种错误有恢复策略 |
| 配置硬编码 | 每次改配置要发版 | Schema + 分层配置源 |
| 遥测上线后再加 | 出问题时已经晚了 | Phase 2 就埋 checkpoint |
