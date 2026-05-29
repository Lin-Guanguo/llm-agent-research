# Agent 分享材料：发现清单

Last Updated: 2026-05-28

## 文档定位

本文不是 blog 草稿，也不是正式分享稿。

目标是整理当前 Agent 调研中的关键发现，作为后续内部分享、对外科普、架构对比和 Dayfold Agent 设计讨论的参考材料。

写作原则：

- 先讲 Agent 的通用理解，再讲外部案例，最后再讲 Dayfold。
- 以发现、判断、对比和待补材料为主。
- 减少口语化叙述。
- Dayfold 不作为前文每一章的论证对象，只在最后作为具体业务案例展开。

## 1. Agent 与 Workflow 的边界

### 1.1 核心判断

- Agent 与 Workflow 更像连续谱，而不是二元分类。
- Workflow 引入 LLM 规划、工具调用、动态路由、验证和重试后，会逐渐接近 Agent。
- Agent 引入显式状态、工具边界、审批、调度和执行规则后，也会逐渐接近 Workflow。
- 因此，判断一个系统是不是 Agent，不如判断它的控制权、状态和反馈机制如何组织。

### 1.2 更有价值的问题

| 问题 | 关注点 |
|---|---|
| 谁决定下一步？ | 模型、程序、manager、router、scheduler |
| 状态放在哪里？ | 对话上下文、todo、graph state、typed runtime state、memory |
| 动作边界是什么？ | tool、capability、sandbox、approval、policy |
| 谁判断成功？ | 模型、测试、validator、evaluator、人 |
| 如何停止或恢复？ | no tool call、stop hook、loop condition、replan、human escalation |

参考表述：

> Agent 不是一个严格的产品类别，而是一组控制方式。真正重要的是：系统如何让模型感知状态、选择动作、接收反馈，并在必要时停止、重试或升级给人。

## 2. 现代 Agent 的最小循环

### 2.1 最小模型

现代 tool-calling agent 的核心循环通常很小：

```text
user request
-> build model context
-> model returns answer or tool call
-> runtime executes tool call
-> tool result returns to context
-> model continues or stops
```

关键点：

- 模型输出 tool call，runtime 执行工具并把 observation 放回上下文。
- 模型不再输出 tool call，通常意味着候选完成。
- 生产系统可以在候选完成处加入 stop hook、completion gate 或人工审批。

### 2.2 这个循环本身不够构成产品

真正的工程复杂度通常在循环外：

| 复杂度来源 | 作用 |
|---|---|
| Context assembly | 决定模型看到什么历史、文件、记忆、工具结果和系统状态 |
| Tool surface | 决定模型可以采取哪些动作，以及动作参数如何表达 |
| Permission / sandbox | 决定模型想做的动作是否真的允许执行 |
| Memory / skills | 决定系统如何跨任务复用历史和过程知识 |
| Summarization / compaction | 决定长任务如何在 token 限制下保持连贯 |
| Stop / recovery policy | 决定什么时候停止、继续、重试或升级 |

参考表述：

> 很多 Agent 看起来复杂，但内核可能都是 model -> tool -> observation -> model。真正区分 Agent 产品能力的，往往是外层 harness：上下文、工具、权限、记忆、生命周期和评估机制。

## 3. 为什么不同 Agent 看起来差很多

### 3.1 差异面

即使使用同一个模型、同一种 tool-calling 协议，不同 Agent 的行为也会差很多。主要差异来自以下 surfaces：

| Surface | 影响 |
|---|---|
| Prompt / policy | 工作风格、持久性、验证习惯、是否询问用户、何时停止 |
| Tool surface | 模型能做什么、如何表达动作、错误如何反馈 |
| Context assembly | 历史、文件、记忆、工具结果、摘要如何进入上下文 |
| Memory / skills / lifecycle | 系统能否跨 session 延续、复用 procedure、后台运行 |
| Runtime gates | sandbox、approval、permission、HITL、call limit |
| Subagents / background work | 是否能并行、隔离、委托、后台继续、定时唤醒 |
| Product integration | CLI、IDE、IM、browser、workflow platform、业务系统 |

### 3.2 三个核心经验

- Prompt 不是“只是 prompt engineering”，在 Agent 产品中它经常承担 soft policy。
- Tool surface 不只是能力暴露，也是在设计 Agent-Computer Interface。
- Runtime gate 是 guidance 和 control 的分界：prompt 可以建议模型不要做某事，但 runtime 才能真正阻止某个动作。

