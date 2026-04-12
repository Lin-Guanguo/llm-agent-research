# 当你开发 Agent 时，你在做什么？

> 从工程视角看三种 Agent 开发模式

Last Updated: 2026-04-12

---

## 一、导言

Anthropic 在 2024 年底的 *Building Effective Agents* 里给了一个被广泛引用的区分：**Workflow 是代码路径预定义的，LLM 只在某些节点决策；Agent 是 LLM 动态决定自己的流程和工具调用**。但到了 2026 年，这条线在实际产品里正在变得模糊——大多数代码框架的默认架构是「workflow 外层 + ReAct agent 内层」，编排平台里的 LLM 节点也在做 agent 式的工具调用。**Workflow 和 Agent 在 2026 年不再是非此即彼，而是逐渐趋同**。

这篇博客从**工程视角**讲三种 Agent 开发各自在做什么。我读了十几个 Agent 产品和框架的源码（包括 Claude Code、Codex、Gemini CLI、Cursor 等 Coding Agent 产品，以及 LangChain、LangGraph、Mastra、Google ADK、AgentScope、Pydantic AI、CrewAI、Dify 等框架和平台），下面的分析都基于这些一手调研。

**三种开发模式**：

1. **低代码平台上的 Agent 开发**——非开发者在 UI 里拖节点搭 workflow，门槛最低
2. **通用 Agent 开发（Single-Agent）**——一个 LLM 在 ReAct 循环里自决一切，Claude Code / Manus 就是这种
3. **企业/领域 Agent 开发（Multi-Agent 编排）**——用 workflow 编排多个 agent 步骤，场景特化，可控性优先

这三种模式用的底层技术可能完全一样（都能用 LangGraph），但**架构选择和工程挑战完全不同**。下面一个一个讲。

---

## 二、低代码平台上的 Agent 开发

门槛最低的一种。你不写代码——你在一个网页 UI 里拖节点、连线、配 prompt，搭出一个带 LLM 的业务 bot。

**你在做什么**：在扣子（Coze）、Dify 这类平台上，用可视化编辑器组装一个 workflow。典型的节点包括：开始节点、知识库检索、LLM 调用、条件分支、消息发送、结束节点。LLM 只是 workflow 里的一种节点——和「调一个 API」「查一个数据库」在流程上是平级的。

**工程本质**：这是一种**可视化的 workflow 编排**。对用户来说是"搭 Agent"，对平台来说是执行一张有向图。

### 平台背后的执行引擎

以 Dify 为例（我读了它的源码）：Dify 在 2024 年把执行引擎拆成了一个独立的 Python 包叫 **graphon**。它的执行模型是一个 **queue-based worker pool**——不是 DAG 拓扑排序，不是 Pregel BSP，而是一个 ready queue + 动态线程池 + 事件分发器。节点跑完之后，dispatcher 根据边的条件决定下一个节点入队。外部控制信号（比如用户点「停止」）通过 Redis 的 command channel 送达。

Dify 内置了 **24 种节点类型**：start、end、llm、knowledge_retrieval、if_else、code（沙箱执行）、http_request、tool、agent、loop、iteration、human_input……其中 agent 节点在 Dify 2.x 里是一个**plugin RPC 边界**——ReAct 循环不在 Dify 核心代码里跑，而是委托给一个外部的插件进程。这意味着不同的 agent 策略（ReAct、function calling、自定义）可以作为插件安装，不用改 Dify 核心。

节点之间的状态传递通过一个叫 **VariablePool** 的二级字典：`variable_pool.get(["node_id", "output_key"])`。弱类型，没有 schema 校验——下游节点拿到的是 raw value，类型不匹配只有运行时才能发现。

### 代表

- **扣子 Coze**（字节，2024-02 国内版上线）——国内大众认知度最高，深度绑定微信/QQ/飞书生态
- **Dify**（开源，2023-04 创建，137k+ stars）——开发者和创业者的自部署选择

国内几乎所有大厂都有自己的版本：百度文心智能体平台、腾讯元器、阿里云百炼……产品名经常改，但工程本质都是同一套范式。

