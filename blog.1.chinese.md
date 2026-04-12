# 当你开发 Agent 时,你在做什么?

> Agent 开发全景图上,大家做的事各不相同,但都是 Agent。

Last Updated: 2026-04-12
Status: Draft v3 (cover page added, §一+§二 merged)

---

## 一、导言

**本文探索**:2026 年「Agent 开发」这个词在中文互联网上至少在指三种完全不同的产品形态。本文把这三类分清楚——给每一类一个定义、几个主流代表、一段历史脉络。

**核心结论**:

1. **三种 Agent 开发**:**Coding Agent**(你直接用的,如 Claude Code、Manus)、**Agent 编排平台**(给业务人员 UI 拖节点,如扣子、Dify)、**Agent 代码框架**(给开发者写库,如 LangChain、Mastra)。它们服务不同的用户、走过不同的历史、用完全不同的技术栈
2. **Coding Agent** 在 2024 年从 demo 走向成熟,2025-2026 进入云端 sandbox
3. **编排平台**不是技术落后,而是用确定性 workflow 换业务可控性,2026 年依然蓬勃生长
4. **代码框架**演化最激烈:从 2022 年的纯 ReAct,到 2023-2024 年的 multi-agent 狂潮,再到 2025 年的全面回归 single-agent
5. **2025 年是关键转折**:Cognition 的《Don't Build Multi-Agents》和 Berkeley + Cornell 的 MAST 论文让业界共识反转为「multi-agent 拆得越细错得越多」。LangChain 1.0(2025-10)连 `create_react_agent` 都 deprecated 了——**ReAct 赢麻了,连名字都不用挂了**

---

## 二、为什么「Agent」这个词在 2026 年这么乱

「Agent」这个词在 2026 年的中文互联网上至少指三种完全不同的产品形态:

- **Coding Agent** —— 终端用户直接使用的产品(Claude Code、Cursor、Manus 这些),LLM 自己决定每一步
- **Agent 编排平台** —— 给业务人员的可视化低代码工具(扣子、Dify 这些),非开发者在 UI 里拖节点
- **Agent 代码框架** —— 给开发者的库(LangChain、LangGraph、Mastra 这些),用 Python/TS 写后端

它们服务的用户、技术栈、解决的问题完全不同——但都被叫做「Agent 开发」。**这是 2026 年「Agent」这个词最大的问题**:三个完全不同的东西挤在同一个名词下,所有人都觉得自己说的才是「真 Agent」。

---

要理解这个混乱怎么来的,先看一眼这个词的历史。

「Agent」不是 LLM 时代发明的。Russell & Norvig 那本经典 AI 教材给的定义非常宽泛:**一个能感知环境、做决策、采取行动的系统**。温度计不算 agent(它不做决策),AlphaGo 算 agent(它决策)。这个定义 70 年代就有,一直用得挺好。

问题是 2022 年 LLM 爆发之后,这个词被重新拉出来用在 LLM 系统上——然后**彻底失控**。三个原因:

**名词跑得快,实现跟不上。** 2023 年 AutoGPT 红起来的时候很多人第一次听到「AI Agent」,默认 Agent = 能自动干活的 AI。但这个朴素理解一年内就被业界拉扯到要覆盖十种不同的东西——从 IDE 改一行代码,到云端跑一晚上做数据分析,到客服流程回答 FAQ,全都叫 Agent。

**「Agent」听起来比别的词都潮。** 公司说「我们有一个 LLM 工作流」投资人没感觉,说「我们有一个 AI Agent」投资人立刻点头。词被滥用是因为它有溢价。

**不同圈子对它的理解完全不同。** 学术圈侧重「自主决策」,产品圈侧重「能替用户干活」,开发者圈侧重「框架抽象出 LLM + 工具的循环」。三个圈子用同一个词指不同的东西。

---

我不尝试给 Agent 下权威定义——给词下定义是学术界的事,他们也没搞定。这篇博客做的是按**产品形态**分类(就是上面那三类)。下面一类一类讲,每类按历史顺序列主流代表。

---

## 三、第一类:Coding Agent

第一类最容易理解,因为它最像一个**终端产品**——装上就能用,像装一个命令行工具或者 IDE 插件。

**定义**:终端用户直接使用的 Agent 产品。你给它一个模糊目标(「把这个 bug 修了」、「帮我调研最近的 AI 论文」),它自己决定怎么做——读哪个文件、调哪个工具、试哪条路径。全程 LLM 自己决定每一步。

