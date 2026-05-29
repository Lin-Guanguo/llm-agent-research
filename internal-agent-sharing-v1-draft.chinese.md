# 内部 Agent 分享文档 V1 Draft

Last Updated: 2026-05-25

## 草稿状态

这是基于 `internal-agent-sharing-outline.md` 起草的第一版中文正文。当前目标不是定稿，而是先把主线写出来，后续可以按章节继续补案例、图表、指标和复盘细节。

本文想回答的问题是：

> 在真实业务里，我们应该如何理解 Agent？为什么很多通用 Agent 选择 soft plan / todo-mediated loop，而我们的 Agent 更偏向 programmatic Plan-and-Execute with typed dataflow？

## 1. 现有 Agent 在解决什么问题

我现在更倾向于把 Agent 和 Workflow 看成一个连续谱，而不是两个严格分开的类别。

一个传统 workflow 通常是开发者提前写好流程：先做什么、再做什么、什么条件下走哪个分支。一个 Agent 则更强调模型在执行过程中根据上下文、工具结果和用户反馈动态决定下一步。但当 workflow 里面开始有 LLM 做规划、路由、工具调用、验证和重试时，它也会越来越像 Agent。反过来，一个 Agent 如果有明确的状态、工具边界、审批和执行规则，它也会越来越像 workflow。

所以问题不应该是“这个系统算不算 Agent”，而应该是：

- 谁决定下一步？
- 状态放在哪里？
- 工具和业务动作的边界是什么？
- 谁判断一步是否成功？
- 什么时候继续、重试、停止或者交给人？

为了避免把外部生态讲成产品百科，我只选三类系统作为参照。

### 1.1 Claude Code / Codex 类 coding agent

Claude Code 和 Codex 这类 coding agent 的核心循环很朴素：

```text
用户任务
-> 构造上下文
-> 模型输出回答或工具调用
-> runtime 执行工具
-> 工具结果回到上下文
-> 模型继续决定下一步
```

它们不是没有 plan。它们有 todo、plan mode、subagent、verification agent，也有 shell、文件编辑、测试、lint、diff 等反馈机制。但这些 plan / todo 主要是帮助模型维持注意力、向用户展示进度、协调复杂任务，通常不是 runtime 可以逐步调度的 executable plan。

这很适合 coding 场景。因为软件任务本身经常是开放的：模型需要先读代码，发现真实问题，再决定改哪里、跑什么测试、如何修复。执行过程中，计划会不断变化。并且 coding 有天然的外部 evaluator：测试、构建、类型检查、lint、浏览器验证、diff review。模型可以在工具反馈里反复试错。

所以 coding agent 选择 soft plan 是合理的。它牺牲了一部分确定性，换来了开放任务上的灵活探索能力。

### 1.2 OpenClaw / personal-agent 产品

OpenClaw 代表的是 personal agent 的方向。它的内核仍然接近 model-driven tool loop，但它的重点不在代码工作区，而在长期个人助理运行环境。

它关心的问题包括：

- 谁在和 agent 对话？
- 这个消息来自 DM、群聊、频道还是系统事件？
- 当前 session、workspace、memory、channel 如何恢复？
- 哪些工具对当前用户可见？
- cron、heartbeat、wake 这类后台入口如何触发？
- shell、消息发送、长期任务等副作用如何审批？
- subagent 如何创建、跟踪、终止和回传结果？

这说明 personal agent 的核心压力不是“把 plan 编译成 workflow”，而是“如何把一个模型变成长期运行、带身份、带记忆、带权限边界的个人助手”。

OpenClaw 对我们的启发是：真实 Agent 一旦接触现实系统，hard boundary 会变得很重要。仅靠 prompt 说“不要乱做”不够，runtime 必须知道哪些工具可以调用、谁有权限、哪些动作需要 approval、结果应该送回哪里。

但 OpenClaw 的 plan 仍然主要是 soft plan：它存在于模型上下文、任务描述、subagent prompt、session history 和 memory 中。runtime 有很多硬边界，但这些边界主要保护 agent shell，不是一个业务 typed plan runtime。

### 1.3 LangChain DeepAgents / Todo-mediated agents

LangChain / LangGraph / DeepAgents 这一组材料最适合作为开发框架参照。

它们其实处在不同层级：

```text
LangGraph
  = stateful workflow runtime

LangChain create_agent
  = prebuilt model-tool loop on top of LangGraph

TodoListMiddleware
  = 给 agent 增加 todo working memory

Deep Agents
  = 在 create_agent 上打包 todo、filesystem、subagents、sandbox、memory、HITL 等长任务 harness
```

这里最关键的是 `TodoListMiddleware`。它会给 agent 增加 todo 状态，注入 `write_todos` 工具，并通过 prompt 指导模型什么时候使用 todo。它让模型更容易维护复杂任务的进度，也让用户更容易看到 Agent 正在做什么。

