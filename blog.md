# 当你开发 Agent 时，你到底在开发什么？

Last Updated: 2026-04-12
Status: Outline v4.3 (pending verification of workflow platforms)
Target length: ~12,000 字符（对齐 `llm-memory-research/blog.2.chinese.md` 和 `blog.3.chinese.md`）
Account: 小红书 "AI 观果"

---

## Positioning

这篇博客是一份 2026 年 Agent 开发现状的**盘点**。

会做三件事：
1. **分类** —— 把所有叫 "Agent 开发" 的东西分成三类：Coding Agent / Agent 编排平台 / Agent 代码框架
2. **逐类盘点** —— 每一类按历史顺序介绍主流代表
3. **总结 + 观察** —— 在最后给出时间线、定调，以及一个关于"Multi vs Single Agent"的个人观察（作为可选视角，不作为判据）

**不是**要立什么鲜明观点，**也不要把**"Multi vs Single" 当成分类的第一性判据——那只是我写完之后回头看这段历史的一个粗略视角，留在 §六 总结里提一下。

---

## Outline v4.3

### 一、开头钩子 (~500 字)

**场景**：咖啡店三个朋友

- A 在扣子拖节点做客服 bot
- B 在 Claude Code 改代码
- C 在 CrewAI 搭 AI team

**Hook**：三个人都说"在做 Agent"——但它们不是一回事。它们分属三种完全不同的产品形态，服务不同的用户，解决不同的问题。

**Promise**：这篇博客给你一个分类 + 每类的主流代表 + 一段 2022-2026 的演化时间线，看完你就知道"做 Agent"到底在说什么。

---

### 二、为什么 Agent 这个词在 2026 这么混乱 (~600 字)

**简短版**，不磨叽：

- Agent 在 AI 教材里本来很宽泛
- LLM 圈改造了它，但含义被拉扯成 4-5 种不同的东西
- 三个原因：名词跑得快 / "Agent" 更潮 / 不同社区定义不同

**本文的做法**：不尝试给 Agent 下一个"标准定义"，而是**按产品形态划分三类**，逐一盘点。划分的问题很简单——

> **你是在用它，还是在搭它？** 如果在用，它本身就是一个终端产品（Coding Agent）。如果在搭，它是平台——看是谁来搭：非开发者在 UI 里拖（编排平台），还是开发者在代码里写（代码框架）。

三类覆盖完市面 95% 的"Agent 开发"语义。

---

### 三、第一类：Coding Agent —— 你直接使用的 Agent (~2000 字)

**定义**：终端用户直接使用的 Agent 产品。你给它一个模糊目标，它自己决定怎么做。
**和其他两类的差别**：你不用它"搭"什么，你直接用它。

#### 按历史顺序列主流代表

**① ReAct 论文 + LangChain v1 (2022-10)** (~250 字)

- 2022-10-06 Yao et al. 发表 *ReAct: Synergizing Reasoning and Acting*
- 2022-10-24 LangChain v1 发布，晚 18 天——学术到产品的转化速度空前
- **但 2022 年底实际没人真正用**——GPT-3.5-turbo API 2023-03 才开放，舞台还没搭

**② 2023-04 的双线爆发：AutoGPT vs BabyAGI** (~500 字) ⭐ 反直觉亮点

- **AutoGPT (2023-03-30)** —— **ReAct 风格**的产品化，用 thoughts JSON 结构化输出（text / reasoning / plan / criticism / speak + command）。严格说它不是经典的 Thought/Action/Observation 格式，但行为是 ReAct 的
- **BabyAGI (2023-04-01)** —— Plan-and-Execute 架构 + 任务队列，三个子 agent（execution / task_creation / prioritization）+ Pinecone 记忆
- 两个项目前后 3 天内发布
- **这是个反直觉的事实**：大部分人记得 AutoGPT，但忘了 BabyAGI。更少人知道这两个爆款的架构完全不同——一个是 ReAct 风格的循环，一个是先拆任务再执行的 Plan-and-Execute
- **2023 年早期的 Agent 狂潮不是一种东西的爆发，是两种范式的同时登场**
- 两者 reliability 都很差，狂热期持续到 2023 年中就冷却了
- （插曲：BabyAGI 原 repo 2024-09 被归档为 functionz，历史代码保留在 `yoheinakajima/babyagi_archive`）