**和其他两类的差别**:你不用它「搭」什么。你开盒即用。它的用户是你自己(或者任何想用它的终端用户),不是别的开发者。

Coding Agent 这一档的历史其实比另外两类都有戏剧性——它经历了「论文很火但没人用 → 爆款来了但全是 demo → 沉寂一年半 → 2024 底突然成为日常工具」的完整弧线。

### 2022-10:ReAct 论文诞生,但没人真用

一切的起点是一篇论文:Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models*,arxiv:2210.03629,**2022 年 10 月 6 日发布**。核心思路极简:让 LLM 按「思考 → 行动 → 观察」的循环跑,每一步自己决定下一步做什么。

同一个月月底——晚了整整 18 天——**LangChain 第一个版本发布**。这是学术到产品少见的「同月转化」速度。作者 Harrison Chase 当时正在读 ReAct 论文,觉得这套循环应该被包装成一个开发者可以直接用的库,然后一周之内写完发到 GitHub。

但 **2022 年底实际没人真用 ReAct**。原因很现实:GPT-3.5-turbo 的 API 到 2023 年 3 月才开放,论文是有了,你想试也试不了。舞台都还没搭起来。

真正的爆发要等半年。

### 2023 年 3-4 月:AutoGPT vs BabyAGI 的双线爆发

这一段是整个博客我最想重点讲的——因为**大部分人对这段历史的记忆其实是错的**。

2023 年 3 月 14 日,GPT-4 发布。业界立刻意识到「LLM 自己干活」这件事好像可以干了。两周内,两个项目前后脚诞生,都成了当时的现象级爆款:

**AutoGPT**(2023-03-30)。作者 Significant Gravitas。它把 LLM 的输出结构化成一个 JSON,包含 `text / reasoning / plan / criticism / speak + command` 六个字段,每次让模型填一次,然后执行 command,再把结果喂回去循环。严格说它不是经典的 Thought/Action/Observation 格式,但行为上是 **ReAct 风格**的。

**BabyAGI**(2023-04-01)。作者 Yohei Nakajima。它的架构**完全不是 ReAct**——它有三个子 agent:一个是 `execution`(执行任务),一个是 `task_creation`(根据执行结果生成新任务),一个是 `prioritization`(对任务队列排序)。再加上一个用 Pinecone 存的向量记忆。**它是一个 Plan-and-Execute 架构**,先拆任务再执行。

**前后 3 天,两个爆款,两种完全不同的架构。**

这是个反直觉的事实:大部分人现在还以为 2023 年的 Agent 狂潮是「一种东西」爆发起来的。**其实不是**。它从一开始就是两条路线并行:一条是让 LLM 在一个大循环里自决每一步(AutoGPT 派),另一条是先让 LLM 把任务拆成 todo list 再一步步执行(BabyAGI 派)。这两条路线后来各自发展出了一整套框架生态——AutoGPT 派最终赢了,BabyAGI 派在 2025 年之后基本消失。但那是后话。

两者当时的 reliability 都极差。你让 AutoGPT 帮你「调研某个话题」,它很可能跑 15 步都在重复同一个 Google 搜索,或者跑着跑着忘了自己要干什么。狂热期只持续到 2023 年中就冷却了。大家开始意识到一件事:**光有框架不够,模型也要够强**。

(小插曲:BabyAGI 原 repo 在 2024 年 9 月被作者归档,改名为 functionz。历史代码现在还在 `yoheinakajima/babyagi_archive` 里。Plan-and-Execute 派在开源世界里的第一个爆款,就这么默默退出了。)

### 2024-03:Devin 发布,引发巨大争议

2024 年 3 月,Cognition AI 发布了 **Devin**,官宣「世界上第一个 AI software engineer」。整个发布 demo 在 Twitter 上炸了——Devin 接一个 bug 任务,自己打开浏览器查文档、写代码、跑测试、跟团队汇报。看起来像魔法。

然后 YouTuber **Internet of Bugs** 做了一期硬核拆解视频,逐帧分析 Devin 发布视频里的每一个步骤,证明其中很多是 cherry-pick 的,部分任务需要大量人工干预,现实能力远不如 demo 展示的那样。Devin 是不是「过度宣传」,在 2024 年是一个公开争议。