参考表述：

> Same model + same tool-call protocol does not make the same agent. Agent 的差异主要来自 harness：prompt policy、tool surface、context assembly、memory / lifecycle 和 runtime gates。

## 4. 外部案例一：Claude Code / Codex-style Coding Agents

### 4.1 定位

- 成熟的通用 coding agent。
- 代表 model-driven tool loop + structured progress artifact。
- 适合分析通用复杂任务如何通过 todo、工具反馈、测试和用户协作推进。

### 4.2 核心循环

```text
user request
-> build context
-> LLM outputs answer or tool call
-> runtime executes guarded tool
-> tool result returns to context
-> LLM decides next action
```

### 4.3 关键发现

| 维度 | 发现 |
|---|---|
| Plan 形态 | todo、plan mode、checklist、subagent task prompt |
| Plan 作用 | 模型注意力、用户协作、进度展示、approval |
| 下一步决策 | 主要由模型根据上下文和工具结果决定 |
| 完成判断 | 模型声明，测试/build/lint/diff/verifier/user review 强化 |
| 动作边界 | shell、file edit、file write、sandbox、approval、hooks |
| 可靠性来源 | 工具反馈、测试、verifier agent、用户审查 |

### 4.4 重要判断

- Coding agent 不是没有 plan，而是 plan 通常不是 executable runtime contract。
- Todo / checklist 更像模型的 working memory 和用户协作界面。
- Coding 场景适合 soft plan，因为任务开放、探索性强、反馈丰富。
- Coding 有较天然的 evaluator：测试、构建、lint、类型检查、diff、浏览器验证。

参考表述：

> Coding agent 的强项不是把任务预先编译成流程图，而是让模型在文件系统、shell、测试和用户反馈中持续探索。它的 plan 更像工作笔记，不是 runtime 调度协议。

## 5. 外部案例二：OpenClaw / Personal-Agent Products

### 5.1 定位

- Personal agent runtime shell。
- 代表长期个人助理、跨 channel、带 memory、带 identity、带 background lifecycle 的 Agent 产品形态。

### 5.2 关键发现

| 维度 | 发现 |
|---|---|
| 核心内循环 | 仍然接近 model-driven tool loop |
| 主要差异 | 外层 runtime shell，而不是 plan runtime |
| 状态重点 | session、identity、workspace、memory、channel、account |
| 动作边界 | owner-only tool、sandbox、exec approval、cron/wake/heartbeat gate |
| 长周期能力 | persistent session、cron、heartbeat、wake、subagent tracking |
| Plan 形态 | 模型上下文、任务描述、subagent prompt、memory/session history |

### 5.3 重要判断

- OpenClaw 不是 unstructured agent，它有大量 runtime control。
- 这些 control 主要用于身份、通道、工具、权限、后台任务和副作用边界。
- 它的中心问题是 personal continuity 和 access boundary。
- 它没有把 typed executable plan 作为主任务状态。

参考表述：

> OpenClaw 展示的是另一类 Agent 工程问题：如何把一个模型变成长期运行的个人助理。它的重点不是 plan runtime，而是 identity、session、memory、channel、background lifecycle 和 side-effect gates。

## 6. 外部案例三：LangChain / DeepAgents / TodoListMiddleware

### 6.1 定位

- 开发框架侧的主流 Agent stack。
- 代表 middleware-based tool loop + todo-mediated planning。
- 适合分析通用框架如何支持 long-task agent。

### 6.2 层级关系

```text
LangGraph
  = stateful workflow runtime

LangChain create_agent
  = prebuilt model-tool loop on LangGraph

TodoListMiddleware
  = todo state + write_todos tool + prompt guidance

Deep Agents
  = todo + filesystem + subagents + sandbox + memory + HITL harness
```

### 6.3 关键发现

| 维度 | 发现 |
|---|---|
| LangGraph | workflow runtime，不等于 agent planning 策略 |
| create_agent | 预置 model-tool-model loop |
| TodoListMiddleware | 给 agent 增加 todo working memory |
| Deep Agents | 把通用 long-task agent 常见能力打包成 harness |
| Plan 形态 | todo / filesystem notes / subagent task / harness state |
| 控制权 | LLM 仍然主导下一步 tool call |

### 6.4 重要判断

