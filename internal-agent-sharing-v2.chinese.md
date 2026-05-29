# Agent 分享材料 V2：从通用 Agent 到 Dayfold 设计

Last Updated: 2026-05-29

## 文档定位

这是面向分享的第二版材料，不是 blog 成稿。

V1 更像发现清单，适合存放概念、表格和参考材料。V2 的目标是减少概念枚举，把内容组织成一条更容易讲的主线：

```text
Agent 的最小循环
-> 为什么不同 Agent 产品差异很大
-> 三个代表案例
-> Plan 的关键分歧
-> Dayfold 作为业务型 Agent 的设计选择
```

## 0. 核心主线

这次分享不试图回答“什么才算真正的 Agent”。这个问题边界很模糊，也不一定重要。

更有价值的问题是：

> 一个系统如何让模型看到状态、选择动作、接收反馈，并在必要时停止、重试或交给程序/人类处理？

从这个角度看，Agent 和 Workflow 不是两个完全分开的东西。Workflow 引入 LLM 规划、工具调用、动态路由和验证后，会越来越像 Agent；Agent 引入显式状态、权限、调度和执行规则后，也会越来越像 Workflow。

本文的核心判断：

1. 现代通用 Agent 的底层循环并不复杂，通常是 `model -> tool -> observation -> model`。
2. 真正拉开 Agent 产品差异的是外层 harness：prompt、tools、context、memory、permissions、lifecycle。
3. 通用 Agent 倾向使用 soft plan / todo，因为它们服务开放任务，需要灵活性。
4. 业务型 Agent 可以选择 Dynamic Workflow：由 planner 动态生成 workflow DAG，再由程序 runtime 执行。
5. Dayfold Agent 更适合被描述为 workflow-centric system，而不是一个自由行动的 LLM manager。

## 1. Agent 的底层循环其实很简单

现代 tool-calling Agent 的最小形态可以写成：

```text
user request
-> build model context
-> model returns answer or tool call
-> runtime executes tool call
-> tool result returns to context
-> model continues or stops
```

在这个循环里，模型并不是一次性回答问题，而是可以通过工具观察外部世界。只要模型继续发起 tool call，runtime 就继续执行并把结果放回上下文；当模型不再发起 tool call，runtime 通常把这次输出视为候选完成。

这解释了很多 Agent 产品的共同点。Claude Code、Codex、OpenClaw、LangChain `create_agent`、DeepAgents，内层都可以近似看成这种工具反馈循环。

但这个循环本身还不是一个好产品。真正困难的是：

- 模型每轮应该看到什么上下文？
- 工具应该如何设计，才能让模型少犯错？
- 哪些动作只是 prompt 建议，哪些动作必须 runtime 强制拦截？
- 长任务如何压缩历史？
- 什么时候应该停止、重试、回滚或让人审批？

所以，很多 Agent 的复杂度并不在“循环算法”本身，而在循环外面的 harness。

参考表述：

> 很多 Agent 看起来复杂，但内核可能都是 model -> tool -> observation -> model。真正区分 Agent 产品能力的，往往是外层 harness。

## 2. 为什么不同 Agent 产品差异很大

同样都在使用 LLM、上下文和工具/能力，为什么 Claude Code、OpenClaw、Dayfold Agent 会形成很不一样的系统？

因为同一个模型，或者相近的工具/能力调用机制，不会自动变成同一个 Agent。差异主要来自这些设计面：

- **Prompt / policy**：规定模型的工作方式、验证习惯、什么时候问用户、什么时候继续。
- **Tool surface**：决定模型可以采取哪些动作，以及动作参数和结果如何表达。
- **Context assembly**：决定历史、文件、记忆、工具结果、摘要如何进入模型上下文。
- **Memory / skills / lifecycle**：决定系统能否跨 session 延续、复用 procedure、后台继续。
- **Runtime gates**：决定哪些动作必须经过 sandbox、approval、permission 或 HITL。

这里有三个关键经验。

第一，prompt 不只是“提示词技巧”。在 Agent 产品里，prompt 经常是 soft policy。它告诉模型应该怎样工作，但它本身不能强制执行。

第二，tool surface 不只是能力列表。工具名、参数 schema、错误返回、路径语义、结果格式，都会影响模型如何行动。Anthropic 把这类问题称为 Agent-Computer Interface，这是很准确的。

第三，runtime gate 是 guidance 和 control 的分界。Prompt 可以告诉模型“不要直接执行危险操作”，但真正阻止危险操作的是 runtime 的权限、审批和 sandbox。

