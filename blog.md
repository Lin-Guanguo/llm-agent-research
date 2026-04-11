# 当你在说开发 Agent 时，你到底在做什么？

Last Updated: 2026-04-12
Status: Outline for discussion (v1)
Target length: ~12,000 字符（对齐 `llm-memory-research/blog.2.chinese.md` 和 `blog.3.chinese.md`）
Account: 小红书 "AI 观果"

---

## Central Positioning

**系列第一篇**：盘点 + 建立坐标系 + 定调。不立论，不锋利，不展开个人观点。目标是让读者看完能获得一个**带得走的分类工具**——下次听到别人说"我在做 Agent"时，能问出正确的问题。

**作者视角**：盘点者 + 一手资料拥有者（源码级读过六个主流框架）+ 生产 Agent 开发者。**不在第一篇暴露自己是 ④ 档，留作续集钩子。**

**系列定位**：这是第一篇，为后续博客建立词汇。后续每一档都可以变成一篇深入的续集：
- Blog 2：为什么 P&E 被所有人绕开？我为什么捡起来做成生产默认（④ 深挖）
- Blog 3：Multi-Agent 是个陷阱吗？Cognition + MAST 的证据链
- Blog 4：typed dataflow——让 LLM 的输出受结构约束（⑤ 专题）
- Blog 5：Mastra 就是 TypeScript 版的 Dify：代码框架的真面目（③ 深挖）
- Blog 6：LangGraph 的 Pregel BSP 执行引擎（③ 深挖）

---

## Outline (v1 draft)

### 开头钩子（~600 字）

**切入场景**（咖啡店三个朋友）：
- 上周和朋友喝咖啡，A 说"我在做 Agent"——我问他在做什么，他说"在 Dify 拖了几个节点搭客服机器人"
- B 说"我也在做 Agent"——他在用 Claude Code 改代码
- C 说"我们团队在 CrewAI 搭一个 AI team"——role-based 多 agent

**Hook**：这三个人说的都叫 Agent，但它们完全不是一回事。"Agent" 是 2026 年 LLM 领域最被稀释的一个词。

**Promise**：这篇博客不是立论，是盘点。我会给你一个坐标系，把市面上所有叫 Agent 的东西都装进去。看完之后，你下次听到"我在做 Agent"能立刻问对问题。

---

### 一、为什么"Agent"这个词这么混乱（~1200 字）

**核心动作**：把"Agent 这个词指了多少种东西"的现实摆出来

**历史脉络**（简短）：
- Agent 在 AI 教材（Russell & Norvig）里本来是很宽泛的"感知-决策-行动"概念
- 2022 ReAct 论文被 LLM 圈改造成"思考-工具调用-观察"的大循环
- 2023-2024 Anthropic 的 *Building Effective Agents* 提出 "Workflow vs Agent" 的区分
- 2025 之后，营销和实际实现严重脱节

**三条导致混乱的原因**（点到为止）：
1. LLM 能力上升期，名词比实现跑得快
2. "Agent" 听起来比 "LLM 应用" 或 "workflow with LLM nodes" 更潮
3. 学术界、产品界、工具链作者对同一个词有不同理解

**金句（可加粗）**：**"'Agent' 现在是 LLM 领域最被稀释的一个词。从 Claude Code 到 Dify 里拖的一个 bot，从 Manus 到 CrewAI 的 role 全家桶，都叫 Agent。但它们有的是同一套架构的不同部署，有的是完全不同的物种。"**

---

### 二、引入分类框架：四档（~2200 字）

**核心动作**：给出分类的唯一判据，然后建立四档坐标系

**判据只有一个**：

> **"这个东西的任务流，最终归谁决定？"**

三种答案对应不同档位：
- LLM 在运行时自决 → **第一档**
- 人在设计时决定 → **第二档 或 第三档**（差别是作者是谁）
- 一个 Plan 产物决定 → **第四档**