### 个人观察：对这一档的长期前景有点怀疑

Agent 编排平台**本质上是低代码平台**——只是这次的"代码"换成了「写一个调 LLM 的 workflow」。低代码平台在传统编程领域已经有几十年历史了（OutSystems、Mendix、钉钉宜搭），它们一直有一个根本矛盾：**简单的需求用低代码足够，复杂的需求总会突破低代码的表达能力**。

而这一档现在面临一个新变量：**AI 写代码的能力本身在变强**。如果有一天业务人员可以直接让 Coding Agent 帮自己写一个 Python 后端，那"让不写代码的人也能搭 agent"这个价值可能就被消解了。

当然这只是一种推演，短期内（至少 2026-2027）这一档依然会蓬勃生长。但十年尺度上，它可能会面临比传统低代码平台更严峻的存在性挑战。

---

## 三、通用 Agent 开发（Single-Agent）

### Coding Agent = 通用 Agent

先纠正一个名字上的误导：**"Coding Agent" 这个叫法其实不准确**。Claude Code、Cursor 最早确实是给开发者写代码用的，但它的底层架构——ReAct 循环 + 工具集 + sandbox——**本身不限于 coding**：

- Claude Code 改代码 → coding 场景
- Manus 在云端 VM 里做 PPT、画图表、写报告 → 通用任务
- Claude Co-worker 帮你做研究、整理数据 → 通用任务

**架构完全一样，只是工具集和 sandbox 范围不同。** 所以 2026 年的事实是：Coding Agent 就是通用 Agent，"Coding"只是这个架构最先跑通的场景。

### 核心架构：ReAct 循环

这类产品的架构极其简单。核心就是一个循环：

```
while True:
    thought = LLM.think(context)      # 思考下一步做什么
    action = LLM.decide_tool(thought) # 决定调哪个工具
    if no action:
        break                          # 没有工具要调 → 任务完成
    result = execute(action)           # 执行工具
    context.append(result)             # 把结果加回上下文
```

这就是 2022 年 10 月 Yao et al. 在 ReAct 论文（arxiv:2210.03629）里提出的 Thought → Action → Observation 循环。**2026 年最成功的 Agent 产品，用的还是这个最原始的循环——没有多 agent，没有 Plan 产物，没有复杂的 workflow 编排。**

工具集也很简单：**文件系统 + shell**。Claude Code 能读取你本地任何文件、执行任何 shell 命令。Manus 的工具集加了一个浏览器。就这些。

### 上下文工程与记忆：真正的工程壁垒

如果 ReAct 循环这么简单，工程难度在哪里？**答案是上下文管理。**

LLM 的上下文窗口是有限的——一个长任务可能跑几十个循环，工具调用结果累积起来很快就会塞爆上下文。所有 Coding Agent 产品的核心工程挑战都是同一个：**如何在几十步之后让 LLM 还能记得自己在干什么。**

我读了六个 Coding Agent 产品的源码，它们的共同架构模式是**分层压缩**——旧的工具结果逐步被摘要化或截断，腾出空间给新的内容。具体的压缩策略各家不同（Claude Code 做了四层渐进压缩，Codex 是双路并行，Gemini CLI 有验证式压缩），但底层思路一致。记忆方面，几乎所有产品都选择了纯文本文件（`CLAUDE.md`、`AGENTS.md`、`GEMINI.md`）而不是数据库或向量检索——因为纯文本对 prompt cache 最友好。

> 这些上下文和记忆的工程细节，我在之前发过的《6 个主流 Agent 的上下文管理与记忆系统深度对比》和《Claude Code 为啥好用》两篇里做了源码级的展开分析，这里不重复了。

**一句话：上下文工程是 Coding Agent 的第一工程挑战，不是 ReAct 循环本身。** 循环谁都能写，但"跑 50 步之后还能保持连贯"——这才是区分产品质量的地方。

### 历史脉络