- TodoListMiddleware 是 soft planning infrastructure。
- Todo list 是 model-visible working memory，不是 runtime DAG。
- Middleware 是通用框架表达控制原语的重要机制。
- LangGraph 是 runtime substrate；LangChain / DeepAgents 是更上层的 Agent harness。

参考表述：

> LangChain 生态说明了一个重要事实：通用 Agent 框架更倾向把 planning 做成 middleware 和 model-visible state，而不是要求所有业务都先定义 executable workflow。

## 7. 从案例回收：Agent 控制原语

### 7.1 高阶原语

| 原语 | 关键问题 | 常见实现 |
|---|---|---|
| State | 系统显式记住什么？ | messages、todos、blackboard、graph state、port values、memory |
| Planner | 谁决定要做什么？ | LLM、manager、developer-authored workflow、typed planner |
| Router / Scheduler | 谁选择下一步？ | LLM loop、router、graph edge、program scheduler |
| Tool / Capability | 系统可以做什么？ | tool call、business capability、subagent、workflow node |
| Evaluator / Validator | 谁判断有效？ | test、lint、critic、judge、schema validator、semantic evaluator |
| Loop | 什么时候继续或停止？ | no-tool stop、loop condition、stop hook、replan policy |
| Memory | 历史如何进入当前任务？ | memory file、vector recall、session history、context engine |
| Side-effect Gate | 谁能阻止危险动作？ | approval、sandbox、permission、policy、HITL |

### 7.2 对“17 种 Agent 架构”的重新理解

原文短摘录，来自《从0开发大模型的17种Agent架构演进详细拆解》：

> "Agent architecture 的本质不是 prompt engineering"

> "新增了什么 state？" / "router？" / "evaluator？"

关键判断：

- 17 种模式更适合作为教学材料，不适合作为互斥 taxonomy。
- 多数模式是不同层级的控制原语组合。
- 判断一个“新架构”是否真的有新意，应看它新增了什么 state、router、evaluator、side-effect gate 或 recovery path。

示例归类：

| 模式 | 更好的解释 |
|---|---|
| Reflection | evaluator / quality loop |
| Tool Use | effect boundary |
| ReAct | observe-act loop |
| Planning | plan externalized as state |
| PEV | verification inserted into loop |
| Multi-Agent | work allocation |
| Blackboard | shared state + dynamic scheduling |
| Ensemble | redundancy + aggregation |
| Dry-Run | side-effect preview + approval |
| Metacognitive | boundary / escalation |

参考表述：

> 不要把 Agent 架构理解成模式名列表。更好的理解方式是：每个系统新增了什么显式状态、什么路由机制、什么 evaluator、什么副作用边界，以及什么恢复路径。

## 8. Plan 在 Agent 里的几种形态

### 8.1 Plan 不只有一种含义

| Plan 形态 | 主要用途 | 完成判断 | 典型系统 |
|---|---|---|---|
| Implicit Plan | 模型内部注意力 | 模型自行判断 | 普通 tool-calling agent |
| Text Plan | 人可读计划 | 模型或用户判断 | plan mode、chat planning |
| Todo List | 任务进度和 working memory | 模型更新 todo status | Claude Code、Codex、LangChain TodoListMiddleware |
| Ledger / Manager Plan | manager 协调 worker | manager 更新 ledger | Magentic-One、manager-agent systems |
| Developer Workflow | 程序定义执行图 | runtime graph state | LangGraph、Google ADK Workflow Agents |
| Executable Plan | runtime 可解释步骤 | runtime step state | Plan-and-Execute systems |
| Typed Executable Plan | typed capability + binding + validation | runtime state + validators | business-specific typed runtimes |

### 8.2 Soft Plan 与 Executable Plan 的核心差异

| 维度 | Soft Plan / Todo | Typed Executable Plan |
|---|---|---|
| 主要对象 | 模型和用户 | runtime |
| 表达形式 | 文本、todo、ledger | structured schema |
| 下一步决策 | 模型/manager | program scheduler |
| 依赖表达 | 自然语言或隐含上下文 | explicit input bindings |
| 完成判断 | model-declared，工具反馈强化 | runtime step state + validators |
| 优势 | 灵活、通用、低接入成本 | 稳定、可审计、低 token、可恢复 |
| 代价 | 不确定、难调度、难验证 | schema/runtime/validator 成本高 |
| 适合任务 | coding、research、open-ended tasks | business workflows with stable contracts |

