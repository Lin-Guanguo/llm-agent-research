# Blog #1: 到底什么是 Agent — Outline

Last Updated: 2026-04-11
Status: Outline ready, drafting next
Target length: ~12,000 characters (matching `llm-memory-research/blog.2.chinese.md` and `blog.3.chinese.md`)
Account: 小红书 "AI 观果"

---

## Central Thesis

> **"所有叫 Agent 的东西，本质上只有三种：一种是 LLM 自己在循环里决定下一步（Claude Code / Manus），一种是人画出流程然后让 LLM 在节点里做局部选择（Dify / Coze / Mastra / LangGraph），还有一种是先产出一个 Plan 再按 Plan 执行（阿里 ADK / 我自己的生产系统）。它们都叫 Agent，因为都有 LLM 在里面。但它们根本不是一回事——控制流谁说了算，决定了一切。"**

## Author's Vantage Point

- Personally builds and maintains a production Agent 2.0 system (P&E + typed dataflow + per-capability context filter)
- Production timeline: 2025-10 first launch as workflow → 2026-03-04 refactor reintroduced P&E
- Has read source-level of Claude Code, Codex, Gemini CLI, Mastra
- Has surveyed 7 Chinese domestic platforms (public sources only)
- **Authority anchor**: "I'm one of the few people who has actually shipped category ⑤ to production and read the code of categories ① and ④a"

## Writing Posture (calibrated with user's standard)

User's stated standard: "我要对一个领域里面至少一个子领域足够了解才能发。"

Mapping:
- **Sub-field I own** (deep authority): ⑤ Custom production P&E + typed dataflow
- **Sub-fields I've investigated with primary sources**: ① ② Coding CLI Agents, ④a Code frameworks (Mastra)
- **Sub-fields I've investigated with secondary sources**: ③ Chinese platforms (public blogs, docs)
- **Sub-fields I acknowledge as underexplored**: ④b P&E code frameworks (Alibaba ADK pending research)

The blog is written from ⑤'s vantage point looking outward — not pretending to be a neutral surveyor of all categories.

---

## Outline (五节)

### 开头钩子 (~500 字)

**Setup**: 2026 年 4 月，我在重构一个线上 Agent 系统。为了避开纯 ReAct 的 token 爆炸，我选了 Plan-and-Execute。一周后调研完业界才发现——**开源生态里"活着"的 P&E 实现已经不多了，主流框架都收敛到了 ReAct。**

**Hook**: 我本来以为这是一个技术选型问题。**结果发现，业界说的"Agent"根本不是同一个东西**。我踩着一个分类混乱往下看，才发现"到底什么是 Agent"这个问题没人说得清。

**Promise**: 这篇博客尝试给这个词划一道线。不是权威定义，是我一个生产系统开发者的视角。

---

### 一、先讲清楚三个你以为一样的东西 (~2500 字)

**核心动作**：把"Agent"这个词背后的三类产品拆开。用具体的例子，不讲术语。

#### 1.1 第一种 Agent：Claude Code 这一档

- **代表**：Claude Code、Codex、Cursor、Devin、Manus、Claude Co-worker
- **场景**：你说一句"把这个 bug 修了"，它自己去读日志、看代码、改文件、跑测试
- **控制流归谁**：LLM 在大循环里自己决定下一步
- **核心抽象**：文件系统 + shell tools + markdown memory
- **金句**：**"它像一个新员工——你给它一个模糊目标，它自己摸索怎么做。"**

#### 1.2 第二种 Agent：Dify / Coze 这一档

- **代表**：Dify、Coze（字节扣子）、腾讯元器、FastGPT、百度 AppBuilder
- **场景**：客服主管想搭一个自动回复机器人，不会写代码，打开 Coze 拖几个节点
- **控制流归谁**：**人在 UI 里画 workflow**，LLM 只是其中一个节点
- **核心抽象**：节点 + 连线 + LLM-as-节点
- **关键引用**：Coze 官方自己说的 **"规划是最难工程化的，用工作流编排与大模型开放性有本质冲突"**
- **金句**：**"它像一张 SOP 流程图——老板画好每一步，第三步'让 AI 帮忙选一下用哪个生成模型'就是'AI 功能'的全部意思。"**
- **5% 规律**：这一档的从业者共识——95% 的场景用 workflow 够了，只有 5% 需要 Agent 自主决策

#### 1.3 第三种 Agent：Mastra / LangGraph 这一档