- **2022-10**：ReAct 论文发布，LangChain 同月诞生（晚 18 天）。但 GPT-3.5-turbo 的 API 到 2023-03 才开放，没人真用
- **2023-03/04**：GPT-4 发布后两周，AutoGPT 和 BabyAGI 同时爆红。有意思的是这两个爆款架构完全不同：AutoGPT 是 ReAct 风格的循环，BabyAGI 走的是 Plan-and-Execute 路线（三个子 agent + 任务队列）。两条路线从一开始就并行存在
- **2024-03**：Cognition 发布 Devin，重新定义了 Coding Agent 的产品形态
- **2024 底 - 2025**：Cursor、Claude Code、Codex CLI 相继成熟。模型够强之后，**架构反而回到了最朴素的循环**——没有多 agent，没有复杂 plan，就是 ReAct + 工具 + 上下文压缩
- **2025-2026**：Manus / Claude Co-worker 把同样的架构从本地 CLI 扩展到云端 sandbox，从 coding 扩展到通用任务

### 工程 insight

> **模型够强之后，架构反而回到最朴素的循环。** Coding Agent 的成功不是因为架构精巧，而是因为模型能力跨过了一个门槛——跨过之后，简单的 ReAct 循环就足以完成复杂任务，更复杂的架构反而成了负担。

---

## 四、企业/领域 Agent 开发（Multi-Agent 编排）

### 和第三类用一样的技术，但做不一样的选择

这一类和上面的通用 Agent 开发**可能用完全一样的框架**（都能用 LangGraph，都能用 Mastra），但架构选择不同。

通用 Agent 选的是 Single-Agent——一个 LLM 在一个大循环里自决一切，不拆。

企业/领域 Agent 选的是 Multi-Agent 编排——需要把任务拆成多个步骤、多个 agent，用 workflow 组织它们的执行顺序，每一步有明确的输入输出。因为企业场景要的是**可审计、可回滚、成本可预估**——而单 agent 的 ReAct 循环在这些维度上天然做不到。

### 核心架构共识："workflow outer + ReAct inner"

2025-2026 年，几乎所有代码框架都收敛到了同一个架构模式：

**外层是一个 workflow（开发者定义），内层每个节点跑一个 ReAct agent。** 不同框架对这个模式的实现不同：

- **Mastra**（TypeScript）：`.then()` / `.parallel()` / `.dowhile()` 组合 workflow，每个 step 内部是 ReAct，默认 `stepCountIs(5)` 停止
- **Google ADK**（Python）：`SequentialAgent` / `LoopAgent` / `ParallelAgent` 三种固定编排原语。底层执行循环是 `while True: _run_one_step_async`，没有默认步数限制
- **LangGraph**（Python）：`StateGraph` + 条件边 + `Send` 动态扇出。底层是 **Pregel BSP 引擎**——这一点和其他框架不同，它**原生支持 cycles**，agent 循环直接表达为图的回边，不需要额外的 while loop 代码
- **Pydantic AI**（Python）：固定的 4 节点图（UserPromptNode → ModelRequestNode → CallToolsNode → loop），每次 run 都是这同一张图。它的独特之处在于用 Python 泛型做类型传递——`Agent[DepsT, OutputT]` 的 `OutputT` 会从定义一路传到最终 result
- **AgentScope**（阿里通义，Python）：`ReActAgent` 为核心 + 可选的 `PlanNotebook` sidecar。PlanNotebook 有真实的 `Plan` / `SubTask` Pydantic 数据模型，但执行依然是 ReAct——plan 的合规靠 prompt hint injection，不是代码强制

### 历史演化：代码框架最激烈的三年

这一档的历史比前两类都精彩。

**2022-10：LangChain v1 诞生。** 和 ReAct 论文同月出生（晚 18 天）。早期就是一个把 ReAct 循环变成 Python API 的库，有 9 种 `AgentType`（`ZeroShotReact`、`OpenAIFunctions`、`StructuredChatReact`……），本质上都是 ReAct 的不同 prompt 包装。

**2023-05：LangChain 官方提出 Plan-and-Execute。** 发了一篇博客推出 `plan_and_execute` 包。博客原话：

> *"User objectives are becoming more complex... The need for increasingly complex abilities and increased reliability are causing issues when working with most language models."*
> —— LangChain, "Plan-and-Execute Agents", 2023-05-10

