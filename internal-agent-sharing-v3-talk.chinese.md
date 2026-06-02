# Agent 分享

Last Updated: 2026-06-02

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

早期 ReAct 主要靠模型输出文本 action，再由宿主程序解析和执行。现在的模型已经把 tool call 变成一等公民，action 不再只是 prompt 里的文本格式，而是 runtime 可以识别、约束、记录和回传 observation 的结构化接口。

这里的 tool 也不只是普通 API。文件读写、shell、计划管理、skill、subagent、长程任务，都可以被包装成模型可调用、runtime 可约束、结果可回到上下文的行动接口。

以 Claude Code 为例，它的 tool surface 大致可以分成几类：

- workspace tools：`Read` / `Edit` / `Write`，让模型观察和修改代码文件；
- execution tools：`Bash`，让模型运行测试、构建、搜索和脚本；
- coordination tools：`TodoWrite` / `TaskCreate` / `TaskUpdate`，让模型维护任务进度；
- delegation tools：`Agent`，让模型把局部任务交给 subagent；
- long-running tools：`CronCreate` / `CronList`，把未来执行也变成 tool surface 的一部分。

这说明一件事：对 Coding Agent 来说，`Bash` 几乎成了开放式软件任务的标准行动接口。软件任务的不确定性很高，模型需要不断读文件、跑命令、看结果、再决定下一步。测试、build、diff、日志和浏览器反馈，都可以通过 tool observation 回到循环里。

OpenClaw 代表另一种方向。它不是把 Agent 主要放进代码仓库，而是放进长期运行的个人助理环境。它的 tool surface 更关心 memory、session、cron、heartbeat、message、browser、owner permission 和后台生命周期。也就是说，Personal Agent 的难点不是单次任务计划，而是身份、记忆、渠道、异步唤醒和副作用边界。

LangChain 则是框架侧的答案。`create_agent` 提供通用 model-tool loop，middleware 再把 todo、memory、tool retry、human-in-the-loop 等能力插进去。比如 `TodoListMiddleware` 本质上就是把任务进度也包装成模型可见、可更新的 tool state。

这里还要单独注意一类 tool：异步 tool。普通 tool 更像函数调用：模型发起调用，runtime 执行，结果立刻作为 observation 回到上下文。但 cron、heartbeat、subagent、background task 这类能力更像异步 tool：一次 tool call 会启动或登记一个长生命周期执行，真实工作在另一个生命周期里并发进行。

它不一定都返回同一种 handle。有些返回 `run_id`、`session_id`、`response_id`、`schedule_id`，有些只是由 runtime 登记一个后台任务或未来触发器。完成后，结果再通过 poll / retrieve、runtime notification、wake event、session update 或新的 user message 回到 Agent 上下文。

所以 tool 不只是“调 API”。在 Agent runtime 里，tool 更像模型能触达的行动空间。不同 Agent 的产品形态，很大程度上取决于：

- 模型能看见什么能力；
- 哪些状态允许被模型修改；
- 哪些动作必须经过 runtime gate；
- 哪些结果会作为 observation 回到上下文；
- 哪些能力只是内部 runtime 逻辑，不暴露给模型自由调用。

这也为下一章做铺垫：当 tool surface 足够丰富之后，问题就从“模型能做什么”进入到“系统如何组织这些 action”。这就是更复杂的 Agent 设计问题。

## 3. 更复杂的 Agent 设计：目前做到哪了

当 tool surface 变大之后，Agent 要回答的不只是“能调用什么工具”，而是：复杂过程中怎么不忘事、不跑偏、不重复劳动，什么时候继续，什么时候停止，失败后要不要换路径。