**方案 C：四档**

| 档 | 名字 | 控制流归谁 | 代表 |
|---|---|---|---|
| **①** | 自主任务 Agent | LLM 在 ReAct 大循环里自决 | Claude Code, Codex, Cursor, Manus, Devin |
| **②** | 工作流平台 | 人在 UI 里画 workflow | Dify, Coze, 扣子, 元器, FastGPT, 百度 AppBuilder |
| **③** | Agent 代码框架 | 开发者在代码里写 workflow | Mastra, LangGraph, LangChain 1.0, Google ADK, AgentScope, Pydantic AI, CrewAI |
| **④** | 结构化生产系统 | 有结构化 Plan 产物 | （主要在企业内部，少数公开尝试） |

**关键洞察 1**：**① 的本地版和云端版其实是一档**——Claude Code（本地）和 Manus（云端）的架构完全一样，都是 ReAct + 工具集，只是部署位置不同。把它们分成两档没意义

**关键洞察 2**：**② 和 ③ 的哲学完全一样**——都是"人写 workflow + LLM 当节点"。唯一的差别是"谁写 workflow"：② 是非工程师在 UI 里拖，③ 是开发者在代码里写。**Mastra 就是 TypeScript 版的 Dify，Google ADK 就是 Python 版的 Dify，这不是比喻而是源码级的精确对应。**

**关键洞察 3**：**④ 档在公开世界里几乎都是 experimental**——这一点在定调章节展开

---

### 三、逐档盘点（~4500 字，全文核心）

每一档用统一结构：**定义 / 代表 / 目标用户 / 执行模型 / 场景例子 / 现状**

#### 第一档：自主任务 Agent

- **一句话定义**：给一个模糊目标，LLM 自己分解、自己选工具、自己验证结果
- **代表**（本地）：Claude Code、Codex、Cursor、Aider —— Coding 场景
- **代表**（云端）：Manus、Devin、Claude Co-worker、GenSpark —— 通用任务
- **目标用户**：Coding 的用户是开发者本人，云端的用户是终端消费者
- **执行模型**：纯 ReAct 大循环。`while True: LLM 决定下一步 → 调工具 → 看结果 → 继续`。工具集是文件系统 + shell + Python + browser
- **场景例子**："把 login.py 里那个 bug 修了" → Claude Code 自己去读日志、看代码、改文件、跑测试，完整循环 15 分钟
- **金句**：**"它像一个新员工——你给它一个模糊目标，它自己摸索怎么做。"**
- **现状**：2026 年**最火的一档**。Coding 侧 Claude Code 是 Anthropic 收入主力，云端任务 Agent 的 Manus 和 Devin 还在竞争领先位置

#### 第二档：工作流平台

- **一句话定义**：让非工程师在 UI 里拖拽节点，搭出一个"带 LLM 的业务 bot"
- **代表**：Dify（开源 137k ⭐）、Coze/扣子（字节）、腾讯元器、FastGPT、百度千帆 AppBuilder、阿里百炼 Visual Studio
- **目标用户**：产品经理、运营、客服主管——**非开发者**
- **执行模型**：节点 + 连线的可视化 workflow。某些节点是 LLM 节点（可以带 ReAct tool use），其他节点是确定性逻辑（知识库查询、API 调用、条件分支）
- **场景例子**：客服主管想搭一个能回答工单的机器人，打开 Coze 拖几个节点配置 prompt 和知识库，半天上线，发布到微信公众号
- **关键引用**：Coze 官方自己承认 **"规划是最难工程化的，用工作流编排与大模型开放性产生本质冲突"** ——国内最大 Agent 平台的产品团队公开说他们**主动绕开了**自主规划这个核心问题
- **金句**：**"它像一张 SOP 流程图——老板画好每一步，第三步'让 AI 帮忙选一下生成模型'就是'AI 功能'的全部意思。"**
- **现状**：**国内企业 Agent 落地的事实标准**。从"业界 5% 规律"（95% 场景用 workflow 就够，5% 才需要真正的 Agent 自决）可以看出这一档的市场逻辑