这是业界第一次官方讨论在 ReAct 之外探索另一种 agent 架构的可能性。

**2023-2024：Multi-Agent 狂潮。** MetaGPT（DeepWisdom 团队，论文 2023-08，作者列表里甚至有 LSTM 发明人 Jürgen Schmidhuber）、AutoGen（Microsoft Research，2023-09 开源）、CrewAI（2023-11 开源，2024-10 商业化）相继涌现。它们的共同点：**把 agent 从一个 LLM 在循环，变成多个 LLM 在互相对话**。"AI team 协作比单 agent 强"成了 2024 年的业界想象。

**2024-01：LangGraph 发布。** LangChain 推出 LangGraph，用 StateGraph 重写 agent。底层 Pregel BSP 引擎原生支持 cycles。早期主打 multi-agent subgraph 组合。

**2025：Cognition + MAST 的质疑。** 这一年发生了两件影响代码框架走向的事。

### Multi-Agent Handoff 的工程陷阱

这个问题值得单独讲,因为**我在源码级调研中发现所有主流框架几乎都踩了同一个坑**。

当 agent A 需要调用 agent B 时，大多数框架的做法是：把 B 包装成一个 tool，A 调用这个 tool，拿到 B 的**最终文本输出**。B 的中间推理、工具调用历史、尝试过又放弃的路径——全部丢失。

- **Mastra**：`AgentTool` 把 sub-agent 包装成 tool，只返回最终文本
- **CrewAI**：`DelegateWorkTool` 返回 `str`，worker 的整个 ReAct trace 对 caller 不可见
- **Pydantic AI**：默认的 agent delegation 模式同样丢失 sub-agent 的 message history

**Google ADK 是我调研过的框架里唯一做对了这件事的**：它的 `transfer_to_agent` 机制让 child agent 共享 parent 的 `InvocationContext`，child 的完整 trace 保留在 session 里。但它的另一个 API `AgentTool` 又回到了"只返回文本"的老路。

这就是 2025 年 6 月 Cognition 的 Walden Yan 在 *Don't Build Multi-Agents* 里说的那件事。他给了两条原则：

> Principle 1: **"Share context, share full agent traces"**
> Principle 2: **"Actions carry implicit decisions, and conflicting decisions carry bad results"**

并用了一个具象的例子来解释——假设你让 multi-agent 系统做一个 Flappy Bird 克隆，planner 拆出两个 subtask：「做一个有绿水管和碰撞盒的滚动背景」和「做一只可以上下移动的鸟」。subtask 已经写得非常具体了——「绿水管」几乎就是 Flappy Bird 的视觉商标。但两个 subagent 各自一跑：一个做出 Super Mario Bros 风格的背景，另一个做出一只不像游戏素材、移动方式也完全不像 Flappy Bird 的鸟。两个 subagent 各自"完成了任务"，最后 combiner agent 只能硬着头皮拼在一起。

同期，Berkeley + Cornell 在 2025 年 3 月发了 MAST 论文（arxiv:2503.13657），用 150 条 trace 构建了 14 种 multi-agent 失败模式的 taxonomy，然后用 1600+ 条 trace 验证。一个关键发现：**14 种失败模式里有 8 种属于「架构级」**——不一定能靠换更聪明的模型解决。

这两份证据——Cognition 的 production 经验 + MAST 的定量数据——让业界开始重新审视"拆 agent 越多越好"这个假设。这场辩论现在还没有完全结束（CrewAI 这类框架依然有用户），但代码框架的设计天平在 2025 下半年明显开始往"单 agent + workflow 外层"一侧倾斜。

**2025-10-22：LangChain 1.0 发布。** 彻底重写为 LangGraph 的 wrapper。Classic 时代的所有老 API（9 种 AgentType、AgentExecutor、plan_and_execute）全部挪到 `langchain-classic` 兼容包。最值得玩味的是：**连 `create_react_agent` 都被 deprecated 了**，新的入口叫 `create_agent`——不带"react"这个词。