这里可以借用一篇文章的总结：[《从0开发大模型的17种Agent架构演进详细拆解》](https://mp.weixin.qq.com/s/5f0I2apY4oFsHrttANBOJg) (accessed: 2026-05-19)。这篇文章有价值的地方，不是把 Agent 拆成 17 个互斥架构，而是把很多系统能力放到一张表里：Reflection、Tool Use、ReAct、Planning、PEV、Multi-Agent、Memory、ToT、Dry-Run、Metacognitive、Self-Improve 等。

如果基于那张表重新归纳，现在更复杂的 Agent 设计其实主要在做几件事：

- **质量闭环**：从一次生成，变成生成、评估、修订、再评估，解决输出质量不稳定和错误静默传播。
- **行动接口**：把搜索、文件、shell、业务 API、subagent、长程任务都变成 tool，解决模型只能“说”不能“做”的问题。
- **显式流程**：把隐式推理变成 todo、plan、workflow、router、loop，解决复杂任务里的遗忘、跑偏和不可审计。
- **工作分配**：用 subagent、role decomposition、ensemble 或 controller 分配任务，解决单个 prompt 承担过多角色的问题。
- **长期状态**：用 memory、session、blackboard、graph state 保存和召回历史，解决跨轮失忆和中间状态丢失。
- **验证和闸门**：用 verifier、validator、dry-run、approval、policy gate 控制继续、重试、拒绝和真实副作用。
- **搜索和模拟**：用 ToT、mental loop、dynamic workflow、parallel fan-out 在行动前探索候选路径，解决局部贪心和真实试错成本高的问题。

所以更准确的说法不是“现在有很多 Agent 架构”，而是：大家都在给最小 tool loop 增加 state、control flow、evaluator、memory、work allocation 和 side-effect gate。

落到主流产品和框架上，可以先看三层。

### 3.1 Context Plan：让模型别忘事

第一层是把计划放进上下文。

Claude Code 的 todo / plan mode、LangChain 的 `TodoListMiddleware`、Superpower 这类插件里的 `brainstorm -> write plan -> execute plan`，都可以放到这一类理解。

它们目前做到的是：让模型和用户共享一份任务进度，把目标、步骤、约束和当前状态留在 attention 里。

它主要解决的问题是开放任务里的遗忘、重复和协作不可见。缺点也明显：runtime 通常不直接执行这个 plan，某个步骤是否完成，很多时候还是由模型更新 todo 或继续行动来声明。

这适合 Coding Agent 这类开放任务，因为软件任务本身有测试、build、diff、日志等外部反馈，模型可以边观察边调整。

### 3.2 Workflow：把稳定流程交给程序

第二层是把流程写成 workflow。

LangGraph、Google ADK Workflow Agents、很多企业低代码 Agent 平台都在这个方向：开发者显式写节点、边、条件、循环和人工确认点，runtime 负责按图执行。

它目前做到的是：把稳定流程从模型注意力里拿出来，交给程序化 runtime。

它主要解决的问题是可控、可审计和可恢复。开发者知道有哪些步骤、每一步的输入输出是什么、哪些动作必须审批、失败应该走哪条分支。

代价是动态性下降。能走哪些路径，通常需要开发者提前设计。对高度开放的任务，如果一开始就强行写死 workflow，系统会变重，也会牺牲灵活性。

### 3.3 Dynamic Workflows：运行时生成执行结构

第三层是让执行结构在运行时生成。

这里的共同点是：workflow 不完全由开发者预先写死，而是由模型、planner 或其他程序在运行时生成、选择或修复，再交给 runtime 执行。

最近 Claude Code 的 Dynamic Workflows 是一个典型例子。Claude 生成 JavaScript orchestration script，script 负责循环、分支、fan-out / fan-in、并行调 subagents 和保留中间结果；每个 `agent()` 调用仍然启动普通 subagent，真正的任务执行仍然是 model-driven。

它目前做到的是：把复杂任务的“组织方式”也交给模型生成，但执行过程不完全塞回主对话上下文，而是由 runtime 在后台执行、记录和恢复。

它主要解决的问题是开放任务中的规模化编排：多分支探索、并行验证、长任务后台运行、中间结果隔离和 token 控制。

这一层也接近 Dayfold 的问题。但 Dayfold 的选择更窄：不是让模型生成任意脚本，而是让 planner 生成 Planned DAG，再由 deterministic runtime 验证、调度和执行。

### 3.4 小结：通用和专用的分歧

软件工程里也有类似经验。面对复杂需求，人类通常不会直接开做，而是先拆解计划；当同类需求反复出现时，又会把计划进一步沉淀成流程、模板或研发规范，用来提高需求交付的质量下限。

这里强调的是“下限”。如果执行者足够聪明，或者通用 Agent 足够强，它当然可以在更少约束下完成任务。但专用业务场景通常不能只赌上限，而是要保证大多数请求都能稳定落在可接受质量以上。

所以分歧不是“有没有 plan”，而是 plan 到底给谁用：

- Context Plan 主要给模型和用户看；
- Workflow 主要给程序 runtime 执行；
- Dynamic Workflows 尝试让模型生成执行结构，再交给 runtime 执行。

这就引出 Dayfold 的设计选择：在相对稳定的业务域里，把 plan 标准化成 runtime 可以验证、调度、审计和恢复的 Planned DAG。

## 4. Dayfold：Planned DAG Workflow

Dayfold 的目标不是做通用 coding agent，也不是做长期个人助理，而是做业务任务自动化。

我们的约束是：

- token 成本要低；
- 一次成功率要高；
- 多步骤执行要稳定；
- 中间状态要可审计；
- 失败要能定位和恢复；
- 业务动作边界要清楚。

如果完全用 model-driven tool loop，每一步都让模型重新判断下一步，会更灵活，但成本和不确定性都更高。

所以 Dayfold 更接近一种 Plan-and-Execute：

```text
planner 生成 Planned DAG
-> runtime 验证并执行
-> capabilities / internal subagents 完成具体步骤
```

顶层不是一个自由行动的 LLM manager，而是程序化 workflow runtime。LLM 参与其中，但主要产出结构化 artifacts：

- planner 产出 workflow DAG；
- capability / internal subagent 产出 typed output；
- validator / reviewer 产出 structured verdict。

Planner 不是完全无状态 prompt。它会根据当前 request、历史状态、已完成步骤、planner policy、可见能力和 validation feedback 生成或修复 workflow DAG。

Runtime 不让模型自由决定每一步，而是验证 DAG、调度 ready step、执行 capability 或 internal subagent，并写入 port values 和 step state。

业务阶段会影响 planner policy、能力集合和合法 DAG shape。后续如果业务模式更多，这部分会更像一个 planner state machine，但分享里不需要展开具体状态。

Dayfold 选择 Planned DAG Workflow，主要收益是：

- 成本更可控：不需要每一步都让模型重新决定下一步；
- 执行更稳定：runtime 负责验证、调度和状态记录；
- 问题更可定位：错误可以落到 DAG、binding、capability output 或 validator。

代价也明确：

- Plan schema、binding 和 capability contract 会变成核心系统协议；
- validator、repair、fallback 会变成必须维护的基础设施；
- runtime state 和 planner policy 的边界需要持续治理。

最终想表达的不是“Planned DAG Workflow 一定比通用 Agent 好”。

更准确的说法是：

> 通用 Agent 倾向把 plan 留在上下文、todo working memory 或 manager ledger 中，以换取开放任务的灵活性。业务型 Agent 如果更看重低 token、稳定性、审计和恢复，可以让 planner 动态生成 workflow DAG，并交给 deterministic runtime 执行。

## 5. Plan 生产化：验证、修复、评估

前面讲的是 Plan 的形态。真正落到生产系统里，问题会再往下走一层：Plan 不能只是“模型生成出来的一段结构”，它必须能被验证、修复、观测和评估。

这部分可以作为 Dayfold 的核心工程经验来讲：业务型 Agent 的可靠性，往往不来自更复杂的 agent 名词，而来自 plan 周围的一组程序化设施。

第一类是 **plan-level validation**。Planner 产出的 workflow DAG 要先被检查：capability 是否存在，binding 是否合法，依赖是否可达，DAG shape 是否符合当前 planner policy。它保护的是“这一轮 workflow 能不能被 runtime 执行”。

第二类是 **step-level validation and repair**。Capability 或 internal subagent 即使输出了合法 JSON，也可能在业务语义上错。这里需要 subagent-local validator、deterministic repair、retry / fallback。它保护的是“单个节点的模型输出能不能被业务接受”。

第三类是 **evaluation and telemetry**。生产化 Agent 不能只看单次 demo，而要持续记录 one-shot success、valid DAG rate、step failure、token cost、repair path、replan rate、fallback frequency。否则系统会变成“看起来能跑，但不知道为什么成功，也不知道为什么失败”。

一个具体例子是 page index drift。

业务里的 page index 本来就是 zero-based。本轮只要求处理：

```text
requested = [1, 2, 3, 4]
```

也就是说，page `0` 不应该被改动。但模型返回时倾向于把这个 batch 局部编号成：

```text
returned = [0, 1, 2, 3]
```

这不是类型错误。`page_index` 仍然是整数，结构也合法。错误在业务语义：模型把绝对下标改成了 batch-local 下标。

这类错误说明：structured output 只解决“可解析”，不保证“业务正确”。关键原则是：

> Repair must be narrower than validation.

也就是说，validator 可以发现很多错误，但 repair 只能修 runtime 能证明意图的错误。不能证明的，就应该 retry、fallback、drop 或显式失败。

这也自然引出后续优化重点：不是继续堆更多 Agent 模式，而是围绕 production quality 做 semantic validation、evaluation harness、state compaction，以及高频流程的 template fast path。

## 6. 结尾

从冯诺依曼式的存储程序计算机到现代软件工程，计算机长期给人的默认印象是：程序虽然可能写错，但执行本身是确定、可重复的。Agent 把这个印象打破了一部分。

Agent 把这个前提改变了。LLM 进入程序主循环之后，程序开始带上人的特征：会理解、会迁移、会补全，也会误解、遗忘、过度泛化、看起来合理但实际跑偏。

所以很多 Agent 工程问题，本质上像是在把过去用来管理人的方法搬进程序系统：先写计划，执行中检查，关键动作审批，失败后复盘，经验沉淀成 SOP，高风险环节加 reviewer。

在对「下限」有要求的场合，这些方法现在不能只停留在管理语言里，而要变成 runtime 里的 schema、validator、repair、fallback、telemetry 和 evaluation harness。

因此 Comico 的关键不是相信模型，而是承认模型带来的不确定性，并把模型擅长的“理解和组合”与程序擅长的“验证、调度和执行”拆开。

## 7. 延伸阅读

### Agent 基础和设计原则

- [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) (accessed: 2026-06-02) — workflow vs agent、何时增加复杂度、常见 workflow pattern 和 agent-computer interface。
- [OpenAI: A practical guide to building agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/) (accessed: 2026-06-02) — instructions、tools、guardrails、handoffs、orchestration 的产品化视角。
- [OpenAI: Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/) (accessed: 2026-06-02) — Codex 作为 model-tool loop harness 的拆解。
- [ysymyth/ReAct](https://github.com/ysymyth/ReAct) (accessed: 2026-06-02) — ReAct 早期 notebook / prompting 实现。

### Harness 和 Context Engineering

- [Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) (accessed: 2026-06-02) — context 管理、context rot、tool result clearing、subagent 返回压缩等实践。
- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) (accessed: 2026-06-02) — 长程 Agent 的 harness、进度管理、可控性和评估。
- [Claude Code Docs: How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works) (accessed: 2026-06-02) — Claude Code 的 agentic loop、tools、context、sessions、permissions。
- [Claude Code Docs: Tools reference](https://code.claude.com/docs/en/tools-reference) (accessed: 2026-06-02) — Claude Code model-visible tool surface 的官方工具名和行为。

### Dynamic Workflow 和 Framework

- [Claude Code Docs: Dynamic workflows](https://code.claude.com/docs/en/workflows) (accessed: 2026-06-02) — Claude Code workflow runtime、subagent orchestration、后台执行和恢复。
- [Claude: Introducing dynamic workflows in Claude Code](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code) (accessed: 2026-06-02) — dynamic workflows 的产品发布和设计动机。
- [LangChain: Deep Agents overview](https://docs.langchain.com/oss/python/deepagents/overview) (accessed: 2026-06-02) — LangChain 对长任务 agent harness 的框架化封装。
- [LangChain: Deep Agents blog](https://www.langchain.com/blog/deep-agents) (accessed: 2026-06-02) — todo、filesystem、subagents、memory、HITL 等能力如何组合成 deep agent。

### 复杂度和失败模式

- [从0开发大模型的17种Agent架构演进详细拆解](https://mp.weixin.qq.com/s/5f0I2apY4oFsHrttANBOJg) (accessed: 2026-05-19) — 17 种 Agent 模式的教学型拆解，适合作为问题清单参考。
- [FareedKhan-dev/all-agentic-architectures](https://github.com/FareedKhan-dev/all-agentic-architectures) (accessed: 2026-06-02) — 17 种模式对应的代码项目来源。
- [Cognition: Don’t Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) (accessed: 2026-06-02) — 多 agent handoff、context loss 和复杂度税的工程复盘。
- [Why Do Multi-Agent LLM Systems Fail?](https://arxiv.org/abs/2503.13657) (accessed: 2026-06-02) — MAST failure taxonomy，对 multi-agent 失败模式的实证分类。