- **代表**：Mastra、LangGraph、Vercel AI SDK（④a 档，代码写的 workflow）
- **场景**：开发者想搭一个生产级 AI 应用，用 TypeScript 写 workflow，某些 step 里嵌 ReAct Agent
- **控制流归谁**：**开发者在代码里写 workflow**，LLM 在 Step 内 ReAct
- **重磅类比**：**Mastra 本质上就是"TypeScript 版的 Dify"**——哲学完全一致，只是把可视化拖拽换成了代码

**本节收尾**：
- 这三种东西都叫 Agent，因为都有 LLM 在里面
- 但它们解决的问题完全不同（任务自主 vs 应用搭建 vs 业务编排）
- **一个更准确的区分判据**：控制流归谁？

---

### 二、我自己撞到的第四种：先 Plan 再 Execute (~2500 字)

**核心动作**：用自己的生产经验切入，讲第四种（④b + ⑤），也是本文最独家的部分。

#### 2.1 一次折返跑：2025-10 放弃 P&E，2026-03 又回来

- 2025 年 10 月：产品上线，本来计划 P&E，但业务太简单，**每次 Plan 都在规划同一套固定流程**，Plan 被砍，退化成 workflow
- 2025 年底到 2026 年初：workflow 的甜蜜期，但"加一个 case 就是加一个分支"开始痛
- 2026 年 3 月：重构，业务复杂度上来了，**纯 ReAct 会 token 爆炸**（每个结果都得让 LLM 看一眼）
- 在灵活性和成本可控之间，**P&E 是更老、但更合适的答案**
- 选型一句话总结：**"我们要的是预先知道会花多少钱"**

#### 2.2 调研时的发现：我在逆势

- LangChain 把 P&E 归档了
- 主流框架都是 ReAct + TodoWrite 的 scratchpad 模式
- Cognition（Devin 团队）写了 "Don't Build Multi-Agents"，推崇的是单 agent + 优秀压缩
- 开源生态里"活着"的 P&E 实现极少
- **感觉像是我一个人在 2026 年坚持用 2023 年的架构**

#### 2.3 但"逃离 P&E"的真相是"逃离多 Agent"

**这是本节的核心转折**：

- Cognition 反对的不是 Plan 层，是 agent-to-agent handoff 的信息丢失
  - 原则一："Share context, share full agent traces"
  - 原则二："Actions carry implicit decisions"
- 这两条打的是 handoff 丢信息，**不是 Plan 本身**
- Berkeley MAST 研究的 14 个失败模式里，8 个都在讲 handoff 和隐式假设丢失
- **单 LLM 的 P&E 从来没有 handoff 问题**
- 所以我坚持 P&E 不是守旧，而是**精确地避开了业界被诟病的那个点**

#### 2.4 为什么生产环境就是需要 Plan 层

- **预执行成本预估**：我需要在真正跑之前知道这次调用会花多少钱
- **审计和灰度**：合规要求能审，灰度要求能卡
- **失败时的回滚点**：Plan 结构化之后，回滚到哪一步是清晰的
- **这些东西纯 ReAct 给不了**
- **这是第四种 Agent 和前三种的本质差别**：有 Plan 产物

---

### 三、Agent 架构的真实地图 (~2500 字)

**核心动作**：把前两节的观察整理成一张清晰的表。这是文章的"payoff"。

#### 3.1 五档分类

```
| 档 | 代表                       | 控制流归谁           | 有 Plan 产物吗 |
|---|---------------------------|---------------------|---------------|
| ① | Claude Code, Cursor        | LLM 在 ReAct 大循环  | ❌            |
| ② | Manus, Devin, Co-worker    | 同①（云端）          | ❌            |
| ③ | Dify, Coze, 扣子           | 人画 workflow        | ❌            |
| ④a| Mastra, LangGraph          | 开发者写代码 workflow | ❌            |
| ④b| 阿里 ADK / AgentScope      | 代码级 P&E           | 可选          |
| ⑤ | 我的生产系统                | Plan 层 + typed flow | ✅（结构化）   |
```

#### 3.2 关键洞察

- **① ②**：架构完全一样，只是部署位置不同（本地 vs 云）
- **③ ④a**：哲学完全一样，只是"谁写 workflow"不同（拖拽 vs 代码）——**Mastra 是 TypeScript Dify**
- **④b ⑤**：真正有 Plan 产物的是这两档，但几乎没有人公开讨论过

#### 3.3 为什么这个分类不乱？

唯一需要的问题是：**"这个东西的任务流最终归谁决定？"**