但它依然是 soft planning。todo list 不是 runtime DAG。runtime 不会读取某个 todo，然后根据依赖关系自动调度对应能力。step 是否完成，主要还是由模型通过 `write_todos` 更新状态来声明。

这解释了为什么 todo-mediated planning 会成为通用 Agent 的主流方案：它便宜、灵活、框架无关、容易接到模型上下文里。但它也说明，通用框架默认不会替业务方定义 typed executable plan，因为那需要领域模型、能力 schema、binding 规则、validator、恢复策略和副作用边界。

## 2. 从模式名回到控制原语

那篇“17 种 Agent 架构”的文章很有启发，但我不太认同把它们当成 17 个互斥架构。很多所谓架构，其实是不同层级的控制原语，可以组合在一起。

更有用的问法是：这个系统新增了什么状态？新增了什么路由？新增了什么 evaluator？新增了什么副作用边界？

我现在会把 Agent 系统拆成几类原语：

| 原语 | 关注的问题 |
|---|---|
| State | 系统显式记住什么？ |
| Planner | 谁决定应该做什么？ |
| Router / Scheduler | 谁选择下一步可执行动作？ |
| Tool / Capability | 系统可以对外做什么动作？ |
| Evaluator / Validator | 谁判断结果是否有效？ |
| Loop | 什么时候继续、重试、停止或升级？ |
| Memory | 历史如何影响当前任务？ |
| Human / Side-effect Gate | 什么动作必须经过人或策略审批？ |

用这些原语看现有系统，会比背模式名更清楚。

ReAct / tool-use agent 的核心是模型驱动的观察-行动循环。模型看到上下文和工具结果，然后决定下一个 tool call。这种方式适合开放任务，但 token 成本和稳定性较难控制。

Todo / text plan agent 把 plan 外显出来，但主要作用是 attention management 和 progress tracking。它让模型不容易忘记任务，也让用户能看到进度，但通常不是程序调度的执行合同。

Manager / orchestrator agent 让一个 manager 去调 worker 或 tools。OpenAI guide 里的 manager、Anthropic 的 orchestrator-workers、Magentic-One 这类系统都可以放在这个方向里。它们的计划通常是 ledger、任务列表或 delegation state，由 manager 持续解释和更新。

Workflow / graph agent 则是开发者显式写图。LangGraph 就是很好的代表。它稳定、可审计，但动态性取决于开发者怎么设计 graph。

Programmatic Plan-and-Execute 则是另一种组合：LLM 生成结构化 plan，程序 runtime 验证、调度、绑定和执行。这个方向在通用开源框架里不是默认主流，因为它需要更强的业务假设。但对生产业务系统来说，它可能更适合低 token、高稳定性和可审计执行。

所以我现在的判断是：Agent 架构的核心不是模式名，而是控制权如何分配。

```text
模型负责什么？
程序负责什么？
状态在哪里？
谁能判断成功？
谁能阻止副作用？
失败后怎么恢复？
```

## 3. 我们的业务问题与 Agent 设计

我们的场景不是做一个通用聊天 Agent，也不是做一个开放式 coding agent。我们的目标更接近业务任务自动化。

这意味着约束不同：

- token 成本要低；
- oneshot 成功率要高；
- 多步骤执行要稳定；
- 中间状态要可审计；
- 失败要能定位、能恢复；
- 工具和业务动作边界要清楚；
- 业务能力要能够复用已有 typed contract。

在这样的约束下，如果完全依赖模型注意力维护计划，就会有几个问题。

第一，模型每一步都重新思考下一步，token 成本会比较高。第二，step 之间的数据依赖如果只靠自然语言描述，很容易出现漏传、错传或语义误解。第三，todo 标记完成不等于业务步骤真的完成。第四，失败恢复很难程序化，因为 runtime 不知道某个自然语言步骤对应什么业务能力、依赖什么输入、产出什么结果。

所以我们的 Agent 更适合描述为：

> Programmatic Plan-and-Execute with Typed Dataflow.

也就是：模型负责选择和组合业务能力，程序负责验证、调度、绑定、执行和恢复。

一个简化后的理解是：

```text
用户请求
-> Planner LLM 生成 typed execution plan
-> Validator 检查 plan 结构、能力合法性和 binding
-> Coordinator 判断哪些 step ready
-> Capability worker 执行 typed input -> typed output
-> Runtime 写入 step state 和 port values
-> Result digest / replanning / final response
```

这里最关键的不是“我们也有 plan”，而是 plan 的语义不同。

在 coding agent 里，plan 常常是模型的工作笔记。它帮助模型和用户对齐，但 runtime 不直接执行它。

在我们的系统里，plan 更像一个 typed execution contract。它至少要说明：

- 有哪些 step；
- 每个 step 调哪个 capability；
- 下游输入来自哪些上游输出；
- 哪些 step 可以并行；
- 执行结果如何进入后续状态。

这把一部分控制权从模型注意力转移到了程序 runtime。模型仍然重要，因为它负责理解用户意图、选择能力、构造计划。但计划一旦生成，后面的依赖判断、输入绑定、能力执行、结果写入，就不应该每一步都靠模型重新解释。