参考表述：

> Plan 不是一个单一概念。Todo 是一种 plan，workflow graph 也是一种 plan，typed execution plan 也是一种 plan。区别在于：它是给模型看的，给人看的，给 manager 协调用的，还是给 runtime 执行的。

## 9. 最后案例：Dayfold Agent 的设计定位

### 9.1 适合放在最后的原因

前文先解释通用 Agent 的最小循环、差异面、外部案例、控制原语和 Plan 形态。Dayfold 应作为一个具体业务场景下的设计选择出现，而不是作为全文前半部分的论证目标。

### 9.2 Dayfold 的业务约束

| 约束 | 含义 |
|---|---|
| 低 token | 不希望每一步都回到 manager LLM 重新判断 |
| 高 one-shot 成功率 | 希望一次计划和执行尽量完成任务 |
| 稳定执行 | 多步骤业务动作需要可控 |
| 可审计 | plan、step state、port values 可以检查 |
| 可恢复 | 失败能定位到 step、binding 或 capability |
| 业务动作边界清晰 | 能力输入输出、权限和副作用应由 runtime 管控 |

### 9.3 设计定位

Dayfold Agent 不应主要描述为：

- 普通 ReAct agent；
- 多 agent role framework；
- LangChain DeepAgents-style harness；
- 固定 workflow DAG。

更准确的描述是：

> workflow-centric agent system with typed LLM-produced artifacts.

或者：

> Programmatic Plan-and-Execute with Typed Dataflow.

严格来说，它更像：

> workflow + internal subagents / capabilities.

顶层控制权不在一个自由行动的 LLM manager 手里，而在程序化 workflow runtime 里。LLM 参与其中，但主要作为 typed artifact producer：planner 产出 typed execution plan，worker/subagent 产出 typed output，validator/reviewer 产出 structured verdict。

### 9.4 三层结构

当前可以按三层理解：

| 层级 | 作用 | 说明 |
|---|---|---|
| Agent registration / configuration layer | 决定当前 Agent 的运行形态 | 可以理解为 Agent 的注册和运行配置：当前处在哪类业务阶段、使用哪套 planner policy、暴露哪些 capability、允许哪些 plan shape。这里由程序驱动，不需要在分享中展开成“外部状态机”。 |
| Stateful planner layer | 生成 typed plan | Planner 不是完全无状态 prompt。它会根据当前 request、历史状态、已完成步骤、planner policy、可见能力和 validation feedback 生成或修复 plan。 |
| Single-turn execution workflow | 执行这一轮 plan | Validator、coordinator、capability/subagent、binding、port values、runtime state 组成一次确定性的执行流程。 |

当前 planner state 可以先按三个业务状态理解：

| State | 入口 | 转换规则 | 作用 |
|---|---|---|---|
| `onboarding` | onboarding 入口 | 持续保持在 onboarding，不切到其他 state | 使用 onboarding 专用 planner policy 和能力约束。 |
| `chat` | chat 入口 | 在收集和确认到一定阶段后切换到 `story` | 负责对话式收集需求、更新 form、生成 ready/handoff artifact。 |
| `story` | 由 chat 切换进入，或作为正式生成流程进入 | 进入后不再切回 chat | 执行正式 story 生成或编辑流程。 |

因此，当前形态可以理解为两个入口、三个状态：`onboarding` 入口长期停留在 onboarding；`chat` 入口先进入 chat，满足条件后切到 story，并在后续保持 story。未来如果引入 OC 设计、画风设计等更多业务模式，状态数量和切换逻辑可能增加，但如果整体架构不大改，本质上仍然是同一类 planner state machine：程序状态决定 planner policy、能力集合和合法 plan shape，planner 在该状态下产出 typed plan。

简化为：

```text
program-driven agent state / registration
-> stateful planner
-> one-turn deterministic execution workflow
```

这个分层有助于避免把 Dayfold 讲成一个“LLM manager 调一组 worker”。更准确地说，外层程序状态决定当前 Agent 形态，planner 在该形态下产出 typed plan，runtime 再按 workflow 语义执行。

### 9.5 核心分工

