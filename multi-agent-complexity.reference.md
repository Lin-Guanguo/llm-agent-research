# Agent 复杂度取舍：MultiAgent 还有必要吗

Last Updated: 2026-04-08
Topic Summary: 沿昨天的 Agent 架构调研继续。引入两条新材料：(a) 深读 Cognition "Don't Build Multi-Agents" 那篇 vendor post-mortem，(b) Berkeley 的 MAST 论文 (arXiv:2503.13657)——后者用 1600+ 条标注 trace 把 Cognition 的定性论证升级成定量证据。把 MAST 的 14 个失败模式与 Agent 2.0 的 typed dataflow 设计做精确映射，得到"结构上能消除 5 个、剩 9 个是单 LLM 调用质量和 verification 问题"的具体诊断。最后给出一个比"用不用 multi-agent"更有用的 meta 命题：**结构化没死，死的是"把结构强加在 LLM 之间的接口上"——typed 箭头从 2023 年的"约束 LLM 输出"反转为 2026 年的"在 LLM 周围搭结构"**。

## 背景

昨天写完 [Agent 架构调研](./Agent架构调研-2026-04-07.md) §10 之后，剩下两个没充分展开的问题：

1. Cognition 那篇 "Don't Build Multi-Agents" 在 §10.6 只被当作 "P&E 死因 #3" 引了一句，但它真正想说的事比"context 碎片化"更精确，值得单独深读
2. §10 的实证基础几乎全是 vendor blog 和单仓库源码审计，缺一份**学术 ground truth**——同期是否有论文用受控方法量化过 multi-agent 的失败模式？

今天的对话挖到了 Berkeley 那篇 **Why Do Multi-Agent LLM Systems Fail?** (arXiv:2503.13657, 2025-03)，正好补上了缺的那一块。所以这个 topic 是昨天 §10 的**直接续集**——重点在补强证据链 + 把昨天没回答的"Agent 2.0 应该往哪走"做一次具体诊断。

不和昨天 §10 重复——昨天讲"P&E 在 2026 年的生存现状"，今天讲"为什么复杂越来越没用 + 下一步具体怎么决策"。

---

## 一、相关材料

### 1.1 自有仓库

**CyberMnema（本仓库）：**

- [Agent 架构调研 (2026-04-07)](./Agent架构调研-2026-04-07.md) — 昨天的全景调研，本文的直接前置
- §10 节"2026 实证调研：开源 P&E 还剩下什么"——本文要交叉引用的那块
- §八.5 的反思（"独立重新发明前人答案"）——本文 §六 要在它上面再加一层

**llm-memory-research（个人外部研究仓库）：**

- `/Users/linguanguo/dev/llm-memory-research/summary.md` — 跨 20+ 项目的 memory/context 系统化对比，本文 §六 要引
- `/Users/linguanguo/dev/llm-memory-research/findings.md` — 10 个 cross-domain finding，特别是 #2 (两种哲学)、#5 (server-side trend)、#6 (sub-agent 是 compression 策略)、#9 (compression vs retrieval 二分)
- `/Users/linguanguo/dev/llm-memory-research/context.summary.md` — 7 个 agent 的 context 管理对比
- `/Users/linguanguo/dev/llm-memory-research/anthropic-context-engineering.research.md` — Anthropic 官方 context engineering 指引

### 1.2 学术论文

**今天新增的核心材料：**