**③ Devin 发布 (2024-03)** (~250 字)

- Cognition AI 发布 Devin，宣称"第一个 AI software engineer"
- 引发巨大争议（Internet of Bugs 逐帧拆解）
- 但它重新定义了 Coding Agent 的产品形态
- **伏笔**：两年后（2025-06）同一家公司会发文自我否定——"Don't Build Multi-Agents"

**④ Cursor / Claude Code / Codex (2024-2025)** (~500 字)

- 2024 年模型能力显著提升，Coding Agent 从"demo 级"走向"生产力工具"
- Cursor 把 Agent 能力做进 IDE
- 2024 底 Anthropic 发布 Claude Code CLI
- OpenAI 发布 Codex CLI
- **这是 Coding Agent 从"狂热 demo"走向"真正被开发者每天用"的分水岭**
- 架构上是"文件系统 + shell tools + LLM 大循环"的统一模式

**⑤ Manus / Claude Co-worker / GenSpark (2025-2026)** (~500 字)

- 从本地 CLI 扩展到云端 sandbox
- Manus 是国内最出圈的云端任务 Agent
- 架构和 Coding Agent 完全一致——只是部署位置从本地变成云端 sandbox
- 扩展了 Coding Agent 的应用范围：从"帮程序员写代码"到"帮任何人完成长任务"

---

### 四、第二类：Agent 编排平台 —— 非开发者搭 AI 应用 (~1800 字)

**定义**：非工程师在 UI 里画流程，搭出一个带 LLM 的业务 bot 或应用
**和其他两类的差别**：不是终端产品（你不直接用它），也不是代码库（你不写代码）——它是一个**可视化的低代码搭建器**
**为什么存在**：企业场景需要的是**确定性**——可审计、可回滚、成本可预估。完全让 LLM 自决在这些场景里不够可控，于是业界发展出了"人画流程、LLM 做节点"的产品形态

#### 扣子官方自己怎么看

**引用原话**（可加粗）：**"规划是最难以工程化的，用工作流编排方式，但这与大模型的开放性产生本质冲突"**

这是国内最大的 Agent 编排平台的产品团队公开承认——**他们主动绕开了自主规划这个核心问题**。

#### 按历史顺序列主流代表

> ✅ 本节所有产品名和日期已通过公开资料求证，详见 `platform-verification.md`

**① Dify (2023-04-12 GitHub 创建)**

- 开源标杆，**137k+ stars**（GitHub API 实测 2026-04）
- 执行引擎 2024 年后拆成独立包 `graphon`——queue 驱动的 worker pool
- 给开发者和 AI 创业者用，支持自部署

**② 扣子 Coze (2024-02-01 国内版 coze.cn 上线)**

- 字节跳动，**国内大众认知度最高**
- 海外版 coze.com 稍早（2023-12 底海外 beta）
- 目标用户：PM、运营、内容创作者
- 发布到微信、QQ、飞书生态，深度绑定字节社交矩阵
- 扣子官方原话（可引用）：**"规划是最难以工程化的，用工作流编排方式，但这与大模型的开放性产生本质冲突"**

**③ 腾讯元器 (2024-05-17)**

- 腾讯基于混元大模型的智能体搭建平台
- 对标扣子 Coze
- ⚠️ 注意区分：**不是"腾讯元宝"**（后者是 C 端聊天 App）

**④ 百度：一个产品家族，按用户分层**

百度的 agent 编排产品不是一个，而是**按用户分层的两个主产品** + 一个历史演变：

- **历史**：灵境矩阵 (2023-09) → 文心大模型智能体平台 (2023-12) → **文心智能体平台 AgentBuilder** (2024-04-16 正式更名)
- **文心智能体平台 AgentBuilder (2024-04)** —— **对标扣子的 C 端产品**，非开发者友好
- **千帆 AppBuilder (2023-12-20 开放)** —— 百度智能云的**企业级** agent 开发工作台，面向开发者和企业
- 这两个是**不同产品服务不同用户群**，不要混为一谈

**⑤ 阿里云百炼 (Bailian / Model Studio)**

- 阿里云的大模型与智能体开发平台
- 集成阿里开源的 **AgentScope** 框架（通义实验室，23.4k stars）作为 SDK 层
- ⚠️ 注意：**不叫 "Visual Studio"**，媒体曾经用过 "ModelStudio-ADK" 这种称呼但不是官方产品名