#### 第三档：Agent 代码框架 ← **这一档是博客里最硬核的部分**

- **一句话定义**：开发者在代码里写 workflow，把 LLM 做成 workflow 里某些节点的智能引擎
- **代表**：Mastra（TS）、LangGraph（Python）、LangChain 1.0、Google ADK、Pydantic AI、AgentScope、Vercel AI SDK、CrewAI（默认模式）
- **目标用户**：开发者，尤其是要把 LLM 集成进生产业务系统的人
- **执行模型**：**外层是 Workflow DAG，内层是 ReAct**。Mastra 的 `.then()/.parallel()/.dowhile()`、Google ADK 的 `SequentialAgent/LoopAgent`、LangGraph 的 `StateGraph`、Pydantic AI 的固定 4 节点图——**全部都是这个结构**
- **场景例子**：你要做一个客服 bot 的后端服务。在 Mastra 里写一个 `Workflow`：第一步 `.then(knowledgeBase.search)`，第二步 `.then(agentNode)`（LLM 带工具调用），第三步 `.then(ticketSystem.create)`。每一步都是开发者写的 TypeScript 代码
- **重磅类比**：**Mastra 就是 TypeScript 版的 Dify。Google ADK 就是 Python 版的 Dify。AgentScope 就是阿里版的 Dify。** 这不是比喻，是源码架构上的精确对应——它们和 ②档的真实差别只是"谁写 workflow"（UI 拖 vs 代码写），执行引擎本身是同一个物种
- **金句**：**"所有代码 Agent 框架本质上都是代码版的 Dify。你以为你在写 Agent，其实你在写 workflow。"**
- **现状**：**开发者构建生产 Agent 的绝对主流**。2026 年所有新发布的 Python/TS Agent 框架都属于这一档

#### 第四档：结构化生产系统（带 Plan 产物）

- **一句话定义**：先产出一个结构化的 Plan 对象，再按 Plan 执行每一步
- **代表**：**几乎都是企业内部系统**。公开世界里能找到的都是 experimental 或历史遗迹
- **执行模型**：Plan 阶段 → Execute 阶段，两者有明确分离和典型的 handoff
- **现状预告**：**这一档在公开世界里非常稀有——不是因为没人做过，而是因为做过的人都把它藏起来了。**这是下一篇博客的主题，这里先埋钩子

---

### 四、定调：主流 / 兴起 / 消退（~2000 字）

**这一节是博客的 payoff**，读者最想看的部分

#### 🔥 绝对主流（2026 年 Q2）

1. **ReAct 作为执行模型**：所有代码框架的默认。从 Claude Code 到 Mastra 到 LangChain 1.0，整个生态收敛到这一个答案
2. **可视化工作流平台**：国内企业 Agent 落地的事实标准，5% 规律成为业界共识
3. **代码框架 = Workflow outer + ReAct inner**：几乎所有代码框架都是这个结构
4. **单 agent + 优秀压缩**：Devin 团队 Cognition 的 *"Don't Build Multi-Agents"* 文章让"尽量用单 agent"成为主流共识

#### 📈 正在兴起

1. **类型优先的代码框架**：Pydantic AI 代表的"用 Python 泛型 + Pydantic 约束 LLM 行为"方向
2. **LangGraph 成为底层执行引擎**：LangChain 1.0 直接建在 LangGraph 上。LangChain 正在从"Agent 框架"变成"开发者 UX 层"，LangGraph 变成底层
3. **Agent-to-Agent 协议**：Google A2A、AgentScope 的 `A2AAgent` class、多家框架都在做跨服务的 Agent 通信标准
4. **BSP 执行引擎**：LangGraph 用 Pregel BSP 模型原生支持 cycles，和传统 DAG 根本不是一回事，正在被其他框架模仿