- [arXiv:2503.13657 — Why Do Multi-Agent LLM Systems Fail?](https://arxiv.org/abs/2503.13657) (Cemri, Pan, Yang et al., UC Berkeley, 2025-03)
- [HTML 全文](https://arxiv.org/html/2503.13657v1)
- [PDF](https://arxiv.org/pdf/2503.13657)
- [MAST 数据集 GitHub](https://github.com/multi-agent-systems-failure-taxonomy/MAST) — 1600+ 条标注 trace
- [Hugging Face Paper Page](https://huggingface.co/papers/2503.13657)

**已在昨天 §参考来源中的相关论文（重列以方便定位）：**

- [arXiv:2408.02442 — Let Me Speak Freely?](https://arxiv.org/html/2408.02442v1) (Tam et al., 2024) — JSON mode 让 LLaMA-3-8B 在 Last Letter 任务降 38 点；本文 §五 与 typed dataflow 的代价讨论相关
- [arXiv:2509.08646 — Plan-then-Execute](https://arxiv.org/abs/2509.08646) — P-t-E 的安全性与 replanning 形式化
- [arXiv:2411.04468 — Magentic-One](https://arxiv.org/html/2411.04468v1) — Microsoft 唯一活跃的 hybrid P&E
- [arXiv:2311.05772 — ADaPT](https://arxiv.org/abs/2311.05772) — As-Needed Decomposition
- [arXiv:2404.11584 — Survey of Emerging AI Agent Architectures](https://arxiv.org/abs/2404.11584)

**值得补加但 CyberMnema 还没收的学术 survey（标 ⚠️ 的我不 100% 确定 arXiv ID，引用前请自行核对）：**

- ⚠️ arXiv:2309.07864 — "The Rise and Potential of LLM-based Agents: A Survey" (Xi et al., Fudan, 2023-09) — 2023 年的百科 survey
- ⚠️ arXiv:2308.11432 — "A Survey on LLM-based Autonomous Agents" (Wang et al., Renmin, 2023-08)
- ⚠️ arXiv:2404.13501 — "A Survey on the Memory Mechanism of LLM Agents" (2024-04)
- ⚠️ "Understanding the Planning of LLM Agents: A Survey" (2024-02) — 2024 年专门的 P&E zoo 百科，对照 §10 看可以看清楚"死亡时间线"

### 1.3 Vendor / 实践者 post-mortem

**今天深读的核心材料：**

- [Cognition — Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) — Walden Yan, 2025
- [Cognition — Don't Build Multi-Agents #applying-the-principles 节](https://cognition.ai/blog/dont-build-multi-agents#applying-the-principles) — Edit Apply 故事和 Claude Code subagent 的具体应用
- 关于 Cognition AI / Devin / Scott Wu / 2024 launch 的背景——见 §二.1 中的简介

**已在昨天 §参考来源中（重列）：**

- [Anthropic — Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) — Workflow vs Agent 二分的当代 common vocabulary
- [Anthropic — Multi-Agent Research System](https://www.anthropic.com/engineering/multi-agent-research-system) — Anthropic 自家 Claude Research 是怎么做的（dynamic spawn，不是预规划 DAG）
- [LangChain — Deep Agents blog](https://blog.langchain.com/deep-agents/) — TodoWrite no-op 原话的来源
- [LangChain — How and when to build multi-agent systems](https://blog.langchain.com/how-and-when-to-build-multi-agent-systems/) — 和 Cognition 那篇对着看
- [Manus — Context Engineering for AI Agents](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) — 公开承认放弃 todo.md
- [Pragmatic Engineer — How Claude Code is built](https://newsletter.pragmaticengineer.com/p/how-claude-code-is-built)
- [ZenML LLMOps — Claude Code single-threaded master loop](https://www.zenml.io/llmops-database/claude-code-agent-architecture-single-threaded-master-loop-for-autonomous-coding)

### 1.4 仓库参考实现

完整列表见昨天 [§参考来源](./Agent架构调研-2026-04-07.md#参考来源)。本文需要重点引用的：

- [Stanford STORM](https://github.com/stanford-oval/storm) — 长文本 P&E gold standard
- [GPT-Researcher](https://github.com/assafelovic/gpt-researcher)
- [LangChain `open_deep_research](https://github.com/langchain-ai/open_deep_research)` — 从严格 P&E 重写为 supervisor/ReAct hybrid 的 vendor 公开案例
- [BabyAGI Technical History](https://babyagi.wiki/) — BabyAGI 2o 的 "as LLMs improved, most agent scaffolding became unnecessary"
- [OpenManus PlanningFlow](https://github.com/FoundationAgents/OpenManus/blob/main/app/flow/planning.py)

---

## 二、为什么开这个 topic：从昨天 §10 的反向追问开始

昨天 §10 整体是描述性的——"市面上的 P&E 在 2026 年活成什么样了"。今天的对话从一个**追问**起步：

> 既然 P&E / 多 agent / 复杂脚手架普遍在退潮，那它们是**输给了什么**？是输给"模型变强了所以不需要"，还是输给"复杂度本身就是负资产"？

这两个解释听起来差不多，但 implication 完全不同：


| 解释          | 推论                                                                                   | 对 Agent 2.0 的意思                                                              |
| ----------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| 输给"模型变强了"   | 复杂脚手架在 2023 年是合理的，2026 年只是不再必要。模型再退化它们就有用了                                           | Agent 2.0 是"提前一代"的工程，等模型某天又遇到瓶颈时还会回潮                                         |
| 输给"复杂度是负资产" | 复杂脚手架**本身**就在引入失败模式，模型变强只是让原来藏住的 bug 暴露出来。即使把模型固定在 2023 年水平，Cognition / MAST 的论点依然成立 | Agent 2.0 在某些维度上**永远**在和复杂度做对——typed dataflow 解决一部分，但剩下的是 multi-agent 架构的固有税 |


整个今天的工作就是确认到底是哪一种解释。**结论：是后一种**。证据来自三条独立线：

1. **Cognition 的定性 production 经验**（vendor 角度）
2. **Berkeley MAST 的 1600 trace 量化**（学术角度）
3. **Tam et al. "Let Me Speak Freely?" 的 reasoning 退化测量**（单点定量证据）

三条都不依赖"模型变强"这个外因。它们说的都是：**复杂度本身在制造问题**。

---

## 三、Cognition "Don't Build Multi-Agents" 深读

### 3.1 关于 Cognition 的来历

Cognition AI 是做 **Devin** 的那家公司。Devin 是 2024-03 那个号称"第一个 AI software engineer"的产品，发布 demo 在 Twitter 上炸过一轮。CEO **Scott Wu**，著名竞赛程序员出身（IOI 金牌、Putnam 高分），早期团队大量竞赛/算法 background。**Walden Yan**（"Don't Build Multi-Agents" 的署名作者）是联合创始人之一。

2024-04 Founders Fund 领投 Series A，估值约 $2B。早期 demo 后被 YouTuber **Internet of Bugs** 做过非常严苛的逐帧拆解，证明部分任务是 cherry-pick + 人工干预。"Cognition 是否在 overpromise" 在 2024 年是一个公开争议。

**为什么这篇文章信息密度高**：一个**商业上靠卖"自主 agent"的公司**，反过来论证"agent 不该被这样搭"——这种 self-disconfirming statement 在 vendor 博客里非常稀有。一般 vendor blog 是 marketing；这种主动唱反调的，往往是真踩过坑后的复盘。

（注：训练数据截止 2025-05，2025 下半年到 2026 之间 Cognition 是否被收购、Devin 是否还在卖、产品形态有没有变，本文未核实。引用时建议自行查 2026 年近况。）

### 3.2 两条原则的精确措辞

直接从 cognition.ai/blog/dont-build-multi-agents 取的原话：

> **Principle 1**: "Share context, and share full agent traces, not just individual messages"
>
> **Principle 2**: "Actions carry implicit decisions, and conflicting decisions carry bad results"

最佳的具体例子是博客里那个 Flappy Bird clone：一个 subagent 生成 Mario 风格的背景，另一个 subagent 生成**不是游戏风格**的鸟。两个 agent 都"完成了任务"，但合在一起就是垃圾。原因是 planner 的隐式假设（"游戏美术风格"）没有被显式传给两个 subagent，而它们也没办法相互看到对方在做什么。

### 3.3 论点其实有四层嵌套

第一次读 Cognition 那篇，容易把它理解成"don't split agents"。但它实际上在说四件**层层嵌套**的事，强度依次降低：

1. **共享 > 分裂** —— 当你犹豫要不要拆 agent，默认不拆
2. **完整 trace > 单条消息** —— 即使必须拆，传给下游的应该是完整的 agent trace，不是被压缩过的"主旨"
3. **顺序 > 并行** —— "parallel" 工作宁可串行化，也要保持单一上下文
4. **一个模型干两件事 > 两个模型各干一件** —— 这是 Edit Apply 那个例子的真正含义

第 4 条是最锋利的一刀，**它不是**"加更多信息"。Edit Apply 故事里他们没加任何信息——他们就是把"决策模型 + 应用小模型"两段流水线塌成"一个模型同时决策和应用"。**没有"看得更多"，只是少了一次模型之间的解释 handoff**。

### 3.4 单线程不是"trust the model"，是"少一次解释"

这是今天的对话里最关键的精确化：**"精确掌控上下文"和"让模型看到更多"不是真正的对立面**。它们俩的共同对立面是：

> 在多个 LLM 调用之间精确传递结构化数据。

证据：Claude Code 是单线程，但它**绝不是**"啥也不管，把原始数据扔给模型"。它有 system prompt、有 CLAUDE.md、有 tool definition、有 context editing API、有 TodoWrite 注意力工程。它**仍然在精确控制上下文**——只是控制的对象从"LLM 之间的 wire format"变成了"喂给单个 LLM 的所有东西"。

也就是说，结构化没死，**死的是"把结构强加在 LLM 之间的接口上"**。这一句话是本文 §六 的命题预演。

### 3.5 他们的真实押注：单线程 + 优秀压缩

Cognition 那节里有一句很容易被忽略的话：

> "For lengthy tasks, they propose introducing a specialized LLM model designed to compress action histories... though they acknowledge this requires significant investment to implement effectively."

这一段的隐含信息是：**他们承认"单线程 + share everything"撞到 context 上限是真问题，而且他们承认压缩很难做对**。所以 Cognition 的真实立场不是"infinite context, trust the model"，而是更精确的：

> 单线程 + 优秀压缩 > 多 agent + 接口传递

他们押注的是"压缩做对"比"接口设计对"更可行。这是一个非平凡的赌注，因为压缩做对同样很难（见 llm-memory-research findings.md §3——"压缩质量是共同的未解决问题"）。但 Cognition 的判断是：**两种"难"之间，压缩失败是逐步的、可观测的；接口失败是突然的、隐藏的**。前者更工程化、更可调试。

这也解释了为什么 Claude Code 在压缩这件事上下了那么大功夫（9-section summary、server-side compaction、`<budget:token_budget>`让模型自己管预算）——他们走的是同一个赌注。

---

## 四、Berkeley MAST：把定性论证变成定量证据

### 4.1 论文出处

**Why Do Multi-Agent LLM Systems Fail?** — Mert Cemri, Melissa Z. Pan, Shuyi Yang 等，UC Berkeley，arXiv:2503.13657（2025-03 首版）。资深作者团队来自 Berkeley Sky Computing/RISELab 那条线。开源了一个叫 **MAST** (Multi-Agent System failures Taxonomy) 的标注数据集，含 1600+ 条 trace，覆盖 7 个主流多 agent 框架。

**研究的框架包括：MetaGPT、ChatDev、HyperAgent、AppWorld、AG2 (前 AutoGen)**，外加 AutoKaggle、Multi-Agent Peer Review、MA-ToT。**这正好覆盖昨天 §10.2 表里那些"marketing 是 P&E、代码是 ReAct"的项目**——MetaGPT、AG2 你已经吐槽过了。

标注质量：**inter-annotator kappa = 0.88**（强一致），方法学上不容易被攻击。

### 4.2 核心产物：14 个失败模式 × 3 个类别


| 类别                                            | 失败模式                                                                                                                                                                                               |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **FC1: Specification & System Design**（5 种）   | FM-1.1 Disobey task spec / FM-1.2 Disobey role spec / FM-1.3 Step repetition / FM-1.4 Loss of conversation history / FM-1.5 Unaware of termination conditions                                      |
| **FC2: Inter-Agent Misalignment**（6 种）        | FM-2.1 Conversation reset / FM-2.2 Fail to ask for clarification / FM-2.3 Task derailment / FM-2.4 Information withholding / FM-2.5 Ignored other agent's input / FM-2.6 Reasoning-action mismatch |
| **FC3: Task Verification & Termination**（3 种） | FM-3.1 Premature termination / FM-3.2 No or incomplete verification / FM-3.3 Incorrect verification                                                                                                |


### 4.3 统计学发现：失败不集中

**没有任何一个 mode 占绝对多数**。不同系统的失败分布形状不同（AG2 和 ChatDev 的曲线就不一样），但每个类别都贡献相当比例的失败。这是"没有 silver bullet" 的实证证据——你不能通过修一个 bug 就让多 agent 系统能用。

### 4.4 +14% 上限：决定性的一刀

整篇论文最有价值的一句结论：

> "improved prompting achieved only +14% gains on ChatDev, remaining insufficiently low for real-world deployment"

意思是：他们**实测**了 prompt engineering 和 orchestration 改进，**只能把成功率拉高 14 个点，远不到生产可用线**。结论原话是 "obvious fixes" 有 "severe limitations"，要求 "structural solutions"（结构性解决方案，不是 tactical 修补）。

implication 极强：

> 多 agent 系统的失败**不是工程实现问题**，是**架构问题**。换更好的 prompt、加 retry、做 self-reflection——这些都顶不到生产线。

这就把 Cognition 那篇从"vendor 主观经验"升级为"academic 实证证据 + 因果解释"。**§10 缺的"独立来源交叉验证"环节，MAST 直接补上**。

### 4.5 与 Cognition 两条原则的精确映射（8/14）

把 14 个 MAST mode 和 Cognition 那两条原则做映射，**8 个直接对应**：

**Cognition Principle 1（"share context, share full agent traces"）→ 4 个 mode：**

- FM-1.4 Loss of conversation history
- FM-2.1 Conversation reset
- FM-2.4 Information withholding
- FM-2.5 Ignored other agent's input

**Cognition Principle 2（"actions carry implicit decisions"）→ 4 个 mode：**

- FM-1.1 Disobey task spec
- FM-1.2 Disobey role spec
- FM-2.3 Task derailment
- FM-2.6 Reasoning-action mismatch

**Cognition 在 2024 年靠 production 经验定性说出来的两个原则，2025 年 Berkeley 用 1600 条标注 trace 量化证实了**。两者独立得出，相互验证。这是这个话题第一次有 vendor 经验和学术实证的硬交叉。

### 4.6 与昨天 §10.6 失败原因的对应

昨天 §10.6 列了 5 条 P&E 死因。MAST 的 14 个 mode 里：


| §10.6 死因/                      | MAST 对应 mode                                   |
| ------------------------------ | ---------------------------------------------- |
| #1 Plan 执行中过期                  | FM-1.4 + FM-2.6 + FM-1.3                       |
| #2 结构化输出降推理质量（Tam et al. 38 点） | MAST 不直接覆盖（这是 single-LLM 的问题，不是 multi-agent 的） |
| #3 Cognition context 碎片化       | FC2 整个类别（6 modes）                              |
| #4 Token / latency 开销          | MAST 不直接覆盖                                     |
| #5 步骤数不可预测                     | FM-1.5 + FM-3.1                                |


**§10.6 的 5 条死因里，3 条被 MAST 量化覆盖了**。剩下 2 条不在 MAST 范围里——它们是 single-LLM 自身的问题，不是 multi-agent 的问题。但这反而证明 §10.6 的归类是对的：multi-agent 的特有问题和 single-agent 的特有问题应该分开看，MAST 占了前者的大部分。

---

## 五、用 MAST 诊断 Agent 2.0：5 vs 9 的劈分

把 14 个 mode 又跑一遍，问：**Agent 2.0 的 typed dataflow 在结构上能 prevent 哪些？**

### 5.1 typed dataflow 在结构上能 prevent 的 5 个 mode


| Mode                                | 为什么能 prevent                             |
| ----------------------------------- | ---------------------------------------- |
| FM-1.4 Loss of conversation history | typed blackboard 显式保留状态，不会"丢"            |
| FM-2.4 Information withholding      | schema 强制输入字段必须就绪才能 fire，下游"扣留信息"在结构上不可能 |
| FM-2.5 Ignored other agent's input  | 依赖图让"忽略"在结构上不可能——没消费就不会 fire             |
| FM-1.1 Disobey task spec            | schema 是约束，不能输出 schema 之外的字段             |
| FM-1.5 Unaware of termination       | plan 有明确的终止条件（DAG 节点跑完）                  |


这 5 个 mode 是 typed dataflow 架构本身**结构上**消除的——不是靠 prompt 或 retry，是靠数据结构。

### 5.2 救不了的 9 个 mode


| Mode                                 | 为什么 typing 救不了             |
| ------------------------------------ | -------------------------- |
| FM-2.6 Reasoning-action mismatch     | 发生在单次 LLM 调用内部，typing 管不着  |
| FM-1.3 Step repetition               | typing 不看历史，重复执行不违反 schema |
| FM-1.2 Disobey role spec             | 语义约束，typing 是结构约束          |
| FM-2.3 Task derailment               | 同上                         |
| FM-2.2 Fail to ask for clarification | 行为模式，不是数据流                 |
| FM-2.1 Conversation reset            | 实现层 bug，和架构无关              |
| FM-3.1 Premature termination         | LLM 自身判断问题                 |
| FM-3.2 No or incomplete verification | 同上                         |
| FM-3.3 Incorrect verification        | 同上                         |


注意这 9 个里面有一个明显的聚类：**verification 类全部三个 + reasoning-action mismatch + role spec + task derailment**——这些都集中在"单次 LLM 调用的质量"和"verification 怎么知道结果对"上。

### 5.3 这个分析回答了昨天 §七 没回答的问题

昨天 §七 写"Agent 2.0 可继续探索的方向"列了三条（replanning / DAG 化 / HITL 检查点），但**没有判断哪条 ROI 最高**。MAST 的 5/9 劈分给出了具体回答：

> Agent 2.0 的 typed dataflow 在结构上消除了约 1/3 的 MAS 失败模式。剩下 2/3 是 typing 救不了的，主要分布在"单次 LLM 调用质量"和"verification 质量"上。

implication：

1. **再投资 typed dataflow 边际收益很低**——你已经覆盖了它能覆盖的全部，再做也只是 5 个 mode 上的边际改进
2. **下一步真正的瓶颈是"agent 内部"问题**：
  - (a) 单次 LLM 调用的 reasoning 质量 → **接入推理模型**（o3 / Sonnet thinking）的杠杆远大于 prompt 优化
  - (b) verification 层 → **怎么知道某一步的产出是对的**。这是 §10.6 死因 #1 在 Agent 2.0 上的具体落地形式
3. **昨天 §七 列的三个方向，按这个框架重新排序**：
  - HITL 检查点 → 治 FM-3.x 三个 verification mode，**ROI 最高**
  - replanning → 治 FM-2.6 reasoning-action mismatch，**ROI 中等**
  - DAG 化 → 已经在 5 个被覆盖的 mode 范围内，**ROI 低**

### 5.4 这个分析也告诉你 typed dataflow 没死

把 §五 反过来读：**typed dataflow 仍然是结构上消除 5 个 MAS 失败 mode 的最有效办法之一**。Cognition 的 "don't build multi-agents" 处方是"全部塌缩到单 LLM"，这能消除全部 14 个 mode（因为没有多 agent 了），但代价是 (a) 失去并行、(b) 失去可审计性、(c) context 上限直接撞墙。

**typed dataflow 是另一条路**：用结构化约束消除 5 个 mode，剩下 9 个用其他手段补。这条路的成本低于"全塌缩"：

- 仍然能并行
- 仍然有显式 plan 可以 review
- 仍然能精确缓存中间结果

所以 §六.4 那个"Agent 2.0 在 2026 范式里逆流"的判断需要补一句：**逆流不等于错路。逆流只意味着你比别人多承担一些工程税，换来 5 个 mode 在结构上被永久消除**。这个交易在你的具体域里是否合算，取决于你的任务对那 5 个 mode 有多敏感。

---

## 六、范式拐点：typed 箭头的反转

### 6.1 一个新的 meta 命题

把今天的所有讨论收束成一句：

> **结构化没死。死的是"把结构强加在 LLM 之间的接口上"。typed 箭头从 2023 年的"约束 LLM 输出"反转为 2026 年的"在 LLM 周围搭结构"**。

这是 §三.4 里的精确化命题的一般化版本。

### 6.2 两种结构化的对比

```
2023 的信仰：结构是 LLM 输出的约束
  → "让 planner 输出 typed plan object"
  → "用 Pydantic schema 强制对齐"
  → DSPy Signature / ReWOO / LLMCompiler / Agent 2.0
  → 代价：让 LLM 戴着镣铐说话，触发 reasoning 退化（Tam et al. 38 点）

2026 的实践：结构是 LLM 外部状态的表示
  → "让 LLM 自由写，系统事后结构化"
  → Claude Code TodoWrite（LLM 自由产出，系统结构化存储）
  → Mem0 / Graphiti（LLM 说话，另一个 LLM 抽 fact）
  → Artifact 系统（结构是 side channel，不在控制流上）
  → MCP tool schemas（边界 typed，内部 free-form）
  → 代价：需要一个旁路抽取层；状态可能和"LLM 心里想的"漂移
```

两者都有"结构"，但第一种是**把 LLM 塞进盒子**，第二种是**在 LLM 外面搭盒子**。MAST 的数据 + Cognition 的论点 + Tam et al. 的退化测量 一起证明第一种在生产里输了；但第二种正在以 memory 系统、context artifact、MCP 边界的形式**反而越来越流行**。

精确描述：**LLM 的输出应该是自由的，但 LLM 的"状态"应该是结构化的**。自由和结构不再同时作用在同一件东西上。

### 6.3 LangGraph 和 P&E 的关系也要重新定位

今天对话里另一个收获：**Plan-and-Execute 在结构上就是"LangGraph 固定边模式 + LLM 当作者"**。

把"谁来写控制流图"作为光谱：


| 位置  | 谁写 DAG      | 什么时候写       | 代表                                      |
| --- | ----------- | ----------- | --------------------------------------- |
| 1   | 人           | 编码时         | LangGraph base tutorial（`add_edge` 固定边） |
| 2   | 人 + 少量运行时分支 | 编码时 + 少量 if | LangGraph `add_conditional_edges`       |
| 3   | **LLM**     | **每次任务开始时** | **Plan-and-Execute**                    |
| 4   | LLM         | 每一步重新决定     | ReAct                                   |
| 5   | LLM + 搜索    | 并行展开多条      | ToT / LATS                              |


P&E ≈ LangGraph base 的"作者权 lifting"——把 DAG 作者从人/编码时换成 LLM/运行时开头，执行语义完全不变。LangChain 官方的 P&E tutorial 就是直接写成 LangGraph 图的（planner 是 node、execute 是 node、replan 通过 conditional edge 回到 planner）。

但**这个 lifting 在两个维度上发生**，不是一个：

1. **作者**（人 → LLM）
2. **节点粒度/类型**（typed function → 自由文本 → 结构化 JSON → typed dataflow）

经典 P&E（ReWOO）在维度 1 上 lift 了，维度 2 没动（节点是字符串占位符）。Agent 2.0 在两个维度上都走到了顶——LLM 写 DAG **而且** 节点是 typed dataflow。这是它在 ReWOO 谱系里"走得比任何论文都远"的原因。也是它在 2026 年"找不到同代产品"的原因——因为生态在两个维度上都退潮了。

### 6.4 §八.5 的反思要补一条

昨天 §八.5 写的"独立重新发明前人答案"现在可以加一条**反向**条款：

**前一版反思**："碰到工程问题时，去搜 dataflow / 流处理 / 增量计算等领域——前人已经做过"。

**今天补一条**："但是也要搜**最近 12-24 个月的 post-mortem 和 deprecation 记录**，因为前人做过的东西**也会过时**。"有文献"不等于"还是好主意"。"

完整版变成：

1. **碰到工程问题时**：先搜相关理论领域，前人大概率做过
2. **但是**：也要搜最近 12-24 个月的 post-mortem，因为旧答案会被新证据推翻
3. **特别警惕"marketing 与 code 脱节"**：2026 年开源 agent 生态里大量项目打着 "P&E" 的旗号但代码是 ReAct，去读源码不信 README（昨天 §10.2 的发现）
4. **frontier 模型能力的提升会悄悄改变架构选择**：2023 年 P&E 合理是因为 GPT-3.5 planner 比 executor 强；2026 年 Sonnet 4.5 / Opus 4.6 够聪明了，"scaffold 代替模型 reasoning" 的理由消失了
5. **新加**：**架构选择有"结构化方向"的维度**——同样是结构化，2023 范式选了"约束 LLM 输出"这条死路，2026 范式选了"在 LLM 周围搭结构"这条活路。判断一个新设计要看它把 typed 箭头指向哪边。

---

## 七、回到原问题：MultiAgent 还有必要吗

### 7.1 三种仍然合理的场景（沿用昨天 §10.8 的分类，加 MAST 的视角）


| 场景                                                                   | 为什么仍然合理                                                | MAST 视角                                                    |
| -------------------------------------------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------- |
| **长文本 / 多页结构化内容生成**（STORM / GPT-Researcher / OpenManus PlanningFlow） | 任务天然有层级（outline → section → paragraph），"写什么"大部分独立于执行反馈 | 这类任务 FC2 (inter-agent misalignment) 风险低，因为 sub-task 之间天然解耦 |
| **Read-only + 并行 fan-out**（Deep Research）                            | 动作空间是只读，世界不会在脚下改变；errors 的代价只是浪费几次搜索                   | FC3 (verification) 是主要问题但相对可控；FC2 因为没有写入冲突，影响小             |
| **多 agent orchestration 需要 HITL / 审计**（Magentic-One）                 | plan 是天然的 review point                                 | 用人工 verification 兜底 FC3，剩下 FC1+FC2 用 typed interface 控制    |


### 7.2 一个比"用不用 multi-agent"更有用的诊断问句

Cognition 那篇真正的实用价值不在结论，而在它给的**因果解释**。一旦接受这个因果，就有了一个通用的诊断工具：

> 任何时候你的 agent 系统出错，先问一句：**这个错是某个 handoff 上的隐式假设丢了吗**？

- 如果**是**：处方是"减少 handoff"或"显式化假设"
- 如果**不是**：问题在别处，handoff 数量不是关键变量

**这个诊断问句比"用不用 multi-agent"有用 100 倍**。它把"架构选择"问题转化成了"具体 bug 的根因"问题。Cognition 给的不是答案，是问句。MAST 的 14 个 mode 是这个问句的 "yes" 分支的具体清单。

### 7.3 给 Agent 2.0 的具体决定

综合 §五 的 5/9 劈分 + §七.1 的场景表 + §七.2 的诊断问句：

**短期（持续投入 Agent 2.0）：**

1. 把当前的 typed dataflow + Planner + Executor 作为既定架构，不动
2. 下一步重点投在**两个未覆盖区**：
  - **verification 层** —— 治 FC3 三个 mode，ROI 最高。具体形式是 HITL 检查点 + 程序化校验扩展
  - **单次 LLM 调用质量** —— 用推理模型替换关键节点的 LLM，特别是 Planner。用 reasoning 模型的内部 CoT 替代部分外部脚手架
3. 不投入 DAG 化 / 更激进的 typing —— 已经在 5 个被覆盖的 mode 范围内，边际收益低

**中期（监测信号）：**

- 跟踪每次失败：是不是 §五.2 那 9 个 mode 之一？如果是，记下哪个
- 如果 9 个 mode 出现频率开始集中在 1-2 个，考虑针对那 1-2 个做专门的 mitigation
- 如果失败大量集中在 FM-2.6 (reasoning-action mismatch) 或 FM-1.3 (step repetition)，那是 Planner 质量在拉低系统——切推理模型
- 如果失败大量集中在 FC3 verification，那是输出层缺校验——上 HITL 或更严格的程序化校验

**长期（架构 re-evaluation 触发条件）：**

- 如果**任务域开始向 coding / 计算机使用 / 开放探索漂移**——典型 ReAct 域——考虑切到单 LLM + TodoWrite 模式。这是 §10.9 里"路径未知 + 动作影响 state"那一格
- 如果**reasoning 模型变得足够便宜**（比如 o4-mini 价格降到当前 GPT-4 水平），重新评估"是不是 Planner 一个推理模型 LLM 就能干掉整个 plan + execute 流程"——这是 BabyAGI 2o → 174 行的同款坍缩
- 如果**生产中频繁出现"plan 看起来对但执行结果不对"**——FM-2.6 + FM-1.4 的组合症状——证明 handoff 问题已经在你这边显现，不能再忽略。这时候要 seriously 考虑 Cognition 路线的塌缩

### 7.4 收束：复杂度不是英雄主义

整个今天的工作其实在说一件事：**Agent 复杂度不是英雄主义**。过去两年很多 vendor 把"我们的 agent 有 N 个角色 / N 步规划 / N 层 verification"当成卖点，**这是一种工程上的反向选择压力**——MAST 证明每多一层 handoff 就多一次失败机会，Cognition 证明每多一个 sub-agent 就多一次隐式假设丢失。

复杂度的正确角色是：**当且仅当某个具体失败模式不能用更简单的办法消除时，引入正好够消除它的复杂度**。typed dataflow 是这种意义上正当的——它精确消除 5 个 MAST mode 而没有引入新的 mode。但 typed dataflow + Planner role + Executor role + Validator role + Critic role + Replanner role 这种"全家桶"式的多 agent 架构就是反向的——每加一个 role，5 个 mode 没多消除一个，但 FC2 inter-agent misalignment 的 6 个 mode 全部加权。

Agent 2.0 的设计在今天的视角下仍然是**有原则的复杂度**——它复杂，但每一份复杂度都对应一个可以指出的 mode。这是它和 MetaGPT / CrewAI / AutoGen 那种"role 全家桶"架构的根本区别。**不要因为"P&E 在 2026 退潮了"就放弃 Agent 2.0；要因为"5 个 mode 已经覆盖完了"而把下一步的精力投到剩下 9 个上**。

---

## 待续 / TODO

- 读完 MAST 论文全文（本文只读了 abstract + 摘要 + 关键段落），核对 14 个 mode 的精确定义和论文中的具体频率分布数据
- 核实 §一.2 标 ⚠️ 的几个 arXiv ID（2309.07864 / 2308.11432 / 2404.13501），加入参考来源
- 找一篇 2025-2026 关于 Cognition 现状的文章，更新 §三.1 的公司背景脚注
- 把本文 §五 的"5/9 mode 劈分"作为 Agent 2.0 的内部技术备忘录，team 内部对齐"下一步该投哪儿"
- 后续值得开新 topic 的方向：**verification 层在 LLM agent 里的设计模式**——FM-3.x 三个 mode 是当前 Agent 2.0 最大的未覆盖区，但学术界和工程界都没系统化的处方