**⑥ FastGPT (sealos/labring)**

- 开源知识库 + 工作流平台，**27k+ stars**
- 专注知识库 + 工作流垂直场景，不是通用 agent 平台
- 2023-02 项目创建，2023 年中成熟

#### 不属于这一类但经常被误放进来的

- **火山引擎 AgentKit** —— 字节云服务端的云原生 agent 基础设施（8 模块的执行底座），对标 AWS Bedrock AgentCore。**不是可视化编排平台**，是**基础设施层**
- **秒哒 MiaoDa** (百度) —— 2024-11 发布，零代码**应用生成器**（类 Lovable/v0），**不是 agent 编排平台**

#### 这一档的共同特征

- 核心抽象：**节点 + 连线 + LLM 作为一种节点类型**
- 都内置知识库、插件、工具调用
- 都面向非工程师（PM、运营、客服主管、内容创作者）
- 2026 年依然蓬勃生长——因为它们解决的是业务落地的确定性需求，不是技术先进性

---

### 五、第三类：Agent 代码框架 —— 开发者用代码写的 Agent 基础设施 (~2700 字)

**定义**：开发者用代码写 Agent 的库和框架
**和第二类的差别**：第二类是 UI 低代码产品，第三类是开发者写 Python/TypeScript 代码
**这一档的独特性**：**演化最快也最激烈**的一档。从 2022 年的 LangChain 到 2025 年的 LangChain 1.0，只用了三年，中间经历了 multi-agent 爆发、role-based 框架狂潮、以及最后的收敛

#### 按历史顺序列主流代表

**① LangChain v1 (2022-10-24)** —— 一切的起点

- 和 ReAct 论文同月出生（晚 18 天）
- `AgentExecutor` + 9 种 `AgentType`（ZeroShotReact / StructuredChatReact / OpenAIFunctions / ...）
- 所有早期变体都围绕 ReAct 的不同 prompt 格式展开

**② LangChain 2023-05：官方承认 ReAct 有问题** ⭐ 关键转折

- 2023-05-10 LangChain 博客发布 `plan_and_execute` 包
- 原话：**"complex objectives + reliability → prompt size unsustainable"**
- **业界第一次官方承认"单 ReAct 不够用"**
- 时间上正好卡在 AutoGPT/BabyAGI 爆发之后一个月——狂热刚退潮，问题立刻浮现
- 从这时起，代码框架开始往更复杂的方向探索

**③ AutoGen (2023-08 论文 / 09 开源) + MetaGPT (2023-08)**

- 学术界的 multi-agent 爆发
- role-based、对话式多 agent 范式
- 塑造了"AI team 比单 agent 强"的业界想象

**④ CrewAI (2023-11-14 开源，2024-10 商业化)** ⭐ Role-based 多 agent 爆款

- **"CEO / CTO / Analyst" 团队范式的旗手**
- 2024 年最火的 role-based 多 agent 框架
- 让"AI 团队协作"的想象具象化
- **伏笔**：2025 年被 Cognition + MAST 同时发文挑战

**⑤ LangGraph (2024-01-22)**

- LangChain 推出，用 StateGraph 重写 agent
- Pregel BSP 执行引擎，原生支持 cycles
- 起初主打 multi-agent subgraph 组合

**⑥ 2024-2025：新一代框架登场** (一段合并讲)

- **Mastra** (TypeScript 生态首选)
- **Google ADK** (Google 官方 Python 套件)
- **AgentScope** (阿里通义实验室，23.4k stars)
- **Pydantic AI** (类型最讲究)
- 这一批新框架的共同特征：外层是 workflow DAG 组合，内层每个 step 里跑 ReAct 循环
- "workflow 做骨架 + ReAct 做节点"成为 2025 年新框架的默认姿态

**⑦ LangChain 1.0 (2025-10-22)** ⭐ 一个句号

- 彻底重写为 LangGraph wrapper
- Classic 的所有老 API 挪到 `langchain-classic` 兼容包
- **连 `create_react_agent` 都 deprecated 了，新入口叫 `create_agent`**
- **象征意义**："连 ReAct 这个名字都不用挂了"——2022 年和 ReAct 论文同月出生的 LangChain，在 2025 年的终极 API 里连名字都不提 ReAct 了