2022 年 10 月 LangChain 和 ReAct 论文同月诞生。三年后 LangChain 的终极 API 里不再提 ReAct 的名字——不是因为它被抛弃了，恰恰相反：**它赢到了所有框架都默认是它，名字都不用再挂出来了**。

### 工程 insight

> **§三 的通用 Agent 和 §四 的企业 Agent，本质区别就是 Single vs Multi。** 一个选择不拆（一个大循环搞定），一个选择拆（workflow 编排多个步骤/agent）。2025 年之后代码框架的天平开始从"拆"往"不拆"一侧倾——但这是一个进行中的趋势，不是定论。在企业场景中，可审计、可回滚、成本可预估的需求依然存在，这些需求目前单靠 Single-Agent 的 ReAct 循环还给不了。

---

## 五、总结

### 时间线

| 年份 | 低代码平台 | 通用 Agent (Single) | 企业 Agent (Multi / 代码框架) |
|---|---|---|---|
| 2022-10 | — | ReAct 论文 | LangChain v1 |
| 2023-03/04 | — | AutoGPT + BabyAGI | — |
| 2023-04 | Dify 开源 | — | — |
| 2023-05 | — | — | LangChain Plan-and-Execute |
| 2023-08/09 | — | — | MetaGPT / AutoGen |
| 2023-11 | — | — | CrewAI 开源 |
| 2024-01 | — | — | LangGraph |
| 2024-02 | 扣子上线 | — | — |
| 2024-03 | — | Devin | — |
| 2024 底 | — | Claude Code / Cursor 成熟 | — |
| 2025-03 | — | — | MAST 论文 |
| 2025-06 | — | — | Cognition "Don't Build Multi-Agents" |
| 2025-10 | — | — | LangChain 1.0 (create_react_agent deprecated) |
| 2025-2026 | — | Manus / Claude Co-worker | — |

### 2025 年的转折

2025 年是代码框架演化的一个转折点。Anthropic 在 2024 年底给了 Workflow vs Agent 的通用定义，Cognition 从 production 经验角度质疑了 multi-agent 的可靠性，MAST 论文从学术数据角度提供了量化证据。

这三件事加起来给了业界一个值得认真对待的观点：**multi-agent 系统的不稳定不一定来自模型能力，可能来自架构本身**。但需要说明的是，这个观点目前并没有成为定论——CrewAI 这类框架依然有用户，有它的适用场景。代码框架的趋势在转向，但不是所有人都在同一条船上。

### Agent 和 Workflow 的趋同

一个值得关注的趋势是 Agent 和 Workflow 正在融合。Mastra 外层是 workflow DAG、内层每个节点跑 ReAct。扣子/Dify 是 workflow 编排平台，但 LLM 节点又允许 agent 式的工具调用。Claude Code 看起来是纯 Agent，但你给它一个 todo.md 让它按列表执行，它就在跑一个隐式的 workflow。

**大部分 2026 年的 Agent 产品，都落在"纯 Agent"和"纯 Workflow"的光谱中间**。

---

## 六、结尾

三种 Agent 开发模式，三个 one-liner：

- **低代码平台**：给非开发者的可视化 workflow 编辑器。工程本质是图执行引擎 + LLM 节点
- **通用 Agent**：一个 LLM 在 ReAct 循环里自决一切。工程壁垒在上下文压缩，不在循环本身
- **企业 Agent**：用 workflow 编排多个 agent 步骤。和通用 Agent 用一样的技术，但选择拆——而拆不拆这件事，2025 年以来一直在被辩论

---

**续集预告**

这篇博客讲的是三种 Agent 开发模式各自长什么样。但有一个灰色地带这次没展开——某些生产场景里，通用 Agent 的 ReAct 循环给不了成本预估和审计追溯，企业 Agent 的 multi-agent 编排又有 Cognition 指出的 handoff 问题。

人们在这个灰色地带做了一些介于两者之间的东西——比如 Plan-and-Execute 加 typed dataflow 的组合。它既不是纯 Single-Agent，也不是传统的 Multi-Agent workflow，而是另一种东西。

**下一篇**我聊这个灰色地带，以及为什么我自己做的生产 Agent 系统在这里。