但争议归争议,**Devin 做成了一件事**:它重新定义了「Coding Agent」这个产品形态应该长什么样。在 Devin 之前,AutoGPT 是一个你下载到本地跑的脚本,几乎没人觉得它是个产品;Devin 之后,大家意识到 Coding Agent 应该是一个**长时间运行、能自己探索、有持久记忆、能跟人对话的产品**。

还有一个伏笔:**两年后,同一家公司会发一篇文章自我否定**,叫 *Don't Build Multi-Agents*。我们放到 §六 讲。

### 2024 年底 - 2025:Cursor、Claude Code、Codex 的成熟

Devin 之后的 18 个月,是 Coding Agent 从「demo 级」真正走向「生产力工具」的阶段。

- **Cursor** 把 Agent 能力做进 IDE。开发者在编辑器里直接跟 Agent 说「把这个函数重构成 async」「帮我把这个 bug 修了」,Agent 在代码库里自己搜索、改文件、给出 diff。Cursor 在 2024 年完成了一次现象级增长——年底 ARR 已经做到 $400M,是当时所有 AI coding 工具里收入排第二的(仅次于 GitHub Copilot),也是增长最快的独立 AI 编码产品
- **Claude Code CLI** 2024 年底由 Anthropic 发布。它是一个**命令行工具**,不是 IDE 插件,也不是网页产品。你在终端里跟 Claude 说话,它可以读取你本地任何文件、执行任何命令。这种「纯 shell + 文件系统」的极简架构后来成了 Coding Agent 的事实标准
- **OpenAI Codex CLI** 紧随其后发布,形态和 Claude Code 几乎一样

架构上,这一批产品有一个共同点:**文件系统 + shell tools + LLM 大循环**。就这三样。LLM 在一个无限循环里思考、调 shell、读文件、改文件、再思考。没有多 agent,没有 workflow,没有复杂的 Plan 产物——极简,但能跑。

**这是 Coding Agent 的范式胜利**。模型能力够了之后,架构反而回到了最朴素的循环。

### 2025-2026:Manus / Claude Co-worker / GenSpark

最新一波是把 Coding Agent 的架构**从本地 CLI 扩展到云端 sandbox**。

- **Manus**(2025)是国内最出圈的云端任务 Agent。你给它一个任务,它在自己的云端虚拟机里打开浏览器、写代码、画图表、做 PPT,做完发你一个链接
- **Claude Co-worker / Claude Projects** 是 Anthropic 在 2025-2026 的对应产品
- **GenSpark** 是另一个同类竞品

架构跟本地 Coding Agent **完全一致**——只是把 sandbox 从你的本地机器挪到了云端。扩展的不是技术路线,而是应用范围:从「帮程序员写代码」扩展到「帮任何人完成需要几十步操作的长任务」。

到 2026 年中,Coding Agent 这一档已经相当成熟了。你问一个技术人「你用过 AI 写代码吗」,大概率他会说「用过」——他用的那个工具,大概率就是这一档里的某个。

---

## 四、第二类:Agent 编排平台

第二类是**你不直接用、而是用它搭出别的东西给别人用**的产品。

**定义**:非工程师在 UI 里画流程,搭出一个带 LLM 的业务 bot 或应用。

**和其他两类的差别**:它不是终端产品(你不直接用它做事),也不是代码库(你不写代码)。它是一个**可视化的低代码搭建器**——目标用户是 PM、运营、客服主管、内容创作者,不是开发者。

**为什么会有这一类**:因为企业场景需要的不是「LLM 自己决定怎么干」,而是**可审计、可回滚、成本可预估的确定性**。完全让 LLM 自决,在客服、订单、合规这类场景里不够可控——跑偏了谁负责?成本超了谁兜底?于是业界发展出了一个折中方案:**人画好流程,LLM 只在某些节点里做小范围决策**。把自主决策的权力关进确定性 workflow 的格子里。

这一档我只讲两个代表——国内一个、开源一个;C 端一个、开发者一个。其他产品大同小异,后面一段带过。

### 扣子 Coze(2024 年 2 月 1 日国内版上线)

扣子是字节跳动出品,目前**国内大众认知度最高的 agent 编排平台**。它的发布时间稍晚于海外版 coze.com(后者 2023 年 12 月底在海外 beta),国内版 coze.cn 是 2024 年 2 月 1 日正式上线。