---

### 六、总结：时间线 + 定调 + 一个可选视角 (~2500 字)

#### 6.1 统一时间线表（跨三类一张表）


| 年份         | Coding Agent                     | Agent 编排平台                                       | 代码框架                                                |
| ---------- | -------------------------------- | ------------------------------------------------ | --------------------------------------------------- |
| 2022-10    | ReAct 论文                         | —                                                | LangChain v1                                        |
| 2023-02    | —                                | FastGPT 项目创建                                     | —                                                   |
| 2023-03/04 | AutoGPT + BabyAGI                | —                                                | —                                                   |
| 2023-04    | —                                | **Dify 开源**                                     | —                                                   |
| 2023-05    | —                                | —                                                | LangChain `plan_and_execute`，官方承认 ReAct 有问题         |
| 2023-08    | —                                | —                                                | MetaGPT 论文                                          |
| 2023-09    | —                                | **灵境矩阵**（百度文心一言插件生态平台）                         | AutoGen 开源                                          |
| 2023-11    | —                                | —                                                | CrewAI 开源                                           |
| 2023-12    | —                                | Coze 海外版 / **千帆 AppBuilder**                    | —                                                   |
| 2024-01    | —                                | —                                                | LangGraph                                           |
| 2024-02    | —                                | **扣子 coze.cn 国内版上线**                             | —                                                   |
| 2024-03    | Devin                            | —                                                | —                                                   |
| 2024-04    | —                                | **文心智能体平台 AgentBuilder**（灵境矩阵改名升级）               | —                                                   |
| 2024-05    | —                                | **腾讯元器上线**                                       | —                                                   |
| 2024 底     | Claude Code / Cursor 成熟          | —                                                | —                                                   |
| 2024-10    | —                                | —                                                | CrewAI 商业化                                          |
| 2024-12    | —                                | —                                                | Anthropic *Building Effective Agents*               |
| 2025-03    | —                                | —                                                | **Berkeley + Cornell MAST 论文**                      |
| 2025-06    | —                                | —                                                | **Cognition "Don't Build Multi-Agents"**            |
| 2025-10    | —                                | —                                                | **LangChain 1.0** (create_react_agent 也 deprecated) |
| 2026       | Manus / Claude Co-worker         | —                                                | —                                                   |
| 2026       | Manus / Claude Co-worker           | —           | —                                                   |


#### 6.2 2025 年的转折点：Cognition + MAST 的双重证据

**2024 底到 2025 年是代码框架演化的重大转折**：

- 2024-12-19 Anthropic 发 *Building Effective Agents*，给 Workflow vs Agent 的 common vocabulary
- 2025-03-17 Berkeley + Cornell 发 MAST 论文 (arXiv:2503.13657)
  - 150 traces 构建 taxonomy + 1600+ traces 验证
  - 14 个失败模式，**8 / 14 是架构级失败**（不是实现级）
  - **定量证明了"multi-agent 系统失败 = 架构问题，不是模型不够聪明"**
- 2025-06-12 Cognition（Devin 团队）发 *Don't Build Multi-Agents* (Walden Yan)
  - Principle 1: "Share context, share full agent traces"
  - Principle 2: "Actions carry implicit decisions"
  - **Flappy Bird 例子**（视觉化记忆点）：一个 subagent 生成 Mario 风格背景，另一个生成不是游戏风格的鸟——两个 agent 各自"完成了任务"但合在一起是垃圾

**核心 insight**（可加粗）：

> **"问题不是'模型变强所以不需要复杂脚手架'，而是'复杂度本身是负资产'。**
> **multi-agent 的失败不是因为模型不够聪明，而是因为每次 handoff 都是一次隐式假设丢失的机会。"**

**这个共识对代码框架的影响**：

- 新一代代码框架（Mastra、Google ADK、Pydantic AI、AgentScope）默认都是"workflow 外层 + 单 ReAct 内层"
- LangChain 1.0 连 `create_react_agent` 都 deprecated，新名叫 `create_agent`
- CrewAI 依然存在但"role-based 多 agent"不再是业界共识
- **工作流平台（第二类）依然生长**——因为它解决的是业务确定性需求，这是另一条赛道，和技术先进性无关

