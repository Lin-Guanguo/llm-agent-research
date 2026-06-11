# 从工程实践上理解 Agent，核心很简单

Last Updated: 2026-06-09

副标题：通用 Agent 保持最小循环，专用 Agent 开始比拼业务组装和效果评估。

从工程实践上看，Agent 的核心并不复杂：模型在上下文里判断下一步，要么调用工具并观察结果，要么输出结果并结束。

真正复杂的是外层系统：怎样把模型的不确定性，放进一个可观察、可约束、可恢复的工程结构里。

我的判断是：通用 Agent 的主形状已经基本收敛。它们会保持在最简单、最灵活的 model-tool-observation loop 上，用更丰富的 tool surface 和更强的 harness 承接复杂性。

专用 Agent 的重心则变了。它不追求覆盖所有未知任务，而是在相对稳定的业务场景里，把一类流程重复跑很多次。基础积木已经齐全之后，真正的问题变成：怎么组合这些组件，才能在长期运行里稳定交付，而不是偶尔跑通。

这里的“效果好”不是 demo 里能跑一次，而是在大量重复任务里，失败能被看见、能被处理，高风险动作不会越界。

这是我最近读 Anthropic / OpenAI 的 Agent 工程文章、研究 Claude Code / Codex / OpenClaw / LangGraph 等系统后，越来越强的感受。

早期大家还在探索“Agent 到底该怎么写”。从 ReAct 到 Planning、Memory、多 Agent，各种名字都很有启发。那时更像是在确认 Agent 系统需要哪些控制原语。

但到今天，很多底层组件已经比较清晰了。模型调用工具、接收 observation 已经成为标配；计划、记忆、subagent、长程任务、验证和审批，也逐渐沉淀成常见的 harness 能力。

所以问题正在从“有哪些 Agent 范式”转向“怎么组合这些原语”：组件是否贴合业务，结构是否顺着模型能力，质量下限是否能通过验证和评估被持续拉上来。

这篇文章想讨论的就是这个变化：通用 Agent 为什么保持简单，专用 Agent 为什么开始需要按业务搭积木。

## 1. Agent 的最小模型

现代 tool-calling Agent 的底层循环可以简化成：

```text
model -> tool call -> observation -> model
```

模型看到上下文，决定要不要调用工具。Runtime 执行工具，把结果放回上下文。模型再根据新的 observation 决定下一步。如果模型不再发起 tool call，系统通常认为这一轮可以结束。

这个思想最早可以追溯到 ReAct。ReAct 让模型交替生成 reasoning trace 和 action，典型形式是：

```text
Thought -> Action -> Observation -> Thought ...
```

早期实现还没有今天的 function call / tool call 协议，本质上是让模型输出一段文本 action，再由宿主程序解析并执行。今天的模型把 tool call 做成了一等公民，但核心循环没有变太多。

Claude Code、Codex、OpenClaw、LangChain `create_agent`，内层都可以近似看成这种 model-tool-observation loop。

所以一个很重要的判断是：

> Agent 的核心循环很简单，复杂的是外层 harness。

这里的 harness 指的不只是 prompt，而是 prompt 外面的运行系统。它决定模型能看到什么、能做什么，以及每一步结果如何被约束、恢复和评估。过去我们说 prompt engineering，后来开始说 context engineering，现在很多真实系统其实是在做 harness engineering。

这也是为什么通用 Agent 的形状会收敛到最小循环。开放任务需要最大灵活性，越早把流程写死，越容易限制模型根据 observation 调整路径。通用 Agent 的复杂度不会消失，但它更多沉到 harness 和 tool surface 里，而不是把主循环变成很重的固定 workflow。

## 2. Tool Surface 决定 Agent 的产品形态

如果 ReAct 给的是最小循环，那么复杂 Agent 的下一步，不是让模型“想得更多”，而是设计模型到底能做哪些 action。

这里的 tool 不只是普通 API。文件读写、shell、计划管理、skill、subagent、长程任务，都可以被包装成模型可调用、runtime 可约束、结果可回到上下文的行动接口。

这就是 tool surface。

同样是 model-tool-observation loop，不同产品用起来差异很大，差异很大一部分来自 tool surface。

### Coding Agent

以 Claude Code / Codex 这类 coding agent 为例，软件任务的不确定性很高。用户可能让它修 bug、读代码、跑测试、改架构。模型一开始通常不知道真实计划是什么，需要先观察文件系统和运行结果。

所以 coding agent 的 tool surface 通常围绕几类能力展开：