#### 📉 正在消退 / 已过时

1. **手写 ReAct while loop**：LangChain Classic 的 `AgentExecutor` 已全部废弃，1.0 改用 LangGraph
2. **Role-based multi-agent handoff 的原教旨主义**：CrewAI 的"CEO/CTO/Analyst"模式被 Cognition + Berkeley MAST 同时批评后，除了 CrewAI 自己很少有新入场
3. **LangChain 0.x 的 9 种 AgentType**：`ZeroShotReact`、`ConversationalReact` 全部废弃
4. **独立的 P&E 框架**：LangChain 实验包已死，LangGraph P&E 教程重定向到 middleware。**P&E 这条路不是消退，是被整个行业刻意关门了**

#### 💡 三个反常识的观察

1. **几乎所有"代码 Agent 框架"本质上是"代码版的 Dify"**——外层都是 workflow，内层才是 ReAct。它们和可视化平台的真实差别只是"谁写 workflow"
2. **"Multi-agent" 在 2024 是热词，到 2026 已经是警告词**——Cognition 和 MAST 的证据让"多 agent"成了有风险的字眼
3. **LangChain 和 LangGraph 的关系正在倒转**——历史上 LangGraph 是 LangChain 的一个子项目，2026 年 LangChain 1.0 反而成了 LangGraph 的 wrapper

---

### 五、结尾：这个坐标系怎么用（~500 字）

**核心动作**：给读者一个可带走的工具

**下次你听到"我在做 Agent"**：
1. 先问：**你的任务流是谁决定的？**
2. 答"LLM 自己决定" → 第一档
3. 答"我画的流程图" → 第二档
4. 答"我写的代码" → 第三档
5. 答"有一个 Plan 先出来" → 第四档

**续集钩子**：
- 第三档里其实还有一条暗线——每一个代码框架都评估过 Plan-and-Execute 然后把它藏进了 experimental
- 下一篇我们聊这条被所有人绕开的路，以及我为什么在生产环境选了它

---

## Pending Decisions (需要用户拍板)

1. **分几档？**
   - 推荐方案 C（4 档）：① 自主任务 / ② 工作流平台 / ③ 代码框架 / ④ 生产结构化系统
   - 备选方案 B（5 档）：单独拆出 ④a ReAct派 vs ④b P&E派
   - 备选方案 A（更激进的 4 档）：合并更多
   - **当前大纲用方案 C**

2. **要不要在结尾暗示自己做的是第四档？**
   - 建议：含蓄地提一句，作为续集钩子
   - 当前大纲已经这么写了（"下一篇我们聊⋯我为什么选了它"）

3. **要不要列具体的产品 URL 和 star 数？**
   - 建议：列——给读者可以顺藤摸瓜的线索，也让博客看起来有做功课
   - 当前大纲没列，起草时补

4. **开头的"咖啡店三个朋友"场景**
   - 建议：用——很戳人
   - 或换成"群里三个朋友"、"AI 观果读者来信"等更符合你个人调性的版本

---

## 研究依据索引

所有论据都有源码级 research 文件支撑：

- `summary.md` — Agent Map 完整分类
- `findings.md` — 9 条跨项目发现
- `mastra.research.md` — Mastra 源码级（论点 3）
- `google-adk.research.md` — Google ADK 源码级（论点 3）
- `agentscope.research.md` — AgentScope 源码级（论点 3、④）
- `pydantic-ai.research.md` — Pydantic AI 源码级（论点 3、兴起）
- `langgraph.research.md` — LangGraph 源码级（论点 3、兴起、定调）
- `langchain.research.md` — LangChain 源码级（论点 3、消退）
- `crewai.research.md` — CrewAI 源码级（论点 3、④、消退）
- `domestic-platforms.research.md` — 国内 7 平台（论点 2）