- LLM 决定 → ① ②
- 人决定 → ③ ④a
- Plan 产物决定 → ④b ⑤

其他问题（单 vs 多 agent、ReAct vs P&E、代码 vs 可视化）都是次要的。

#### 3.4 一个有意思的观察：国内外生态的不对称

- 海外强的是 ④a（Mastra、LangGraph、Vercel AI SDK）
- 国内强的是 ③（Dify、Coze、扣子、FastGPT）
- 国内几乎跳过了 ④a，直接有少量 ④b（阿里 ADK）

这个不对称本身值得一篇单独的博客，这里点到为止。

---

### 四、那"Agent"到底应该叫什么？(~1500 字)

**核心动作**：把第三节的分类反过来用——给每一档一个更准确的名字。

#### 4.1 提议的更精确命名

| 现在叫 | 建议叫 |
|---|---|
| ① Claude Code 式 Agent | **自主任务 Agent** (Autonomous Task Agent) |
| ② Manus 式 Agent | **云端自主 Agent** (Cloud Autonomous Agent) |
| ③ Dify / Coze 式 Agent | **LLM 工作流平台** (LLM Workflow Platform) —— **不叫 Agent 更诚实** |
| ④a Mastra 式 Agent | **可编程 Agent 框架** (Programmable Agent Framework) |
| ④b 阿里 ADK | **Plan-and-Execute 代码框架** |
| ⑤ 生产系统 | **结构化 Agent 生产系统** (Structured Agent Production System) |

#### 4.2 最反直觉的一条建议

**③ 不应该叫 Agent，叫 LLM Workflow Platform 才诚实。**

理由：
- 它们的核心抽象是 workflow，不是 agent
- 它们的"Agent 模式"只是 workflow 里的一个节点类型
- 叫 Agent 只是因为"听起来更潮"
- 改名不会让它们变弱，只会让它们更准确地被理解

#### 4.3 这个分类的用处

下次你听到有人说"我在做 Agent"：
- 问一句"你的任务流是谁决定的？"
- 答案立刻告诉你他在哪一档
- 比问"你用 ReAct 还是 P&E"有用 100 倍

---

### 五、结尾：一个比"什么是 Agent"更有用的问题 (~500 字)

**核心动作**：不给定义，给一个诊断工具。

#### 5.1 真正有用的问题

> **"这个东西的任务流最终归谁决定？"**

这个问题把所有 Agent 放进对应的档位。

还有一个更具体的子问题（来自 Cognition 和 MAST）：

> **"当这个 Agent 出错时，错是因为某个 handoff 上的隐式假设丢了吗？"**

这个问题适用于所有档位，帮你诊断多 Agent 架构的失败模式。

#### 5.2 回到开头

我在 2026 年 4 月逆势选了 P&E。一周后我以为我是少数派。现在我知道——我只是在一个**几乎没人公开讨论的档位**里工作。这个档位不是错的，是**沉默的**。

如果这篇博客让一个读者意识到"原来我们也在做 ⑤ 这档事，只是没人这么说过"——这就够了。

---

## 待办与开放问题

### 写作前需要确认

1. **产品实名化？** — 用"我维护的线上 Agent 系统"还是"我们的产品"还是具体名字？
2. **第四档是否引用"阿里 ADK"的名字？** — 目前阿里 ADK 还没做深度调研，引用需谨慎
3. **typed dataflow 要不要在博客里展开？** — 建议不展开，留作下一篇续集的钩子

### 结构性疑问

1. 第二节（我自己的折返跑）可能太长，占 20% 字数是否过度？考虑压缩到 1500 字
2. 第四节的"重命名提议"是否太 prescriptive？可能需要软化语气
3. 小红书的硬核集受众能不能接受五档分类的复杂度？可能需要一个更视觉化的图

### 需要用 AI 画的图

- 图 1：三档 Agent 的对比示意图（可视化控制流谁决定）
- 图 2：折返跑时间线
- 图 3：五档分类表格（也可以用 markdown 表格）
- 图 4：Mastra ≡ TypeScript Dify 的结构对比

---

## 博客起草路径

1. 第一轮：按大纲直接写初稿，不管字数
2. 第二轮：量字数，如果超 15k 就重点删第二节
3. 第三轮：找 1-2 个核心金句加粗
4. 第四轮：确认所有引用（Cognition 原文、Coze 官方原话）都有来源
5. 第五轮：过一遍"一个不懂 Agent 的读者能不能看懂"，给每个术语加一句白话解释