这个设计可以理解成一种 compiled manager / deterministic orchestrator-workers。

在普通 manager agent 里，manager 每一步都看上下文，然后决定调用哪个 worker。在我们的设计里，planner 先生成一个结构化计划，runtime 把这个计划解释成确定性的执行过程。manager 的一部分职责被“编译”进了程序 runtime。

## 4. 设计取舍与复盘

这个设计不是普遍更好，它只是更适合我们的约束。

和 Todo / Text Plan Agent 相比，我们更稳定、更可审计，但灵活性更低。Todo 适合开放任务，因为模型可以边做边改计划。Typed plan 适合业务流程，因为业务能力和数据结构相对可控。

和 Manager / Orchestrator Agent 相比，我们降低了反复 manager reasoning 的 token 成本，也让 runtime authority 更清楚。但代价是必须定义更强的 plan contract，不能只靠自然语言 delegation。

和固定 Workflow 相比，我们更动态。每轮用户输入都可以生成新的 plan，而不是完全走预定义 DAG。但代价是 planner、validator、binding、capability schema 都会复杂很多。

和 Multi-Agent Role Framework 相比，我们的重点不是有多少角色，而是 typed execution handoff。多个 agent 可以作为 worker 存在，但核心问题不是“拆成几个角色”，而是“上游输出如何可靠成为下游输入，runtime 如何知道一步是否 ready 和 done”。

这个方向的优点比较明确：

- token 成本更可控；
- 执行过程更可审计；
- 中间状态更结构化；
- 部分失败更容易定位；
- 对业务能力复用更友好；
- 对 oneshot 成功率更有帮助。

但代价也很明确：

- plan schema 会变复杂；
- planner prompt 更难写；
- binding 规则需要长期维护；
- validator 会越来越重要；
- 类型正确不等于语义正确；
- runtime 复杂度会明显高于普通 tool loop；
- 一些产品状态机会泄漏到 planner policy 里。

如果重做一次，我觉得有几件事应该更早做。

第一，更早调研公开方案。不是为了照抄，而是为了更早知道 soft plan、todo、manager、workflow、typed plan 的边界在哪里。

第二，更早定义 plan IR。plan 一旦成为 runtime contract，它就不只是 prompt 输出格式，而是系统核心协议。这个协议越早稳定，后面的 validator、debug、metrics 和 UI 才越好做。

第三，更早建立 evaluation harness。Agent 系统的难点不只是“能不能跑”，而是“失败率是多少、失败在哪里、是否可恢复、一次成功率如何、token 成本是否可控”。没有 evaluator，很难判断架构改动到底有没有变好。

第四，更早区分三种状态：

- soft plan：给模型看的计划；
- executable plan：给 runtime 执行的计划；
- runtime state：执行过程中真实发生了什么。

这三者混在一起时，系统会变得难 debug，也容易把模型声明的完成误认为真实完成。

第五，高频任务应该考虑 template fast path。不是所有任务都需要完整 planner 推理。对于稳定、高频、结构相似的业务任务，可以让程序提供模板，让模型只填关键参数或少量分支。

第六，要更重视 semantic validation。类型验证和结构验证只能保证“形状正确”，不能保证“业务语义正确”。真正生产化以后，semantic evaluator、人工审批、回放测试、对照样本和指标体系都会成为核心工程。

## 5. 暂定结论

我现在对这件事的理解是：Agent 开发已经逐渐从“控制原语探索”进入“如何按业务搭积木”的阶段。

早期大家都在摸索：ReAct、tool use、planning、multi-agent、reflection、memory、workflow、human-in-the-loop。现在这些组件已经比较清晰。真正的竞争会变成：

- 谁能把这些原语按业务场景组合得更稳；
- 谁能让工具和上下文更契合模型；
- 谁能把评测、回放、观测和恢复做好；
- 谁能在 token 成本、稳定性和灵活性之间做出合适取舍。

所以我们的设计不应该被表述成“比通用 Agent 更高级”。更准确的说法是：

> 通用 Agent 倾向于把 plan 留在模型注意力和 todo working memory 里，以换取开放任务的灵活性。我们的业务场景更需要低 token、高稳定性和可审计执行，所以选择把 plan 外化成 typed executable contract，并交给 deterministic runtime 执行。

这应该是这篇内部分享最核心的观点。

## 后续需要补的材料

- 三个外部案例的压缩对比表：Claude Code / Codex、OpenClaw、LangChain DeepAgents。
- 一张“soft plan vs executable plan vs typed executable plan”的对比表。
- 一张 Dayfold runtime 简图：Planner、Validator、Coordinator、Capability、Binding、State。
- 一组具体但脱敏的失败模式：错 plan、错 binding、能力输入不满足、语义正确性不足、用户中断、replan。
- 一组评测指标候选：oneshot success rate、token cost、step failure rate、replan rate、human intervention rate、semantic error rate。