参考表述：

> Same model + same tool-call protocol does not make the same agent. Agent 的差异主要来自 harness：prompt policy、tool surface、context assembly、memory/lifecycle 和 runtime gates。

## 3. 三个代表案例

### 3.1 Claude Code / Codex：开放式软件任务

Coding Agent 的任务空间非常开放。用户可能让它修 bug、改架构、读代码、写测试、跑服务、查日志。模型往往不知道真实计划是什么，必须先探索环境。

这类系统的典型形态是：

```text
read files
-> inspect code
-> make todo / plan
-> edit files
-> run tests / build / lint
-> observe failures
-> revise
```

Claude Code / Codex 不是没有 plan。它们有 todo、plan mode、checklist、subagent task、verification agent。但这些 plan 多数是给模型和用户用的：帮助维持注意力、展示进度、申请 approval，而不是 runtime 逐步调度的执行合同。

这很合理。软件任务需要灵活探索，而且有天然 evaluator：测试、构建、lint、类型检查、diff、浏览器验证。模型可以在工具反馈里反复修正。

结论：

> Coding Agent 的 plan 更像工作笔记和协作界面，不是 runtime 调度协议。

### 3.2 OpenClaw：长期个人助理

OpenClaw 代表的是 personal agent。它的内层仍然接近 model-driven tool loop，但重点不在代码工作区，而在长期个人助理运行环境。

它真正要解决的问题包括：

- 谁在和 Agent 对话？
- 当前消息属于哪个 session、channel、account？
- 哪些 memory 应该进入上下文？
- 哪些工具只有 owner 可以使用？
- cron、heartbeat、wake 这类后台入口如何触发？
- shell、消息发送、长期任务等副作用如何审批？
- subagent 如何创建、跟踪、终止和回传结果？

所以 OpenClaw 的核心不是 workflow DAG runtime，而是 personal-agent shell：identity、session、memory、channel、background lifecycle、side-effect gates。

结论：

> Personal Agent 的中心问题是 continuity 和 access boundary，而不是把每个任务编译成 workflow。

### 3.3 LangChain / DeepAgents：框架化的通用 Agent

LangChain 这一组材料可以拆成三层：

```text
LangGraph
  = stateful workflow runtime

LangChain create_agent
  = prebuilt model-tool loop on LangGraph

DeepAgents
  = todo + filesystem + subagents + sandbox + memory + HITL harness
```

这里最相关的是 `TodoListMiddleware`。它给 Agent 增加 todo state，注入 `write_todos` 工具，并通过 prompt 指导模型什么时候使用 todo。

它是 planning，但属于 soft planning：todo 是 model-visible working memory，不是 runtime DAG。Runtime 不会读取某个 todo，然后自动根据依赖关系调度业务能力；下一步仍然由模型根据上下文决定。

这解释了为什么通用框架倾向于 todo-mediated planning。它足够通用、便宜、灵活，不要求业务方先定义 capability schema、binding DSL 和 validator。

结论：

> 通用框架更倾向把 planning 做成 middleware 和 model-visible state，而不是要求所有业务都先定义 executable workflow。

## 4. 从案例回到 Plan：关键分歧在哪里

前面三个案例共同说明一件事：通用 Agent 不是没有 plan，而是 plan 的语义不同。

Plan 的核心分类可以压缩成三类：

1. **Attention Plan**
   - 计划主要存在于模型上下文里。
   - Text plan、todo list、manager ledger、plan notebook 都可以归到这一类。
   - 作用是帮助模型维持注意力、进度和协作。
   - Runtime 通常不直接调度它。

2. **Workflow**
   - 计划由开发者写成代码、graph 或 DAG。
   - Runtime 按预先定义的流程执行。
   - 稳定、可审计，但动态性需要开发者提前设计。

3. **Dynamic Workflow**
   - Workflow DAG 不是完全由开发者预先写死，而是由 planner 在运行时生成或选择。
   - Runtime 仍然负责验证、调度、执行和记录状态。
   - 这是 Dayfold 更接近的方向。

这里最重要的分歧是：

| 问题 | Attention Plan | Workflow | Dynamic Workflow |
|---|---|---|---|
| 主要对象 | 模型和用户 | Runtime | Planner + runtime |
| 下一步谁决定 | 模型/manager | 程序流程 | Planner 生成 DAG，程序调度 |
| 依赖如何表达 | 自然语言或隐含上下文 | 代码/graph edge | workflow DAG + bindings |
| 完成如何判断 | model-declared + 工具反馈 | runtime state | runtime state + validators |
| 优势 | 灵活、通用、接入成本低 | 稳定、可审计 | 动态性和可执行性兼具 |
| 代价 | 难调度、难验证 | 动态性弱 | schema/runtime/validator 成本高 |