扣子的目标用户说白了就是「**不写代码的产品经理、运营、客服主管、内容创作者**」。你打开扣子,拖几个节点——知识库检索、条件分支、LLM 调用、发送消息——配几个 prompt,就能搭一个能用的 bot。然后一键发布到微信、QQ、飞书、抖音。它的杀手锏是**和字节生态的深度绑定**:搭的 bot 可以直接挂到微信公众号、QQ 群、飞书工作区,甚至嵌入抖音小程序。这是国内其他厂商学不来的。

但扣子的产品形态本身比它的传播能力更值得说一句。**扣子的解法是用工作流编排去绕开 LLM 自主规划这个问题**——让产品经理把流程画出来,LLM 只在节点内做小范围决策。

知乎有一篇拆解扣子的分析文章观察得很准:

> 「规划是最诱人的部分,是机器人从被动到主动的分界线,但也是最难以工程化的。工作流编排是一种方式,但这与专家知识和大模型的开放性有着本质的冲突。」
>
> ——《手拆大模型平台-字节扣子 Coze-agent 工程化能力是否达标》(知乎,第三方分析)

这不是扣子官方的承认,是第三方分析者抓到的一个矛盾——**但它说出了整个第二类存在的真相**:不是技术上落后,而是故意保守,**用「放弃 LLM 的开放性」去换「业务可以用」**。这是企业落地 AI 应用的真实约束,不是技术路线的优劣。

### Dify(2023 年 4 月 GitHub 创建)

Dify 是扣子的**开源对照**。如果说扣子是给非开发者的 C 端产品,Dify 就是给开发者和 AI 创业者的自部署版本。

一些数据:
- GitHub 上 **137k+ stars**(2026 年 4 月实测),在 agent 相关开源项目里排前三
- 完全开源,可以 Docker Compose 自部署
- 2024 年后,它的执行引擎拆出来成了一个独立包叫 **graphon**,是一个 queue 驱动的 worker pool 架构。这件事本身挺硬核的——它说明 Dify 团队认真对待「执行引擎可以独立演化」这件事

Dify 和扣子对比,**产品哲学是同一个**(workflow + LLM 节点 + 可视化 UI),但受众完全不同。扣子是给「不想学技术的小白」,Dify 是给「想快速搭原型并且想自部署的开发者/创业者」。扣子胜在生态和易用性,Dify 胜在开源和灵活性。

### 其他同类产品

国内几乎所有大厂都有自己的版本:**百度的文心智能体平台、腾讯元器、阿里云百炼、FastGPT**……产品名经常改(百度这个产品 2023 年 9 月到 2024 年 4 月就改过两次名字),但本质都是「workflow + LLM 节点 + 非工程师 UI」这一套范式,没有本质差异。不一一列举。

### 这一档的共同特征

- 核心抽象:**节点 + 连线 + LLM 作为一种节点类型**
- 都内置知识库、插件、工具调用
- 都面向非工程师
- **2026 年依然蓬勃生长**——因为它们解决的是「业务落地的确定性需求」,这个需求和技术先进性无关。只要企业还要做「可控的 AI 应用」,这一档就会一直在

---

## 五、第三类:Agent 代码框架

最后这一档是给**开发者**的。你写代码——Python 或 TypeScript——import 一个库,用它提供的 API 搭一个 agent,然后把这个 agent 作为你产品后端的一部分跑起来。

**定义**:开发者用代码写 Agent 的库和框架。

**和第二类的差别**:第二类是 UI 低代码产品,第三类是开发者写代码。用户从「产品经理」变成了「后端工程师」。

**这一档的独特性**:**演化最快最激烈**的一档。从 2022 年的 LangChain 到 2025 年的 LangChain 1.0,只用了三年,中间经历了 multi-agent 爆发、role-based 框架狂潮、然后是戏剧性的收敛。这段历史比前两类都精彩得多——按时间顺序走一遍。

### 2022-10:LangChain v1 —— 一切的起点

LangChain 的第一个版本和 ReAct 论文同月出生(§三 提过),早期的它只是一件很单纯的事:**一个把 ReAct 循环变成 Python API 的库**。

