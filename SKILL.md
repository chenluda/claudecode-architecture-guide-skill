---
name: claudecode-architecture-design-guide
description: >-
  Extract and apply architecture guidance from the Claude Code engineering system to other projects.
  Use when doing system architecture design, technical strategy, building agent/tool/plugin platforms,
  designing permission models, streaming pipelines, state machines, or phased migration planning.
---

# 系统架构设计指导

从 Claude Code 成熟工程体系中提炼的架构原则与设计模式，可用于从零构建新系统或改造现有系统。

## 核心目标

不是复制功能，而是迁移底层的工程原则、机制与演进路径。无论是从零开始还是改造已有系统，都从"选对骨架"出发。

## 必须产出的 4 个交付物

1. **精华映射**（Essence Map）— 原则 + 为何重要
2. **模式选择**（Pattern Selection）— 现在/稍后/跳过
3. **风险与控制**（Risk & Controls）— 失败模式 + 防护措施
4. **执行计划**（Execution Plan）— 分阶段路线图

## 工作流

### 第 1 步：定义目标项目上下文

- 起点：**从零构建** / 改造现有系统
- 阶段：0→1 原型 / 增长期 / 平台期
- 团队规模与所有权模型
- 可靠性与合规约束
- 预期规模与运行时画像

### 第 2 步：阅读并应用参考资料

- [architecture-principles.md](references/architecture-principles.md) — 12 条核心架构原则
- [pattern-catalog.md](references/pattern-catalog.md) — 15 个可迁移设计模式
- [review-checklists.md](references/review-checklists.md) — 架构评审清单
- [from-zero-blueprint.md](references/from-zero-blueprint.md) — **从零构建蓝图**（5 Phase 完整路径）
- [migration-playbook.md](references/migration-playbook.md) — 现有系统迁移手册

### 第 3 步：选择路径

- **从零构建** → 按 [from-zero-blueprint.md](references/from-zero-blueprint.md) 的 Phase 1-5 顺序推进
- **改造现有系统** → 按 [migration-playbook.md](references/migration-playbook.md) 选择对应阶段路径

### 第 4 步：按 ROI 选择模式

| 优先级 | 标准 | 示例 |
|--------|------|------|
| **立即采用** | 高风险消减、低复杂度 | 权限分层、优雅关停 |
| **下一阶段** | 中等工作量、高杠杆 | 流式工具执行、自动压缩 |
| **推迟/跳过** | 依赖生态或规模条件 | 多 Agent 协调、插件市场 |

### 第 5 步：生成决策日志

每个被选模式需记录：

- 选用的模式名称
- 被否决的替代方案
- 成功度量指标
- 负责人与上线阶段

## 输出模板

```markdown
# 系统精华迁移计划: <目标项目>

## 1) 精华映射
- 原则: <名称> — <为何重要>

## 2) 模式选择
### 立即采用
- <模式> — <理由> — <成功指标>

### 下一阶段
- <模式> — <理由> — <成功指标>

### 推迟/跳过
- <模式> — <理由与前置条件>

## 3) 风险与控制
- 风险: <失败模式> — 控制: <防护措施>

## 4) 分阶段路线图
### Phase 1：骨架搭建
- <交付物>

### Phase 2：加固与可观测
- <交付物>

### Phase 3：扩展与生态
- <交付物>

## 5) 开放问题
- <问题>
```

## 12 条核心架构原则

详见 [architecture-principles.md](references/architecture-principles.md)。

1. **分离编排、执行与策略层** — query.ts（编排）/ toolOrchestration.ts（执行）/ permissions.ts（策略）无交叉 import
2. **Agent 循环即显式状态机** — 类型化 State 集中可变数据，Config 入口快照不可变，Continue/Terminal 类型化转移
3. **工具并发的读写分区** — 只读工具并发执行，写入工具串行；isConcurrencySafe 在工具侧声明
4. **依赖注入要窄、要有工厂函数** — 4 字段 QueryDeps + productionDeps() 工厂；typeof fn 自动同步类型
5. **最小化状态管理** — 34 行 Store；四层状态层次（进程→应用→查询→工具）
6. **Feature Flag 驱动的渐进式上线** — 构建时 feature() 死代码消除 + 运行时 statsig 灰度
7. **启动路径并行化与延迟加载** — I/O 先飞再 import 重模块；非核心模块按需 import()；memoize(init)
8. **流式架构以 AsyncGenerator 为核心** — 背压友好的 yield* 组合贯穿整条管线
9. **多层权限模型** — deny→ask→check→sandbox→classifier→mode 分层决策链，结果类型化
10. **优雅降级与恢复链** — max_output_tokens 恢复、模型 fallback、流式回退、reactive compact
11. **Hook 系统的生命周期覆盖** — 15+ 生命周期事件；JSON-in/JSON-out，语言无关
12. **插件架构的渐进式信任** — 内置 > 市场 > Git > 会话，分级能力 + SSRF 防护

## 决策规则

- 优先渐进式强化，而非重写
- 优先确定性行为，而非隐式聪明
- 优先一个稳定接口，而非多个临时扩展
- 优先显式预算/限制，而非尽力而为假设
- 优先 AsyncGenerator 流式，而非回调地狱
- 优先组合式依赖注入，而非全局单例