- 文件读写：读代码、改文件、写新文件；
- 搜索定位：按文件名、文本、符号或引用查上下文；
- 命令执行：跑测试、构建、脚本、诊断命令；
- 计划管理：维护 todo、计划、任务进度；
- 委托执行：启动 subagent，分担搜索、分析、实现或 review；
- 后台任务：让长时间运行的命令或子任务在另一个生命周期里继续。

对 coding agent 来说，shell 几乎成了开放式软件任务的标准行动接口。测试、build、diff、日志，都可以作为 observation 回到循环里，让模型继续调整下一步。

### Personal Agent

OpenClaw 这类 personal agent 的重点不一样。它不是把 Agent 主要放进代码仓库，而是放进长期运行的个人助理环境。

它的 tool surface 更关心：

- 记忆和长期偏好；
- 会话和多渠道消息；
- 定时任务、heartbeat、未来唤醒；
- 浏览器、外部应用和设备；
- 用户身份、权限和副作用边界；
- 子会话、子 agent 和后台生命周期。

Personal agent 的难点不是单次任务计划，而是 continuity 和 access boundary：它要知道“谁在和我说话”“这属于哪个 session”“哪些记忆相关”“哪些动作需要确认”“后台任务完成后怎么回到用户上下文”。

### 异步 Tool

这里还要单独注意一类 tool：异步 tool。

普通 tool 更像函数调用：模型发起调用，runtime 执行，结果立刻作为 observation 回到上下文。

但 cron、heartbeat、subagent、background task 更像异步 tool：一次 tool call 启动或登记一个长生命周期执行，真实工作在另一个生命周期里并发进行。完成后，结果再通过 poll、retrieve、runtime notification、wake event、session update 或新的 user message 回到 Agent 上下文。

这说明 tool surface 不只是“函数集合”。它决定模型能触达什么空间：文件系统、shell、记忆、会话、后台任务、未来时间点、外部系统，甚至其他 Agent。

这也是通用 Agent 继续演化的主要方向。它们未必会把主流程变得很复杂，但会持续扩展模型可使用的行动空间，并通过更好的上下文、权限和评测来提高上限。

## 3. 范式研究沉淀成控制原语清单