抽象叫 `AgentExecutor` + **九种 `AgentType`**:`ZeroShotReact`、`StructuredChatReact`、`OpenAIFunctions`、`ReactDocstore`、`SelfAskWithSearch`……每一种都是围绕「ReAct 的某种 prompt 格式」展开的不同实现。本质上**全部都是 ReAct**,只是在 prompt 工程上做不同的包装。

### 2023-05:LangChain 官方承认「单 ReAct 不够用」

AutoGPT 和 BabyAGI 爆发之后一个月——**2023 年 5 月 10 日**——LangChain 发了一篇官方博客,推出了一个叫 `plan_and_execute` 的新包。这篇博客里说了一段业界第一次**官方**承认 ReAct 局限性的话:

> 「User objectives are becoming more complex... Developers and organizations are starting to rely on agents in production... The need for increasingly complex abilities and increased reliability are causing issues when working with most language models.」
>
> —— LangChain, *"Plan-and-Execute Agents"*, 2023-05-10

翻译一下就是:任务一复杂、想让 LLM 在生产环境可靠地完成它,prompt 就会被塞爆(更多工具说明 + 更多历史上下文),ReAct 循环撑不住。

这是一个关键的时间点。从这里开始,代码框架开始往**更复杂**的方向走——拆任务、多 agent、显式 plan……整个行业觉得「光靠 ReAct 不够,需要更多结构」。后面两年的 multi-agent 狂潮,在这一刻就埋下了种子。

### 2023-08 / 09:AutoGen + MetaGPT —— 学术派的 multi-agent 爆发

**MetaGPT**(论文 2023 年 8 月)来自国内 AI 公司 **DeepWisdom**(深度赋智),第一作者 Sirui Hong,通讯作者是 DeepWisdom CEO Chenglin Wu,合作者覆盖 KAUST、CUHK、CMU 等多所机构——作者列表里甚至挂着 LSTM 发明人 **Jürgen Schmidhuber** 的名字。它把「多个 agent 扮演不同角色协作」做成了一个框架,最有名的 demo 是让多个 agent 扮演「产品经理、架构师、工程师、QA」协作完成一个软件项目。

**AutoGen**(论文 2023 年 8 月、开源 2023 年 9 月)来自 Microsoft Research,从 MSR 的 FLAML 项目分出来,思路类似:多个 agent 通过对话协作完成任务。

这两个项目的共同点是「**agent 不再是一个 LLM 在循环,而是多个 LLM 在互相对话**」。它们塑造了整个 2024 年的业界想象——「AI team 协作比单 agent 强」。

### 2023-11:CrewAI —— role-based 多 agent 的旗手

**2023 年 11 月 14 日**,CrewAI 开源。它是把 MetaGPT/AutoGen 的多 agent 思路**彻底产品化**的一个框架。你在 CrewAI 里定义一个 `Crew`,里面放几个 `Agent`,每个 Agent 给一个 role(CEO / CTO / Analyst / Researcher),定义它能用的工具,然后给整个 Crew 一个任务——让它们「协作」完成。

这个「AI team」的叙事非常吸引人。2024 年 CrewAI 成了 multi-agent 圈最火的框架,一度被视为未来方向。它在 2024 年 10 月还完成了商业化。

**伏笔**:2025 年 Devin 的母公司 Cognition 发了 *Don't Build Multi-Agents*,加上 Berkeley + Cornell 发了 MAST 论文,role-based 多 agent 的范式被**同时**两个重量级证据打脸。之后 CrewAI 还在,但「role 全家桶」的原教旨主义不再是业界共识了。我们留到 §六 展开讲。

### 2024-01:LangGraph —— 图式 agent 的底层引擎

**2024 年 1 月 22 日**,LangChain 推出 **LangGraph**。这是一个底层更硬的库——它把 agent 抽象成一个 **StateGraph**,用**节点 + 条件边**来表达控制流。底层用了一个叫 **Pregel** 的图计算引擎(名字和思路都来自 Google 2010 年的 Pregel 论文),**原生支持 cycles**——这是传统 DAG 编排工具做不到的。

LangGraph 早期主打的卖点还是 multi-agent subgraph 组合。但它的真正价值要等到 2025 年才显现——到那时候,它成了整个 LangChain 生态的**底层执行引擎**。这一点等下讲 LangChain 1.0 的时候会回头说。

### 2024-2025:新一代代码框架登场