#### 6.3 正在兴起的 / 正在消亡的

**正在兴起**：

- "workflow 外层 + 单 ReAct 内层"作为代码框架的默认模式
- LangGraph 作为 LangChain 生态的底层执行引擎
- 类型优先的框架（Pydantic AI）
- Agent-to-Agent 协议（Google A2A、AgentScope 的 A2AAgent）

**正在消亡**：

- LangChain Classic 的 9 种 AgentType
- Role-based 多 agent 原教旨主义（CrewAI 式 "CEO/CTO/Analyst"）
- 独立的 Plan-and-Execute 框架（LangChain experimental 已挪到 langchain-classic）
- 手写 ReAct while 循环
- LangGraph 的 `create_react_agent` API（2025-10 被 create_agent 取代）

---

#### 6.4 一个可选视角：Multi-agent vs Single-agent ⭐ 新增

**警告**：下面这一段是我个人写完这三类之后回头看的一个粗略视角，**不是严格分类，也不是业界共识**。放出来是因为它可能帮你更好地串起来这段历史——但如果你觉得不适用，完全可以忽略。

**这个视角是什么**：

如果你强行把"谁决定任务流"作为一个轴，可以把所有 Agent 粗略地分成两端：

- **Multi-agent 倾向**：人工把流程拆出来——UI 拖拽、代码编排、role 分工。**编排平台（第二类）** 大多属于这一端
- **Single-agent 倾向**：一个 LLM 在循环里自决。**Coding Agent（第一类）** 大多属于这一端
- **代码框架（第三类）**：两端都有，而且历史上从 Single（早期 LangChain）→ 实验 Multi（CrewAI / plan_and_execute / AutoGen）→ 回归 Single（LangChain 1.0 `create_agent`）

**为什么这个视角不严格**：

1. **边界模糊**：工作流平台里一个 Agent 节点内部跑 ReAct，算 Multi 还是 Single？
2. **Multi 和 Single 可以嵌套**：Mastra 外层 workflow DAG 是"人拆流程"，内层每个节点跑单 ReAct——它到底是 Multi 还是 Single？
3. **语义漂移**：2024 年讲的 "multi-agent" 主要指 CrewAI 式的 role 分工，2026 年讲的 "multi-agent" 可以指 workflow 里多个节点嵌套 agent，不是一个东西

**但它能帮你看到一个趋势**：

- 2022-2024 年代码框架倾向于做"人工拆流程"——不管是 role-based 多 agent（CrewAI）、Plan-and-Execute（LangChain experimental）、还是 multi-agent workflow（AutoGen）
- 2025 年起 Cognition + MAST 让业界承认"拆多了反而错得多"，代码框架收敛到"单 agent ReAct 循环 + 可选的 workflow 外层"
- **Coding Agent 从头到尾都没动**——它一直是单 LLM 大循环
- **工作流平台也没动**——它一直是"人画流程 + LLM 做节点"，和技术路线博弈无关

**一句话总结**：如果你想给这三类一个 one-liner，大概是——Coding Agent 是让 LLM 自己干，编排平台是给业务人员用来拆流程，代码框架是开发者在这两端之间做选择的工具箱，而 2025 年之后代码框架这一端的钟摆，从"拆"的方向摆回了"不拆"的方向。

---

### 七、结尾：这个坐标系怎么用 + 续集钩子 (~500 字)

**如何使用这个坐标系**：

下次听到"我在做 Agent"，先问一个问题：

> **你的产品形态是什么？**
>
> - 终端用户直接用 → 第一类：Coding Agent（Claude Code、Manus 等）
> - 非开发者在 UI 里拖节点搭 → 第二类：编排平台（扣子、Dify 等）
> - 开发者在代码里写 → 第三类：代码框架（LangChain、LangGraph、Mastra 等）

这三类各有各的用户、演化节奏、主流代表。混着说只会互相听不懂。

**续集钩子**：

这篇博客讲的是"盘点"。但有一个灰色地带这次没展开——某些生产场景里，Coding Agent 的自由度太高（给不了成本预估和审计），编排平台的灵活性又不够（加一个新 case 就是加一个分支），代码框架也不完全合适（Single-agent 默认不给 Plan 产物）。