因此，不能简单说哪种 plan 更高级。Plan 形态应该匹配任务形态。

开放任务适合 Attention Plan，因为真实计划需要边观察边形成。固定业务流程适合 Workflow。业务流程如果需要动态组合能力，同时又要求执行稳定、可审计、可恢复，就可以考虑 Dynamic Workflow。

参考表述：

> Plan 不是一个单一概念。核心区别在于：它是留在模型注意力里，写死成 workflow，还是由 planner 动态生成 workflow DAG 再交给 runtime 执行。

## 5. Dayfold：业务型 Agent 的 workflow-centric 设计

Dayfold 的场景不是通用 coding agent，也不是长期个人助理，而是业务任务自动化。

核心约束是：

- 低 token；
- 高 one-shot 成功率；
- 稳定执行；
- 中间状态可审计；
- 失败可定位和恢复；
- 业务动作边界清晰。

在这个约束下，如果完全使用 model-driven tool loop，每一步都让模型重新判断下一步，成本和不确定性都会变高。Dayfold 的选择是把一部分控制权外化到程序 runtime。

### 5.1 定位：workflow + internal subagents / capabilities

Dayfold 不适合被描述成一个自由行动的 LLM manager。更准确的描述是：

> workflow-centric agent system with typed LLM-produced artifacts.

或者：

> Programmatic Plan-and-Execute with Typed Dataflow.

严格来说，它更像：

> workflow + internal subagents / capabilities.

顶层控制权在程序化 workflow runtime 中。LLM 参与其中，但主要产出结构化 artifacts：

- planner 产出 workflow DAG；
- worker/subagent 产出 typed output；
- validator/reviewer 产出 structured verdict。

### 5.2 三层结构

当前 Dayfold 可以按三层理解：

```text
program-driven agent state / registration
-> stateful planner
-> one-turn deterministic execution workflow
```

第一层是 Agent registration / configuration。它决定当前 Agent 的运行形态：使用哪套 planner policy、暴露哪些 capability、允许哪些 plan shape。

第二层是 stateful planner。Planner 不是完全无状态 prompt。它会根据当前 request、历史状态、已完成步骤、planner policy、可见能力和 validation feedback 生成或修复 workflow DAG。

第三层是 single-turn execution workflow。Validator、coordinator、capability/subagent、binding、port values、runtime state 组成一次确定性的执行流程。

当前 planner state 可以先按三个业务状态理解：

| State | 入口 | 转换规则 |
|---|---|---|
| `onboarding` | onboarding 入口 | 持续保持在 onboarding，不切到其他 state |
| `chat` | chat 入口 | 在收集和确认到一定阶段后切换到 `story` |
| `story` | 由 chat 切换进入，或作为正式生成流程进入 | 进入后不再切回 chat |

未来如果引入 OC 设计、画风设计等更多业务模式，状态数量和切换逻辑可能增加。但如果整体架构不大改，本质上仍然是同一类 planner state machine：程序状态决定 planner policy、能力集合和合法 DAG shape，planner 在该状态下产出 workflow DAG。

### 5.3 单轮执行链路

一轮执行可以简化成：

```text
Planner LLM
-> ExecutionPlan / workflow DAG
-> Planner validator validates / repairs / rejects the DAG
-> Coordinator schedules ready steps
-> Capability or internal subagent runs typed input -> typed output
-> Local validator checks subagent output
-> Runtime writes port values and step state
-> Result digest / replan / finish
```

这里的关键不是“模型也会规划”，而是 plan 的语义。

在 coding agent 中，todo 是模型工作笔记；在 Dayfold 中，planner 输出的是 runtime 可以执行的 workflow DAG。它需要说明：

- 有哪些 step；
- 每个 step 调哪个 capability；
- 下游输入来自哪些上游输出；
- 哪些 step 已 ready；
- 输出如何写回 runtime state。

### 5.4 Global state vs prompt context

Workflow-centric Agent 还需要区分 global state 和 prompt context。

```text
Global state != prompt context
```

Global state 是 runtime source of truth，可以包含 plan、step state、port values、artifacts、errors、checkpoints。它不应该默认全部塞进每个 LLM prompt。

Prompt context 是从 global state 投影出来的视图。每个 capability / subagent 应该拿到 scoped typed input，而不是整个全局上下文。