这段时间涌现了一批新框架:**Mastra**(TypeScript 生态)、**Google ADK**(Google 官方 Python)、**AgentScope**(阿里通义实验室开源,23k+ stars)、**Pydantic AI**(Pydantic 团队出品,类型最讲究)、**Vercel AI SDK** 等等。

它们的共同特征是:**外层是一个 workflow DAG(开发者定义),内层每个节点里跑一个 LLM ReAct 循环**。这个「workflow 外层 + ReAct 内层」的组合,在 2025 年成了**新一代代码框架的默认姿态**。不管它们用 TypeScript 还是 Python,底层是不是 LangGraph,这个架构已经是共识。

### 2025-10-22:LangChain 1.0 —— 一个时代的句号

**2025 年 10 月 22 日**,LangChain 发布 **1.0 版本**。它做了两件大事:

1. **把 Classic 时代的所有老 API 挪到一个叫 `langchain-classic` 的兼容包里**。9 种 AgentType、AgentExecutor、`plan_and_execute`……全部打包挪走,主包里不再有它们
2. **把 1.0 的主 API 直接建在 LangGraph 之上**。`create_agent()` 是唯一推荐入口,它返回的是一个 `CompiledStateGraph`——也就是一个 LangGraph 对象

但最值得玩味的一件事是:**连 `create_react_agent` 都被 deprecated 了**。新的入口叫 `create_agent`,没有「react」这个词。

想想这件事的分量——2022 年 10 月 LangChain 和 ReAct 论文同月诞生,作者是读完 ReAct 论文当晚就开始写 LangChain 的。**三年之后,LangChain 的终极 API 里,ReAct 这个名字已经不再被提起**。

这不是 ReAct 被抛弃了——恰恰相反,它**赢麻了**。它胜出到这种程度——所有代码框架默认都是它,以至于名字都不用再单独挂出来了。当所有 agent 都默认是 ReAct,你就不需要在 API 里把这个词写出来。

如果你问我整篇博客里最想加粗的一句话是什么,是这个:

> **「ReAct 赢麻了,连名字都不用挂了。」**

---

## 六、总结:时间线 + 定调 + 一个观察

三类讲完了。这一章做三件事:**给你一张统一的时间线表、告诉你 2026 年的业界共识是什么、最后给你一个我个人写完之后回头看的视角**。

### 6.1 跨三类的统一时间线

| 年份 | Coding Agent | 编排平台 | 代码框架 |
|---|---|---|---|
| 2022-10 | ReAct 论文 | — | LangChain v1 |
| 2023-03/04 | AutoGPT + BabyAGI | — | — |
| 2023-04 | — | Dify 开源 | — |
| 2023-05 | — | — | LangChain 官方承认「单 ReAct 不够」 |
| 2023-08 | — | — | MetaGPT 论文 |
| 2023-09 | — | — | AutoGen 开源 |
| 2023-11 | — | — | CrewAI 开源 |
| 2024-01 | — | — | LangGraph |
| 2024-02 | — | **扣子国内版上线** | — |
| 2024-03 | Devin | — | — |
| 2024 底 | Claude Code / Cursor 成熟 | — | — |
| 2024-10 | — | — | CrewAI 商业化 |
| 2024-12 | — | — | Anthropic *Building Effective Agents* |
| 2025-03 | — | — | **Berkeley + Cornell MAST 论文** |
| 2025-06 | — | — | **Cognition *Don't Build Multi-Agents*** |
| 2025-10 | — | — | **LangChain 1.0**(create_react_agent 也 deprecated) |
| 2026 | Manus / Claude Co-worker | — | — |

一张表看下来,**代码框架那一列的格子最满**——它的演化节奏确实是三类里最快的,故事也最多。

### 6.2 2025 年的转折点:Cognition + MAST 的双重证据

整个博客最关键的一个时间点是 **2025 年**。这一年发生了三件事,加起来彻底改变了代码框架的走向。

**① Anthropic *Building Effective Agents*(2024 年 12 月 19 日)**

严格说这是 2024 年底,作者 Erik Schluntz 和 Barry Zhang。这篇博客第一次给「Workflow vs Agent」做了一个**业界通用的定义**:**Workflow 是代码路径预定义的,LLM 只在某些节点决策;Agent 是 LLM 动态决定所有流程**。这个区分到现在还是业界讨论的基础词汇。

**② Berkeley + Cornell 的 MAST 论文(2025 年 3 月 17 日)**