| 层 | 职责 |
|---|---|
| Program State / Agent Registration | 决定当前 planner policy、capability surface 和合法 plan shape |
| Planner LLM | 理解用户意图，选择并组合业务能力，输出 typed execution plan |
| Validator | 检查 plan 结构、capability 合法性、binding 合法性 |
| Coordinator | 判断 step 依赖是否 ready，调度执行波次 |
| Capability / Internal Subagent | 接收 typed input，执行业务能力或局部 LLM 任务，产出 typed output |
| Port / Binding System | 将上游输出绑定到下游输入 |
| Runtime State | 记录 pending、ready、completed、failed、waiting、port values |
| Result Digest / Replan | 汇总结果，决定继续、重规划或结束 |

### 9.6 关键语义

| 概念 | 解释 |
|---|---|
| Tool call | Imperative action：模型要求 runtime 现在调用某个工具 |
| Structured output | Declarative artifact：模型产出结构化对象，由 runtime 解释其业务语义 |
| Executable plan | Macro tool argument：模型一次性提供较高层业务执行器的参数 |
| Internal subagent | Workflow node 内部可以使用 LLM 或 tool loop，但对外仍应暴露 typed input / typed output |

简化模型：

```text
program state / agent registration
-> Planner LLM
-> ExecutionPlan
-> validate(plan)
-> execute_plan(plan)
-> typed runtime state
```

### 9.7 取舍

收益：

| 收益 | 说明 |
|---|---|
| 低 token | 不需要每一步都回到 manager LLM 重新判断 |
| 稳定性 | scheduler、binding、validation 由程序负责 |
| 可审计 | plan、step state、port values 可检查 |
| 可恢复 | 错误可以定位到 step、binding、capability |
| 可复用 | capability contracts 可长期维护和组合 |
| 并行性 | ready steps 可以由 runtime fan-out |

复杂度：

| 复杂度 | 说明 |
|---|---|
| Plan schema | plan 成为系统协议，而不只是 prompt 输出格式 |
| Binding DSL | 上下游数据传递需要明确语义 |
| Validator | 结构、类型、capability、binding 都需要验证 |
| Planner prompt | 模型必须理解 capability contracts 和合法 plan shape |
| Runtime state | state 增长、压缩、恢复和可观测性都要设计 |
| Semantic gap | 类型正确不等于业务语义正确 |
| State machine boundary | 程序驱动状态机、planner policy、runtime state 的边界需要持续维护 |

参考表述：

> 通用 Agent 倾向把 plan 留在模型注意力、todo working memory 或 manager ledger 中，以换取开放任务的灵活性。业务型 Agent 如果更看重低 token、稳定性、审计和恢复，可以把 plan 外化为 typed executable contract，并交给 deterministic runtime 执行。

## 10. 复盘与后续补充材料

### 10.1 复盘方向

| 方向 | 说明 |
|---|---|
| Public research | 更早理解 soft plan、todo、manager、workflow、typed plan 的边界 |
| Plan IR | 更早把 plan 当作系统协议，而非 prompt 输出 |
| Evaluation harness | 更早评估 one-shot success、step failure、token cost、semantic error |
| State distinction | 区分 soft plan、executable plan、runtime state |
| Template fast path | 对高频稳定任务使用模板，减少 planner 推理成本 |
| Semantic validation | 从结构/类型验证推进到业务语义验证 |
| Middleware-like hooks | 将 planner policy、validation、approval、metrics、fallback 明确成 hook points |

### 10.2 评测指标候选

| 指标 | 用途 |
|---|---|
| One-shot success rate | 衡量一次规划和执行完成任务的能力 |
| Token cost per task | 衡量 runtime 是否真正降低模型调用成本 |
| Plan validation failure rate | 衡量 planner 输出合法性 |
| Step failure rate | 定位 capability 或 binding 问题 |
| Replan rate | 衡量 plan 稳定性和恢复需求 |
| Human intervention rate | 衡量自动化边界 |
| Semantic error rate | 衡量类型正确但业务错误的问题 |
| Recovery success rate | 衡量失败后是否能继续推进 |

### 10.3 待补材料

优先补充：

- 一张现代 tool-loop agent 最小循环图。
- 三个外部案例的压缩对比表。
- 一张 Plan 形态对比表。
- 一张 Dayfold runtime 简图。
- 脱敏失败案例。
- evaluator / metrics 设计。

可选补充：

- Claude Code / Codex 的 todo/plan 细节。
- OpenClaw 的 runtime boundary 细节。
- LangChain middleware 与 Dayfold hook points 的对应关系。
- “executable plan as macro tool argument”的图示。