有些文章会把 Agent 架构拆成很多种。比如《[从0开发大模型的17种Agent架构演进详细拆解](https://mp.weixin.qq.com/s/5f0I2apY4oFsHrttANBOJg)》里提到 Reflection、Tool Use、ReAct、Planning、PEV、Multi-Agent、Blackboard、Memory、ToT、Dry-Run、Metacognitive、Self-Improve 等。

这些分类有价值，但不应该当成互斥 taxonomy。很多模式并不在同一个抽象层级，也经常可以组合。

我现在更倾向于把它们看成控制原语清单，而不是互相排斥的“17 种架构”。它们的价值不在于告诉你应该选哪个名字，而在于提醒你：当系统出现某类失败模式时，可以引入哪种 state、control flow、router 或 evaluator。

更有用的看法是：每种设计都在解决一个具体问题。

可以问四个问题：

1. 新增了什么 state？
2. 新增了什么 control flow？
3. 新增了什么 router / gate？
4. 新增了什么 evaluator？

例如：

| 问题 | 常见设计 | 本质 |
|---|---|---|
| 输出质量不稳 | Reflection / Self-Improve | 增加 critique、revision 和质量闭环 |
| 模型无法触达外部世界 | Tool Use / ReAct | 增加行动接口和 observation loop |
| 多步任务容易失忆 | Todo / Plan / Workflow | 把步骤、进度和终止条件显式化 |
| 工具失败会静默传播 | PEV / Validator | 把验证接入主回路 |
| 单个 prompt 角色太多 | Multi-Agent / Subagent | 把认知任务拆成角色或上下文隔离单元 |
| 跨轮状态丢失 | Memory / Session / Blackboard | 把历史状态和中间产物显式保存 |
| 副作用风险高 | Dry-run / Approval Gate | 把真实动作放进闸门 |
| 路径会分叉且需要回溯 | ToT / Search / Simulation | 把推理变成搜索或预演 |

这些设计本质上是在处理 LLM 进入工作流之后带来的不确定性。它们不是越多越好，而是要在问题真的出现时才引入。

最小 tool loop 适合开放探索，因为模型可以根据 observation 自由调整路径；约束加得太早，可能限制模型发挥。

所以通用 Agent 会尽量保留这个最小 loop。它不是没有 plan、没有 memory、没有 subagent，而是通常不把所有任务都强行编译成一个标准 workflow。它宁愿多花 token、多做 observation，也要保留开放任务里的动态性。

但专用场景的要求不同。面对固定业务场景和会被重复执行成千上万次的流程，系统不能只追求少数情况下的高上限，还要保证大多数情况下的质量下限。这时就需要把一些原本靠模型临场判断的东西，逐步变成显式 state、workflow、validator、repair、fallback 和 evaluation。

这会形成两条优化路线。

通用 Agent 更关注能力上限。它的主形状已经比较固定：最小 loop + 足够丰富的 harness。它会不断增加更适合模型使用的 tool、更好的上下文组织、更强的模型、更好的评测，但不太愿意把流程限制得太死，因为开放任务的价值就在于模型可以自己探索。

专用 Agent 更关注质量下限。它也需要模型能力，但不会把所有决策都交给模型。它会把高频、稳定、可验证的业务流程收束成显式控制流，用程序负责验证、调度、恢复和评测。

## 4. Plan 是通用和专用的分水岭

到这里，问题可以收束到一个词：Plan。

Plan 解决的不是“模型会不会想”，而是复杂过程中 Agent 还能不能稳定把事做完：不要失忆、不要跑偏、不要重复劳动、不要不知道什么时候停止。

它有点像人在按 SOP 做事。SOP 未必提高能力上限，但能减少低级错误，提高交付质量下限。

从“搭积木”的角度看，Plan 是最关键的中间件。它决定模型的自由探索，什么时候变成 runtime 能验证、调度和恢复的结构。

我倾向于把 Plan 分成三类。

### 4.1 Context Plan

第一类是 Context Plan。

计划主要给模型和用户看，不是 runtime 的标准调度协议。常见形式是 todo list、文字计划、plan mode、manager ledger。

它的作用是帮助模型维持注意力、记录进度、和用户协作。

例如很多 coding agent 会维护一个 todo list。这个 todo list 让用户看到当前进度，也让模型不要忘记任务。但 runtime 通常不会读取 todo list 自动调度下一步。下一步做什么，仍然由模型根据上下文判断。

Context Plan 的优点是灵活，适合开放任务。缺点是 runtime 很难证明它真的完成了，也很难把它作为可恢复、可审计、可验证的执行协议。

这也是通用 Agent 最常见的计划形态：plan 帮助模型和用户协作，但主控制权仍然留给模型。

### 4.2 Workflow

第二类是 Workflow。

计划由开发者写成 graph、DAG 或代码。节点、边、条件、循环、人工确认点都由开发者明确设计，runtime 按图执行。

这类方式的优点是稳定、可审计、容易恢复。开发者知道有哪些步骤，每一步输入输出是什么，失败后走哪条分支，哪些动作必须审批。

缺点也很明显：动态性弱。能走哪些路径，通常需要提前设计。对高度开放的任务，如果一开始就强行写死 workflow，系统会变重，也会牺牲模型根据 observation 动态调整的能力。

Workflow 适合稳定流程。例如客服工单分类、邮件处理、审批流、数据清洗、固定生产管线。这类任务路径相对清楚，模型更适合负责局部判断或内容生成，runtime 负责流程。

### 4.3 Dynamic Workflow

第三类是 Dynamic Workflow。

它的共同点是：workflow 不完全由开发者预先写死，而是在运行时由模型、planner 或其他程序生成、选择或修复，再交给 runtime 执行。

Claude Code 的 Dynamic Workflows 是一个典型例子。Claude 生成 JavaScript orchestration script，script 负责循环、分支、并行调 subagents 和保留中间结果；runtime 在后台执行这个脚本。

这类模式的意义在于：让模型参与“生成执行结构”，但执行过程不完全塞回主对话上下文。中间状态、循环、并行、验证和收敛逻辑可以放到 runtime 里。

Dynamic Workflow 试图同时拿到两件事：

- 模型带来的动态性；
- runtime 带来的稳定性。

代价是系统复杂度上升：plan schema、binding、validator、state management、replay、resume 都要设计。

这也是专用 Agent 最值得关注的区域：它不像固定 Workflow 那样完全写死，也不像 Context Plan 那样完全依赖模型注意力，而是在业务边界内动态生成执行结构。

## 5. Planned Workflow：让业务流程成为 Runtime Contract

如果我们做的是一个通用 coding agent，让模型拿着工具自由探索是合理的。它面对的是开放任务，测试和 diff 又能持续提供反馈。

但很多专用 Agent 不是这样。

以一个内容创作类业务 Agent 为例。它可能要把用户输入转换成多步骤生产流程：

1. 理解用户意图；
2. 选择风格；
3. 生成故事或脚本；
4. 增强页面描述；
5. 生成角色参考；
6. 生成图片；
7. 生成音频；
8. 汇总结果并返回给前端。

如果每一步都让模型重新判断下一步，系统会非常灵活，但成本和不确定性都高。模型可能重复做事、跳过必要步骤、错误引用历史产物、或在业务边界之外“发挥”。

这时，问题不再是“Agent 有没有能力调用这些工具”，而是“这些工具该按什么顺序组合，哪些步骤可以并行，哪些输出必须被验证，失败后怎么恢复”。

所以这类系统更适合一种 Plan-and-Execute：

```text
planner 生成 workflow DAG
-> runtime 验证并执行
-> internal subagents / capabilities 完成具体步骤
```

顶层不是一个自由行动的 LLM manager，而是程序化 workflow runtime。LLM 参与其中，但主要产出结构化 artifacts：

- planner 产出 workflow DAG；
- subagent / capability 产出 typed output；
- validator / reviewer 产出 structured verdict。

一个脱敏后的 plan 大致像这样：

```json
{
  "reply_message": "收到，正在生成这一轮内容。",
  "output_summary": "根据用户输入生成一组多步骤创作任务。",
  "execution_steps": [
    {
      "step_id": "style_select_0",
      "capability_id": "style_select",
      "input_bindings": {
        "intent": "input.user_text",
        "references": "input.reference_assets"
      }
    },
    {
      "step_id": "draft_write_0",
      "capability_id": "draft_write",
      "input_bindings": {
        "intent": "input.user_text"
      }
    },
    {
      "step_id": "page_enhance_0",
      "capability_id": "page_enhance",
      "input_bindings": {
        "style": "style_select_0.style_description",
        "pages": "draft_write_0.pages"
      }
    },
    {
      "step_id": "media_render_0",
      "capability_id": "media_render",
      "input_bindings": {
        "tasks": "page_enhance_0.render_tasks",
        "style_id": "style_select_0.style_id"
      }
    }
  ]
}
```

这个 Plan 不是给模型看的步骤文本，而是 runtime 可以验证、调度和执行的 contract。每个 step 都有 `step_id`、`capability_id` 和 `input_bindings`。Runtime 可以检查：

- capability 是否存在；
- binding 是否引用了可用输出；
- DAG 是否有环；
- 必要步骤是否缺失；
- 当前业务模式下是否允许调度这个能力。

这里还有一个实际设计点：专用 Agent 往往不应该让同一个 planner prompt 覆盖所有对话场景。

端上通常已经有明确入口差异：闲聊 / 需求收集、正式生成、首次引导、角色设定、编辑调整。把这些差异作为 request mode 传入 Agent，可以决定 planner 形态：

```text
request mode
-> planner prompt / policy
-> planner 可见的 capability 集合
-> planner 生成对应场景下的 workflow DAG
```

这样做不是把业务逻辑完全写死，而是在 Plan-and-Execute 之前先做一次场景收束。让 planner 在更小、更明确的场景里生成 DAG，牺牲一点通用性，换来更清晰的 planner policy、更少的可选动作和更稳定的输出下限。

## 6. 搭得好不好，靠程序化设施兜底

真正落到生产系统里，Plan 不能只是“模型生成出来的一段结构”。它必须能被验证、修复、观测和评估。

我认为至少有三层设施。

### 6.1 Plan-level Validation

Planner 产出的 workflow DAG 要先被检查：

- capability 是否存在；
- binding 是否合法；
- 依赖是否可达；
- DAG shape 是否符合当前 planner policy；
- 当前业务模式下是否允许这些步骤。

它保护的是：这一轮 workflow 能不能被 runtime 执行。

### 6.2 Step-level Validation and Repair

某个 subagent 即使输出了合法 JSON，也可能在业务语义上错。

例如某个任务要求只处理第 1、2、3、4 页，page index 本来就是绝对下标，但模型容易把这组 batch 重新局部编号，输出成 0、1、2、3。

这不是类型错误。`page_index` 仍然是整数，结构也合法。错误在业务语义：模型把绝对下标改成了 batch-local 下标。

这类问题说明：

> structured output 只解决“可解析”，不保证“业务正确”。

有时更好的解法不是继续强调“不要弄错数字”，而是改变任务表示。例如不要让模型直接输出容易漂移的数字下标，而是给每个对象稳定代号，让模型在更不容易偏移的表示空间里工作。

Repair 也要非常克制。一个原则是：

> Repair must be narrower than validation.

Validator 可以发现很多错误，但 repair 只能修 runtime 能证明意图的错误。不能证明的，就应该 retry、fallback、drop 或显式失败。

### 6.3 Evaluation and Telemetry

生产化 Agent 不能只看单次 demo。搭积木搭得好不好，最后必须落到指标上。

它需要持续记录：

- one-shot success；
- valid plan rate；
- step failure；
- token cost；
- repair path；
- replan rate；
- fallback frequency；
- human intervention rate；
- 用户侧真实完成率。

没有这些指标，系统会停留在“看起来能跑”，但不知道为什么成功，也不知道为什么失败。

## 7. 最大难点：分配模型和程序的边界

Planned Workflow 的难点不是让模型输出 JSON，而是持续定义模型和程序各自负责什么：

- 哪些自由度允许交给模型；
- 哪些自由度必须被 runtime 收回；
- 哪些错误可以自动修；
- 哪些错误必须重试或交给人；
- 哪些场景应该走通用 loop，哪些场景应该走标准 workflow。

举一个脱敏例子。

用户说：“画面可以更鲜亮一点，色彩艳丽。”

Planner 可能会判断这是一个视觉风格相关需求，于是微调上一轮的风格描述，再交给下游渲染节点。这个行为看起来很聪明，也确实符合用户字面要求。

但在某些业务系统里，这可能是 bug。因为“风格选择”是上游节点的稳定输出，下游节点只能消费它，planner 不能擅自改写它。否则系统的可追溯性、缓存、复用、版本链都会被破坏。

这时就需要 validator 限制 planner 的灵活度：plan 可以选择哪些节点、如何绑定输入，但不能改写某些已经确定的业务 artifact。

这就是专用 Agent 和通用 Agent 的核心差别。

通用 Agent 更像“给聪明人一堆工具，让他自己解决问题”。专用 Agent 更像“让聪明人参与流程设计和局部判断，但关键协议、状态、验证和副作用由系统接管”。

## 8. 结论

计算机长期给人的默认印象是：程序虽然可能写错，但执行本身是确定、可重复的。

LLM 进入程序主循环之后，这个前提被打破了一部分。程序开始带上人的特征：会理解、会迁移、会补全，也会误解、遗忘、过度泛化、看起来合理但实际跑偏。

所以很多 Agent 工程问题，本质上像是在把过去用来管理人的方法搬进程序系统。人会通过计划、检查、审批、复盘和 SOP 来降低协作中的不确定性；Agent 系统也需要类似机制，只是这些机制最终要落到 runtime 里。

在对质量下限有要求的场合，这些方法不能只停留在管理语言里，而要变成可以被程序执行和观测的基础设施。

最后，我对 Agent 架构演进的判断是：

> 通用 Agent 的形状已经基本固定：保持最小 loop，把复杂度放进 tool surface 和 harness。专用 Agent 则进入组装和评估阶段：积木已经齐全，竞争的是业务组合能力、模型契合度和效果评估能力。

真正高级的 Agent，不是更敢做事，而是更知道什么时候继续、什么时候停、什么时候重试、什么时候拒绝、什么时候让程序或人接管。

## 参考资料

- [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) (accessed: 2026-06-09)
- [OpenAI: A practical guide to building agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/) (accessed: 2026-06-09)
- [OpenAI: Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/) (accessed: 2026-06-09)
- [ysymyth/ReAct](https://github.com/ysymyth/ReAct) (accessed: 2026-06-09)
- [Claude Code Docs: Dynamic workflows](https://code.claude.com/docs/en/workflows) (accessed: 2026-06-09)
- [Claude: Introducing dynamic workflows in Claude Code](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code) (accessed: 2026-06-09)
- [Anthropic: Anthropic acquires Bun as Claude Code reaches $1B milestone](https://www.anthropic.com/news/anthropic-acquires-bun-as-claude-code-reaches-usd1b-milestone?s=33) (accessed: 2026-06-09)
- [从0开发大模型的17种Agent架构演进详细拆解](https://mp.weixin.qq.com/s/5f0I2apY4oFsHrttANBOJg) (accessed: 2026-06-09)
- [FareedKhan-dev/all-agentic-architectures](https://github.com/FareedKhan-dev/all-agentic-architectures) (accessed: 2026-06-09)