arxiv:2503.13657,*Why Do Multi-Agent LLM Systems Fail?*。这篇论文做了一件之前没人做过的事:**系统化地量化了 multi-agent 系统的失败模式**。研究者先用 150 条标注 trace 构建了一个**包含 14 种失败模式**的 taxonomy,然后用 1600+ 条 trace 去验证。结果非常关键:**14 种失败模式里,有 8 种是「架构级」失败,不是「实现级」失败**。

「架构级」是什么意思?意思是你换更聪明的模型也救不回来——失败是因为 multi-agent 这种结构本身在制造问题。换句话说:**multi-agent 失败不是因为 GPT-4 不够聪明,而是因为你拆得太多了**。

这是一个定量证据。它把「multi-agent 不靠谱」从一种口口相传的开发者抱怨,变成了**有数据支撑的学术结论**。

**③ Cognition *Don't Build Multi-Agents*(2025 年 6 月 12 日)**

作者是 Cognition AI 的 Walden Yan——也就是 §三 提到的 Devin 那家公司。两年前发 Devin demo 把整个行业搅得天翻地覆的同一家公司,在 2025 年发了一篇文章自我否定,说**别再搭 multi-agent 系统了**。

这篇文章给了两条原则:

> Principle 1: **「Share context, share full agent traces」**(共享上下文,共享完整的 agent trace)
>
> Principle 2: **「Actions carry implicit decisions, and conflicting decisions carry bad results」**(行为里隐含着决策,而冲突的决策会带来糟糕的结果)

抽象。但 Cognition 给了一个非常具象的例子,叫 **Flappy Bird 例子**。我直接复述:

> 假设你让一个 multi-agent 系统给你做一个 Flappy Bird 克隆。Planner 把任务拆成两个 subtask:
>
> - Subtask 1:**做一个有绿水管和碰撞盒的滚动背景**
> - Subtask 2:**做一只你可以上下移动的鸟**
>
> 注意,subtask 已经写得**非常具体**了——「绿水管」几乎就是 Flappy Bird 的视觉商标。
>
> 然后两个 subagent 各自一跑:
>
> - Subagent A 误解了 subtask,做出一个 **Super Mario Bros 风格的背景**
> - Subagent B 做出来一只鸟——但**它看起来不像游戏素材**,**移动方式也完全不像 Flappy Bird 里那只**
>
> 两个 subagent 各自都"完成了任务"。最后 combiner agent 拿到这两坨完全错位的东西,只能硬着头皮把它们拼在一起。

问题不是任何一个 subagent 错了。问题是**它们之间没有共享上下文,也没有共享中间决策**。Planner 写 subtask 时心里想的是「Flappy Bird 风格」这个隐式假设——但这个假设没有显式传递给 subagent。每个 subagent 都做了自己「合理」的选择,但合在一起就是垃圾。

Walden 的原文收尾这一段是:**"Now the final agent is left with the undesirable task of combining these two miscommunications."**(现在最后那个 agent 只剩一件糟心事:把这两份误传拼在一起。)

这是 multi-agent 系统的核心失败模式:**每次 handoff 都是一次隐式假设丢失的机会**。Cognition 用了两年时间,用真实生产环境的教训得到了这个结论。

### 核心 insight

这三件事——Anthropic 的定义、MAST 的定量证据、Cognition 的实战反思——加起来给了业界一个共识:

> **「问题不是『模型变强所以不需要复杂脚手架』,而是『复杂度本身就是负资产』。**
>
> **multi-agent 的失败不是因为模型不够聪明,而是因为每次 handoff 都是一次隐式假设丢失的机会。」**

这个共识让整个代码框架的演化方向**反转**了。2023-2024 年大家都在往「更复杂、更多 agent、更结构化」的方向走;2025 年之后,大家集体往回退——退到「单 agent + 优秀的上下文压缩 + workflow 做外层组织」这个最简朴的模式。

LangChain 1.0 在 2025 年 10 月发布时把 `create_react_agent` 都 deprecated,新名叫 `create_agent`,就是这个共识在 API 层面的兑现。

### 6.3 正在兴起的 / 正在消亡的

**正在兴起**:
- 「workflow 外层 + 单 ReAct 内层」作为代码框架的默认模式
- LangGraph 成为整个 LangChain 生态的底层执行引擎
- 类型优先的框架(Pydantic AI 是代表)
- Agent-to-Agent 协议(Google A2A、AgentScope 的 A2AAgent)