**人们在这里做了一些介于三类之间的东西。** 下一篇我聊这个灰色地带，以及为什么我自己做的生产 Agent 系统是这样的。

---

## 研究依据索引

所有论据都有源码级 research 文件支撑：

- `summary.md` — Agent Map 完整分类
- `findings.md` — 9 条跨项目发现
- `mastra.research.md` / `google-adk.research.md` / `agentscope.research.md` / `pydantic-ai.research.md` / `langgraph.research.md` / `langchain.research.md` / `crewai.research.md` / `dify.research.md` — 8 份代码框架源码级分析
- `domestic-platforms.research.md` — 国内 7 平台调研
- `agent-paradigm-survey.reference.md` — 从 CyberMnema 带过来的历史资料
- `multi-agent-complexity.reference.md` — Cognition + MAST 详细讨论
- `history-verification.md` — 本博客引用的所有历史日期的独立求证报告

---

## v4.3 的关键变化（相对 v4.2）

- ❌ **不再把 Multi→Single 作为核心论点**。v4.2 里贯穿全文的 Multi/Single 对比在 v4.3 里被删除或淡化
- ❌ **§二 不再抛出 Multi/Single 作为"分类判据"**，改用更轻的"你在用它还是在搭它"作为分类起点
- ❌ **§三 §四 §五 的小节标题不再带 Single/Multi 标签**
- ❌ **AutoGPT vs BabyAGI 不再贴 Single/Multi 标签**，改为"ReAct 派 vs Plan-and-Execute 派"的历史事实描述
- ❌ **§五 不再叫"Multi → Single 收敛的主战场"**，改为"演化最快的一档"
- ✅ **§六 新增 6.4 "一个可选视角"**，把 Multi/Single 观察放在这里，明确标注"不严格、不权威、可忽略"
- ✅ **§六 6.2 改名为"Cognition + MAST 的双重证据"**，保留核心内容但不再用 Multi→Single 框架
- ✅ 其他所有历史事实、数据、时间线都不变

## 求证结果（2026-04-12，详见 `platform-verification.md`）

### ✅ 已求证通过，直接采用

- **Dify**：2023-04-12 GitHub 创建，137k+ stars（实测 137,260）
- **扣子 Coze**：国内版 coze.cn 2024-02-01，海外版 2023-12
- **腾讯元器**：2024-05-17 上线（⚠️ 不要和腾讯元宝混淆——后者是 C 端聊天 App）
- **百度产品家族**：灵境矩阵 (2023-09) → 文心智能体平台 AgentBuilder (2024-04-16 改名升级) + 千帆 AppBuilder (2023-12-20 企业级)
- **阿里云百炼 (Bailian / Model Studio)**：含 AgentScope SDK
- **FastGPT**：2023-02 创建，27k+ stars
- **Anthropic Building Effective Agents**：2024-12-19, Erik Schluntz + Barry Zhang
- **LangChain plan_and_execute 博客**：2023-05-10
- **Cognition Don't Build Multi-Agents**：2025-06-12 by Walden Yan，Flappy Bird 原文已抓取
- **MAST 论文**：2025-03-17, Berkeley + Cornell, arXiv:2503.13657
- **BabyAGI 架构**：三个子 agent + Pinecone + P&E 任务队列，全部正确
- **AutoGPT**：thoughts JSON 字段正确，改为"ReAct-style"更准

### ❌ 已从博客里移除

- ~~"阿里百炼 Visual Studio"~~ —— 不是产品名，改为"阿里云百炼 (Bailian / Model Studio)"
- ~~"ModelStudio-ADK"~~ —— 媒体造词，删除
- ~~火山引擎 AgentKit 作为第二类产品~~ —— 它是基础设施层（类似 AWS Bedrock AgentCore），移到"不属于这一类"说明栏
- ~~"妙搭"~~ —— 是"秒哒 MiaoDa"的错字，而且它是应用生成器不是 agent 平台，不进博客

### 🟢 求证过程中的加分收获

1. **百度产品的历史演变链**：灵境矩阵 → 文心大模型智能体平台 → 文心智能体平台 AgentBuilder——这个改名史本身很有故事
2. **腾讯元器 vs 腾讯元宝** 是常见混淆点，博客里可以一句话提醒
3. **Cognition Flappy Bird 原文已抓取**，正文起草时可以直接引用