这个边界很重要。如果每个节点都看到同一大段隐式上下文，系统会逐渐退回 prompt-driven agent：节点开始从 ambient context 推断太多东西，typed workflow 的收益会下降。

工作规则可以写成：

```text
SubagentInput = exactly what this node needs
SubagentOutput = exactly what downstream/runtime consumes
GlobalState = runtime-owned source of truth
PromptContext = derived view, not raw global dump
```

### 5.5 Validator / repair / fallback 不是附属能力

Validator 至少有两层：

| 层级 | 验证对象 | 典型问题 |
|---|---|---|
| Planner validator | Planner 产出的 workflow DAG | capability 是否存在、binding 是否合法、依赖是否可达、DAG shape 是否符合当前 planner state |
| Subagent-local validator | 某个 capability / internal subagent 的结构化输出 | 字段类型虽然正确，但业务坐标、数量、范围、重复、遗漏等语义不正确 |

Planner validator 保护的是“这一轮 workflow 能不能被 runtime 执行”。Subagent-local validator 保护的是“某个节点内部的模型输出能不能被业务接受”。

Dayfold 的 page drift 是第二类问题。这里的问题不是 1-based / 0-based 约定混乱；业务里的 page index 本来就是 zero-based。问题是模型被要求保留原始绝对下标，但它倾向于把一个子批次重新从 0 开始编号。

例如这轮实际只要求改一组 zero-based 绝对下标中的部分页面：

```text
requested = [1, 2, 3, 4]
```

也就是说，page `0` 本轮不应该被改动。但模型返回时把这个 batch 局部编号成：

```text
returned = [0, 1, 2, 3]
```

这类错误通过 schema validation 很难发现，因为字段类型和结构都是合法的。问题不在“形状”，而在模型对局部列表的编号偏好：它把“要处理下标 1、2、3、4 的页面”重写成了“输出列表里的第 0、1、2、3 项”。

所以需要三层边界：

| 层 | 作用 |
|---|---|
| Schema validation | 检查字段、类型、结构 |
| Business validator | 检查业务不变量，例如 index set、重复、遗漏、越界 |
| Deterministic repair policy | 只修 runtime 能证明的错误 |

关键原则：

> Repair must be narrower than validation.

Validation 可以发现很多错误，但 repair 只能修复 runtime 能确定意图的错误。无法确定的错误应该进入 retry、fallback、drop 或显式失败。

这个案例说明：结构化输出只是让模型输出可解析，不等于业务语义正确。真正生产化的 structured-output agent 需要两类 validator：planner 侧验证 workflow DAG，subagent 内部验证本节点输出的业务语义，并配套 deterministic repair、fallback/retry 和 telemetry。

### 5.6 取舍

Dayfold 的收益：

- token 成本更可控；
- 执行过程更稳定；
- plan、step state、port values 可审计；
- 错误可以定位到 step、binding、capability；
- capability contracts 可以长期复用；
- ready steps 可以并行执行。

代价也明确：

- plan schema 成为系统协议；
- binding 规则变成 DSL；
- planner validator 和 subagent-local validator 都会越来越重要；
- planner prompt 必须理解 capability contracts；
- runtime state 需要压缩、恢复和观测；
- 类型正确不等于语义正确；
- 程序状态机、planner policy、runtime state 的边界需要持续维护。

参考表述：

> 通用 Agent 倾向把 plan 留在模型注意力、todo working memory 或 manager ledger 中，以换取开放任务的灵活性。业务型 Agent 如果更看重低 token、稳定性、审计和恢复，可以让 planner 动态生成 workflow DAG，并交给 deterministic runtime 执行。

## 6. 后续优化方向

下一步不应该继续堆更多 Agent 模式，而应该围绕 Dayfold 的 production quality 做评估。

优先方向：

- **Evaluation harness**：衡量 one-shot success、valid-plan rate、step failure、token cost、replan rate、fallback frequency。
- **Semantic validation**：解决“结构合法但语义错误”的 plan 或 structured output。
- **State compaction**：明确 port values、context messages、artifacts 什么时候从 active state 变成 archival state。
- **Template fast path**：对高频稳定流程使用 prevalidated templates，减少 planner 推理成本。
- **Planner state machine evolution**：如果未来业务状态更多，持续评估状态切换逻辑放在程序层、planner 层还是 runtime 层。

最终判断：

> Dayfold 的关键不是让模型每一步都更聪明，而是把模型擅长的“理解和组合”与程序擅长的“验证、调度和执行”拆开。
