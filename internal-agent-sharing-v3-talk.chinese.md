# Agent 分享材料 V3：讲稿骨架

Last Updated: 2026-06-01

## 1. Agent 最小模型：边观察边行动

现代 tool-calling Agent 的底层循环其实很小：

```text
model -> tool call -> observation -> model
```

模型看到上下文，决定要不要调用工具。Runtime 执行工具，把结果放回上下文。模型再根据新的观察决定下一步。如果模型不再发起 tool call，系统通常认为这一轮可以结束。

这个思想最早可以追溯到 ReAct：Yao et al. 在 2022/2023 年提出让模型交替生成 reasoning trace 和 action，典型形式是 `Thought -> Action -> Observation -> Thought ...`。早期官方实现也很轻量，主要是 notebook 形式的 prompting experiments：[ysymyth/ReAct](https://github.com/ysymyth/ReAct)。

它当时还没有今天的 function call / tool call 协议，核心循环本质上是让模型输出一段文本，再由宿主程序解析 action 并执行。原始 notebook 的关键代码如下，中间省略部分用 `...` 表示：

```python
def webthink(idx=None, prompt=webthink_prompt, to_print=True):
    question = env.reset(idx=idx)
    ...
    prompt += question + "\n"

    n_calls, n_badcalls = 0, 0
    for i in range(1, 8):
        n_calls += 1

        thought_action = llm(prompt + f"Thought {i}:",
                             stop=[f"\nObservation {i}:"])

        try:
            thought, action = thought_action.strip().split(f"\nAction {i}: ")
        except:
            print('ohh...', thought_action)
            n_badcalls += 1
            n_calls += 1
            thought = thought_action.strip().split('\n')[0]
            action = llm(prompt + f"Thought {i}: {thought}\nAction {i}:",
                         stop=[f"\n"]).strip()

        obs, r, done, info = step(env, action[0].lower() + action[1:])
        obs = obs.replace('\\n', '')
        step_str = f"Thought {i}: {thought}\nAction {i}: {action}\nObservation {i}: {obs}\n"
        prompt += step_str

        ...

        if done:
            break

    if not done:
        obs, r, done, info = step(env, "finish[]")

    info.update({'n_calls': n_calls, 'n_badcalls': n_badcalls, 'traj': prompt})
    return r, info
```

也就是说，最早的 ReAct 更像是“文本 action grammar + 手写 runtime loop”，不是今天这种模型原生返回结构化 tool call 的形式。

这就是很多 Agent 的共同内核。Claude Code、Codex、OpenClaw、LangChain `create_agent`，内层都可以近似看成这种反馈循环。

一句话：

> Agent 的核心循环很简单，复杂的是外层 harness。

## 2. Tool Surface：复杂设计的共同入口

如果 ReAct 给的是最小循环，那么复杂 Agent 的下一步，往往不是让模型“想得更多”，而是设计模型到底能做哪些 action。

这里的 action 不只是普通 API 调用。文件读写、shell、计划管理、skill、subagent、长程任务，都可以被包装成模型可调用、runtime 可约束、结果可回到 observation 的 tool。

真实系统里能看到很直接的例子。Claude Code 里，文件和命令是 `Read` / `Edit` / `Write` / `Bash`，任务管理是 `TodoWrite` / `TaskCreate` / `TaskUpdate`，subagent 是 `Agent`，旧名甚至叫 `Task`，长程任务也被包装成 `CronCreate` / `CronList` 这样的 tool。

Codex 也有类似设计：`update_plan` 不是业务 API，而是给模型维护协作计划的 tool。LangChain 的 `TodoListMiddleware` 通过 `write_todos` 暴露 todo state。OpenClaw 作为 personal agent，则把记忆、会话、长程任务和外部应用都变成 tool，例如 `memory_search`、`memory_get`、`sessions_spawn`、`cron`、`browser`、`message`。

所以 tool 不只是“调 API”。在 Agent runtime 里，tool 更像模型能触达的行动接口：它可以是文件系统、shell、计划状态、技能、子 Agent、异步任务，甚至是未来某个时间点重新唤醒自己的 hook。

这也解释了为什么同样是 model-tool-observation loop，不同产品用起来差异很大。差异不只来自模型，而来自 tool surface：

- 模型能看见什么能力；
- 哪些状态允许被模型修改；
- 哪些动作必须经过 runtime gate；
- 哪些结果会作为 observation 回到上下文；
- 哪些能力只是内部 runtime 逻辑，不暴露给模型自由调用。

一句话：

> Agent 的产品形态，很大程度上是 tool surface 的设计结果。

## 3. 三个案例：同一个循环，不同产品形态

### 3.1 Claude Code / Codex：开放式软件任务

Coding Agent 的任务空间很开放。用户可能让它修 bug、读代码、跑测试、改架构。模型一开始通常不知道真实计划是什么，需要先观察文件系统和运行结果。

所以 Claude Code / Codex 会使用 todo、plan mode、verification agent、测试和 diff，但这些 plan 多数不是 runtime 执行协议，而是模型的工作笔记和用户协作界面。

这很适合软件任务。因为软件任务有天然反馈：测试能不能过、build 有没有报错、diff 是否合理、浏览器里有没有问题。

结论：

> Coding Agent 适合 Attention Plan：计划帮助模型保持注意力，但下一步仍然由模型根据观察决定。

### 3.2 OpenClaw：长期个人助理

OpenClaw 代表的是 personal agent。它的重点不是把任务编译成 workflow，而是把模型放进一个长期运行的个人助理环境。

它关心的是：

- 谁在和 Agent 对话；
- 当前消息属于哪个 session；
- 哪些 memory 相关；
- 哪些工具只有 owner 可以用；
- cron、heartbeat、wake 如何触发；
- 发送消息、shell、长期任务如何审批。

这类系统的核心问题是 continuity 和 access boundary。也就是说，它要维护身份、记忆、渠道、后台生命周期和副作用边界。

结论：

> Personal Agent 的难点不是 plan runtime，而是长期运行时的身份、记忆、权限和生命周期。

### 3.3 LangChain：框架化通用 Agent

LangChain 代表的是框架侧的答案。

LangGraph 是 stateful workflow runtime；LangChain `create_agent` 给你一个预置的 model-tool loop；middleware 再把 memory、tool retry、human-in-the-loop、PII、context editing、todo 等能力插入循环。

最典型的是 `TodoListMiddleware`。它给 Agent 增加 todo state 和 `write_todos` 工具，让模型可以记录任务进度。

但这个 todo 仍然是 soft plan。Runtime 不会读取 todo 然后自动调度业务节点。下一步还是模型根据上下文决定。

结论：

> 通用框架倾向把 planning 做成 middleware 和 model-visible state，因为这样最通用、最灵活、接入成本最低。

## 4. 通用 vs 专用：有没有标准 Plan

到这里可以把问题收束到一个词：Plan。

只要任务超过一次 tool call，Agent 就必须回答几个问题：下一步做什么，当前进度是什么，什么时候算完成，失败后要不要换路径。不同系统都会长出某种“计划”机制：Coding Agent 里有 todo 和 plan mode，LangChain 里有 `TodoListMiddleware`，OpenClaw 里有 session、cron、subagent 和长期任务生命周期。

但这些机制不一定是同一种东西。真正的分歧不是“有没有计划”，而是：

> 有没有一个标准 Plan，作为 runtime 可以执行的协议。

通用 Agent 通常不把 plan 设计成唯一标准的 runtime contract。

原因也很直接。Coding Agent 面向开放式软件任务，OpenClaw 面向长期个人助理，LangChain 面向框架复用。它们都需要覆盖大量未知场景。如果强行要求所有任务都先变成一个标准 workflow，系统会变重，灵活性也会下降。

所以通用 Agent 更常见的做法是：让 plan 留在模型注意力、todo state、manager ledger 或 notebook 里。Plan 帮助模型和用户协作，但 runtime 不一定把它当成调度协议。

专用业务 Agent 的问题不同。它不需要覆盖所有任务，而是在一个相对稳定的业务域里，把用户意图转换成可靠的业务执行流程。这时 plan 就有机会标准化：

```text
plan = runtime 可以验证、调度、审计和恢复的 workflow contract
```

这也是 Dayfold 可以选择 Planned DAG Workflow 的前提。它属于 dynamic workflows 的大类，但比这个上位名更窄：Plan 的形态是 DAG contract，而不是任意脚本或自由文本。

按这个角度，Plan 可以压成三类：

### 4.1 Attention Plan

计划主要给模型和用户看，不是 runtime 的标准调度协议。

Text plan、todo list、manager ledger、plan notebook 都可以放到这一类。它们的作用是帮助模型维持注意力、记录进度、和用户协作。

这类 plan 的优点是灵活，适合开放任务。缺点是 runtime 很难直接调度它，也很难证明它真的完成了。

### 4.2 Workflow

计划由开发者写成 graph、DAG 或代码。这里有标准流程，但动态性主要来自开发者预先设计。

它的优点是稳定、可审计、容易控制。缺点是动态性弱，很多分支需要提前设计。

### 4.3 Dynamic Workflows

这里把 Dynamic Workflows 当成上位类。

共同点是：workflow 不是完全由开发者预先写死，而是在运行时由模型、planner 或其他程序生成、选择或修复，再交给某种 runtime 执行。

可以先拆成两个 subtype：

```text
Dynamic Workflows
-> Scripted Dynamic Workflow
-> Planned DAG Workflow
```

最近 Claude Code 的 dynamic workflows 就可以归到这一类。它更像 Scripted Dynamic Workflow：Claude 生成 JavaScript orchestration script，script 负责循环、分支、并行调 subagents 和保留中间结果，runtime 在后台执行。参考 [Claude Code Dynamic workflows](https://code.claude.com/docs/en/workflows) (accessed: 2026-06-01) 和 [Introducing dynamic workflows in Claude Code](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code) (accessed: 2026-06-01)。

Dayfold 更接近这个上位类下的 Planned DAG Workflow：planner 在运行时生成或修复 typed DAG；runtime 仍然负责验证、调度、执行和记录状态。

它试图同时拿到两件事：

- planner 带来的动态性；
- workflow runtime 带来的稳定性。

代价是系统复杂度上升：DAG schema、binding、validator、state management 都要设计。

一句话：

> Attention Plan 是给模型维持注意力；Workflow 是开发者写死流程；Dynamic Workflows 是运行时生成执行结构；Planned DAG Workflow 是 planner 生成 typed DAG，runtime 验证和执行。

## 5. Dayfold：为什么选择 Planned DAG Workflow

Dayfold 的目标不是做通用 coding agent，也不是做长期个人助理，而是做业务任务自动化。

我们的约束是：

- token 成本要低；
- 一次成功率要高；
- 多步骤执行要稳定；
- 中间状态要可审计；
- 失败要能定位和恢复；
- 业务动作边界要清楚。

如果完全用 model-driven tool loop，每一步都让模型重新判断下一步，会更灵活，但成本和不确定性都更高。

所以 Dayfold 更像：

```text
planned DAG workflow + internal subagents / capabilities
```

顶层不是一个自由行动的 LLM manager。顶层是程序化 workflow runtime。LLM 参与其中，但主要产出结构化 artifacts：

- planner 产出 workflow DAG；
- capability / internal subagent 产出 typed output；
- validator / reviewer 产出 structured verdict。

## 6. Dayfold 的结构和取舍

当前 Dayfold 可以按三层理解：

```text
program-driven agent state / registration
-> stateful planner
-> one-turn deterministic execution workflow
```

第一层是 Agent registration / configuration。它决定当前运行形态：使用哪套 planner policy、暴露哪些 capability、允许哪些 DAG shape。

第二层是 stateful planner。Planner 不是完全无状态 prompt。它会根据当前 request、历史状态、已完成步骤、planner policy、可见能力和 validation feedback 生成或修复 workflow DAG。

第三层是单轮 deterministic workflow。Validator 检查 DAG，coordinator 调度 ready step，capability 或 internal subagent 执行本地任务，runtime 写入 port values 和 step state。

当前 planner state 可以先按三个业务状态理解：

- `onboarding`：从 onboarding 入口进入后持续保持，不切换到其他 state；
- `chat`：从 chat 入口进入，负责收集和确认需求；
- `story`：chat 到一定阶段后切换进入，进入后不再切回 chat。

未来如果引入 OC 设计、画风设计等更多业务模式，状态会更多，切换逻辑也会更复杂。但如果架构不大改，本质仍然是同一类 planner state machine：程序状态决定 planner policy、能力集合和合法 DAG shape，planner 在当前状态下生成 workflow DAG。

Dayfold 选择 Planned DAG Workflow，主要收益是：

- token 成本更可控；
- 执行过程更稳定；
- workflow DAG、step state、port values 可审计；
- 错误可以定位到 DAG、binding、capability 或 subagent output；
- capability contracts 可以长期复用；
- ready steps 可以并行执行。

代价也明确：

- DAG schema 成为系统协议；
- binding 规则会变成 DSL；
- planner validator 和 subagent-local validator 都会越来越重要；
- planner prompt 必须理解 capability contracts；
- runtime state 需要压缩、恢复和观测；
- 类型正确不等于语义正确；
- 程序状态机、planner policy、runtime state 的边界需要持续维护。

最终想表达的不是“Planned DAG Workflow 一定比通用 Agent 好”。

更准确的说法是：

> 通用 Agent 倾向把 plan 留在模型注意力、todo working memory 或 manager ledger 中，以换取开放任务的灵活性。业务型 Agent 如果更看重低 token、稳定性、审计和恢复，可以让 planner 动态生成 workflow DAG，并交给 deterministic runtime 执行。

## 7. 让 Agent 可用的程序化设施

Dayfold 的另一个经验是：只靠模型生成结构化输出，不等于系统已经可靠。

真正能把 Agent 做成可用系统的，往往是一组程序化设施：

- **schema validation**：先保证模型输出能被解析；
- **business validator**：检查结构合法但业务语义错误的情况；
- **deterministic repair**：只修 runtime 能证明意图的错误；
- **fallback / retry**：不能证明的错误，不强修；
- **telemetry**：记录失败类型、修复路径、token 成本和重试次数。

Validator 也不能只讲一层。至少有两类。

第一类是 **planner validator**。它验证 planner 产出的 workflow DAG 是否能被 runtime 执行：capability 是否存在、binding 是否合法、依赖是否可达、DAG shape 是否符合当前 planner state。

第二类是 **subagent-local validator**。它验证某个 capability 或 internal subagent 的结构化输出是否满足业务语义。

一个具体案例是 page index drift。

业务里的 page index 本来就是 zero-based。本轮只要求处理：

```text
requested = [1, 2, 3, 4]
```

也就是说，page `0` 不应该被改动。但模型返回时倾向于把这个 batch 局部编号成：

```text
returned = [0, 1, 2, 3]
```

这不是类型错误。`page_index` 仍然是整数，结构也合法。错误在业务语义：模型把绝对下标改成了 batch-local 下标。

所以 structured output 只解决“可解析”，不保证“业务正确”。关键原则是：

> Repair must be narrower than validation.

也就是说，validator 可以发现很多错误，但 repair 只能修 runtime 能证明意图的错误。不能证明的，就应该 retry、fallback、drop 或显式失败。

## 8. 结尾：下一步应该优化什么

下一步不应该继续堆更多 Agent 模式，而应该围绕 production quality 做评估。

优先方向：

- **Evaluation harness**：衡量 one-shot success、valid DAG rate、step failure、token cost、replan rate、fallback frequency。
- **Semantic validation**：解决“结构合法但语义错误”的 DAG 或 subagent output。
- **State compaction**：明确 port values、context messages、artifacts 什么时候从 active state 变成 archival state。
- **Template fast path**：对高频稳定流程使用 prevalidated workflow templates，减少 planner 推理成本。
- **Planner state machine evolution**：如果未来业务状态更多，持续评估状态切换逻辑放在程序层、planner 层还是 runtime 层。

最后一句：

> Dayfold 的关键不是让模型每一步都更聪明，而是把模型擅长的“理解和组合”与程序擅长的“验证、调度和执行”拆开。