**正在消亡**:
- LangChain Classic 的 9 种 AgentType
- Role-based 多 agent 原教旨主义(CrewAI 式 「CEO/CTO/Analyst」)
- 独立的 Plan-and-Execute 框架(LangChain experimental 已挪到 langchain-classic)
- 手写 ReAct while 循环
- LangGraph 的 `create_react_agent` API(2025-10 被 `create_agent` 取代)

### 6.4 一个可选视角:Multi-agent vs Single-agent

写完上面这一切之后,如果你想要一个 one-liner 来记住整段历史,我有一个粗略但好用的视角——**Multi-agent vs Single-agent**。

这个视角是这样的:

> 所有叫 Agent 的东西,粗略可以分成两端:
> - **Multi-agent 倾向**:人工把流程拆出来——UI 拖拽、代码编排、role 分工。**第二类编排平台**整体属于这一端
> - **Single-agent 倾向**:一个 LLM 在循环里自决一切。**第一类 Coding Agent** 整体属于这一端
> - **第三类代码框架**:历史上从 Single(早期 LangChain)→ 实验 Multi(CrewAI / AutoGen / plan_and_execute)→ 回归 Single(LangChain 1.0 `create_agent`)。它是这个视角下唯一一个**两端都待过**的类别

这个视角不是严格分类——边界经常模糊(比如 Mastra 外层 workflow DAG 像 Multi,内层每个节点跑单 ReAct 像 Single,你很难硬把它归到一边)。但即使不强求精确,这个视角能帮你看清楚一个**清楚的趋势**:

- 2022-2024 年,代码框架明显倾向于「拆」——不管是 role-based 多 agent、Plan-and-Execute、还是 multi-agent workflow,共同点都是「人工把任务分解」
- 2025 年起,Cognition 和 MAST 让业界承认「拆多了反而错得多」,代码框架收敛到「单 agent ReAct + workflow 外层」
- Coding Agent 和编排平台从头到尾都没在这场博弈里——它们各自服务不同的用户和需求,和技术路线博弈无关

一句话总结:**Coding Agent 是让 LLM 自己干,编排平台是给业务人员用来拆流程,代码框架是开发者在这两端之间做选择的工具箱——而 2025 年之后代码框架这个钟摆,从「拆」的方向摆回了「不拆」的方向**。

---

## 七、一个简单的诊断问题

回到博客开头那三个产品形态:

- 那个**用扣子拖节点搭 HR 咨询 bot 的产品经理** → 第二类:Agent 编排平台。她的产品形态是「人画 workflow + LLM 当节点」。她不需要懂代码,她需要懂业务流程
- 那个**用 Claude Code 修 bug 的开发者** → 第一类:Coding Agent。他不在「搭」什么,他直接用一个产品。他是 Coding Agent 的终端用户
- 那个**用 LangGraph 写产品后端的工程师** → 第三类:Agent 代码框架。他在用一个 Python 库写后端,他既是开发者也是用户的供应商

下次再有人告诉你「我在做 Agent」,你只需要问一个问题:

> **你的产品形态是什么?**

- 终端用户直接用 → 第一类(Coding Agent,Claude Code、Manus 这些)
- 非开发者在 UI 里拖节点 → 第二类(编排平台,扣子、Dify 这些)
- 开发者在代码里写 → 第三类(代码框架,LangChain、LangGraph、Mastra 这些)

这三类各有各的用户、演化节奏、主流代表。混着说只会互相听不懂。

---

**最后留一个续集钩子。**

这篇博客做的是「盘点」——把 2026 年市面上的 Agent 开发分类讲清楚。但有一个**灰色地带**这次没展开:某些生产场景里,Coding Agent 那种 LLM 完全自决的形态给不了**成本预估、审计追溯、合规控制**;编排平台的可视化 workflow 又太僵——加一个新业务 case 就得重画一张图;代码框架的默认 single-agent ReAct 也不完全对——因为它不给你**结构化的 Plan 产物**。

人们在这个灰色地带做了一些**介于三类之间**的东西。它们既不是 Coding Agent,也不是编排平台,也不完全是主流代码框架——它们是另外一种东西。

**下一篇**我聊这个灰色地带,以及为什么我自己做的生产 Agent 系统,就在这个地带。

---

*完*






