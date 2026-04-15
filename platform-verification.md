# Platform Verification Report

Last Updated: 2026-04-11

Verification of product names, release dates, positioning, and supporting citations for the blog at `/Users/linguanguo/dev/llm-agent-research/blog.md` — specifically the second category "Agent 编排平台 / Agent orchestration platforms" and related technical claims.

Legend: ✅ verified as stated · ⚠️ partially correct, needs adjustment · ❌ wrong, must fix

---

## 1. 百度系产品 (The Most Important Question)

**User asked**: 百度的那个产品到底叫什么？是 AppBuilder？灵境矩阵？妙搭？插件库？

**Short answer**: 百度有**三个不同的产品**，对应三个不同的用户群和不同的时间线。它们不是同一个东西，而且名字确实容易混。

### 1.1 灵境矩阵 (Lingjing Matrix) — 已更名退役

| 字段 | 值 |
|---|---|
| 首次发布 | 2023-09 (2023百度联盟大会发布，9月1日开始内测) |
| 原定位 | 文心一言**插件生态平台** — 给开发者做 ChatGPT-Plugin 风格的插件 |
| 目标用户 | 第三方插件开发者 |
| 最终命运 | 2023-12-18 升级为"文心大模型智能体平台"；2024-04-16 再次更名为 **文心智能体平台 (AgentBuilder)** |
| 状态 | 名字已退役，能力被合并进 AgentBuilder |

**验证**:
- [百度发布文心一言插件生态平台"灵境矩阵" - OSCHINA](https://www.oschina.net/news/257981)
- [百度推出"灵境矩阵"，为开发者打开通往大模型应用的大门 - 知乎](https://zhuanlan.zhihu.com/p/656752046)

**判决**: 用户记忆"灵境矩阵 / 插件库"是**对的，但过时了**。2024-04 后这个名字已经不存在，继任者是文心智能体平台。

### 1.2 文心智能体平台 / AgentBuilder — C 端创作者产品 ✅

| 字段 | 值 |
|---|---|
| 产品名 | **文心智能体平台 (AgentBuilder)** |
| 正式发布 | 2024-04-16 (灵境矩阵改名升级) |
| 官网 | https://agents.baidu.com/ |
| 口号 | "想象即现实" |
| 目标用户 | **内容创作者 / 运营 / 中小企业** — 做 C 端智能体、发到百度搜索和文心一言分发 |
| 定位 | 文心一言生态的 Agent Store + 低代码编排平台，类似**扣子 Coze** |
| 2024-07 状态 | 免费开放文心大模型 4.0；3万+智能体、5万+开发者入驻 |

**验证**:
- [文心智能体平台AgentBuilder 官网](https://agents.baidu.com/)
- [百度文心智能体平台全解析](https://cloud.baidu.com/article/3408309)
- [智能体成商业经营"标配"，文心智能体平台汇聚超5万开发者](https://www.stcn.com/article/detail/1177564.html)

### 1.3 千帆 AppBuilder / 百度智能云千帆 — B 端企业产品 ✅

| 字段 | 值 |
|---|---|
| 产品名 | **千帆 AppBuilder**（隶属"百度智能云千帆大模型平台"） |
| 首次发布 | 2023-10 (千帆 AppBuilder 组件)；**2023-12-20 百度云智大会正式开放**为 AI 原生应用开发工作台 |
| 官网 | https://cloud.baidu.com/product-s/qianfan_home |
| GitHub SDK | [baidubce/app-builder](https://github.com/baidubce/app-builder) |
| 目标用户 | **企业开发者 / 企业 IT** |
| 定位 | **企业级** Agent 开发平台 — 强调企业级 RAG、企业级 Agent、企业级集成、企业级安全部署 |
| 与 Coze/Dify 的区别 | 一样是 workflow 编排，但面向 B 端私有化部署、企业集成、合规 |

**验证**:
- [千帆大模型平台官网](https://cloud.baidu.com/product-s/qianfan_home)
- [百度智能云千帆 AppBuilder 大模型应用开发解读](https://www.53ai.com/news/LargeLanguageModel/2024110893241.html)
- [baidubce/app-builder GitHub](https://github.com/baidubce/app-builder)

### 1.4 秒哒 (MiaoDa) — 零代码应用生成，不是 Agent 编排 ⚠️

| 字段 | 值 |
|---|---|
| 产品名 | **秒哒 (MiaoDa)** |
| 首次公布 | 2024-11-12 (百度世界2024大会，李彦宏发布) |
| 正式上线 | 2025-03-24 |
| 官网 | https://www.miaoda.cn/ |
| 目标用户 | **完全非开发者** — "一句话生成应用" |
| 定位 | **零代码 AI 应用生成平台** — 多 agent 协作（产品经理/UI/开发工程师 agent 分工）生成 H5、网站、小程序 |
| 和 Agent 编排平台的关系 | **不是同一类** — MiaoDa 是面向终端用户的"AI 生成应用"产品，用多 agent 做 codegen，而不是给人去"编排 agent"。类别上更接近 Lovable / v0 / Bolt.new，不是 Coze / Dify |

**验证**:
- [百度发布文心iRAG和无代码"秒哒"两大AI技术 - 新华网](http://www.news.cn/tech/20241112/041eaddb008049fdb7a9b93d7713c4f1/c.html)
- [秒哒 MIAODA 百度智能云官方文档](https://cloud.baidu.com/doc/MIAODA/s/Sm88db6er)

### 1.5 判决：博客应该怎么写百度

博客第二类"Agent 编排平台"里，百度代表的**最佳选择**是：

> **文心智能体平台 (AgentBuilder)** — 2024-04 正式发布，面向 C 端创作者/运营，对标**扣子 Coze**。历史沿革：灵境矩阵 (2023-09) → 文心大模型智能体平台 (2023-12) → 文心智能体平台 AgentBuilder (2024-04)。

**如果博客要强调 B 端 / 企业视角**，应该同时或改用：

> **千帆 AppBuilder** — 2023-12-20 开放，企业级 Agent 开发工作台，对标 Dify 企业版。

**不要用**的名字：
- ❌ "百度 AppBuilder" —— 歧义，应该说 "千帆 AppBuilder" 或 "百度智能云千帆 AppBuilder"
- ❌ "灵境矩阵" —— 2024-04 已退役，写进 2026 的博客会显得信息过期
- ❌ "插件库" —— 不是正式产品名，是灵境矩阵的功能描述
- ❌ "妙搭" —— 用户记错了，百度的是 **"秒哒"（MiaoDa）**，而且它不是 Agent 编排平台，是零代码应用生成器
- ❌ 把"秒哒"塞进第二类 —— 类别不对

**推荐写法（给博客起草用）**:

> 百度同时运营两条产品线：面向 C 端创作者的 **文心智能体平台 (AgentBuilder)**（2024-04 上线，前身是 2023-09 的灵境矩阵插件平台），对标扣子；和面向企业开发者的 **千帆 AppBuilder**（2023-12 正式开放），对标 Dify 企业版。两者都是 workflow 风格的低代码编排。

---

## 2. 扣子 Coze (字节)

| 断言 | 验证 | 状态 |
|---|---|---|
| 国内版 coze.cn 发布日期 **2024-02-01** | ✅ 多个来源确认：字节跳动去年 12 月在海外推出的 Coze，"已于 2 月 1 日正式在国内上线，中文名'扣子'" | ✅ |
| 海外版 coze.com 发布时间 **2023-12** | ✅ "字节跳动去年 12 月在海外推出" — 对应 2023-12 | ✅ |
| 仍在运营 (2026-04) | ✅ coze.cn 官网活跃，定位已扩展为"AI 办公助手一站式平台" | ✅ |

**验证**:
- [字节跳动推出"一站式AI开发平台"Coze 扣子 - 腾讯新闻 (2024-02-03)](https://news.qq.com/rain/a/20240203A07KG700)
- [扣子 coze.cn 官网](https://www.coze.cn/)

**博客可以保留**: "**扣子 Coze（2024-02-01 国内版上线，海外版 coze.com 2023-12 先发）**"

---

## 3. Dify (langgenius)

| 断言 | 验证 | 状态 |
|---|---|---|
| 首次 commit 2023-03 | ⚠️ GitHub repo 创建日期是 **2023-04-12**。项目本身可能 2023 年 3 月就在内部起步，但公开 repo 和发布都是 4 月 | ⚠️ |
| 2023-04 发布 | ✅ GitHub repo 创建 2023-04-12 | ✅ |
| GitHub stars 137k+ | ✅ 实测 GitHub API 返回 **137,260 stars** (查询时间 2026-04-11) | ✅ |
| 开源 | ✅ GitHub 公开 repo | ✅ |
| "Agent 编排平台最知名开源产品" | ✅ 137k stars 是该品类第一，远超 FastGPT (27.7k) 等同类 | ✅ |

**验证**:
- [langgenius/dify GitHub](https://github.com/langgenius/dify) (137,260 stars, created 2023-04-12, 最近 push 2026-04-11)

**博客建议修改**:
- "2023-03 首次 commit" → "**2023-04 开源发布** (GitHub repo 创建于 2023-04-12)"
- "137k+ stars" → "**137k+ stars** (2026-04)" ✅ 可保留

---

## 4. 腾讯元器

| 断言 | 验证 | 状态 |
|---|---|---|
| 真实存在 | ✅ | ✅ |
| 准确产品名 | ✅ **腾讯元器** (Tencent Yuanqi) — 正式名；不是"腾讯智能体" | ✅ |
| 发布时间 | **2024-05-17** 腾讯云生成式 AI 产业应用峰会上线 | ✅ |
| 目标用户 | C 端 = 个人开发者/创作者；B 端 = 企业（"腾讯云智能体开发平台"） | ✅ |
| 和腾讯元宝的关系 | **不是同一个** —— 腾讯元宝是 2024-05-30 发布的 **C 端聊天 App**（类 ChatGPT），元器是 agent 创作平台 | - |
| 是否运营中 | ✅ 仍在运营 | ✅ |
| 基于模型 | 腾讯混元大模型 | ✅ |

**验证**:
- [AI智能体创作与分发平台"腾讯元器"上线 - 央广网 (2024-05-17)](https://tech.cnr.cn/techph/20240517/t20240517_526708228.shtml)
- [腾讯元器：面向未来的一站式 AI 智能体创作与分发平台](https://blog.csdn.net/i042416/article/details/147543070)

**博客可以写**: "**腾讯元器**（2024-05-17 上线，基于混元大模型的一站式智能体创作与分发平台，对标扣子，主打微信/QQ 分发通道）"

**注意**: 不要把元器和元宝写混了。

---

## 5. 火山引擎 AgentKit (字节)

| 断言 | 验证 | 状态 |
|---|---|---|
| 真实存在 | ✅ 火山引擎官网有专门产品页 | ✅ |
| 准确产品名 | ✅ **火山引擎 AgentKit** (官方页面：`volcengine.com/product/agentkit`) | ✅ |
| 与扣子 Coze 的关系 | **互补**，不重叠：扣子 Coze 是 C 端可视化 bot 平台；AgentKit 是 B 端云原生 agent 开发套件（安全、运行时、沙箱、网关、记忆、监控、评测、护栏 **8 大模块**）。两者都是字节，但走不同客户群 | ✅ |
| 是否是"Agent 编排平台" | ⚠️ **不完全是** —— AgentKit 更像"云原生 agent 基础设施套件"，不是拖拽编排画布。它给客户提供的是运行时/网关/记忆/评测这些**运行底座**，需要客户自己写代码集成 | ⚠️ |

**验证**:
- [AgentKit 火山引擎官方产品页](https://www.volcengine.com/product/agentkit)
- [AI 云原生 Agent 套件 - 火山引擎](https://www.volcengine.com/solutions/ai-cloud-native-agentkit)
- [火山引擎AgentKit升级，8大模块让企业Agent快速投产 - 知乎](https://zhuanlan.zhihu.com/p/1986764984575886310)

**博客建议修改**:
- 不要把 AgentKit 和扣子/Dify 放在同一类 "Agent 编排平台" 里。它属于**基础设施层 (infrastructure)**，定位类似 AWS Bedrock AgentCore。
- 写法建议：*"火山引擎 AgentKit（字节云原生 agent 基础设施，8 大模块：安全/运行时/沙箱/网关/记忆/监控/评测/护栏，不是可视化编排，而是给 B 端开发者用 SDK 集成的运行底座）"*

---

## 6. 阿里云百炼 / AgentScope / "ModelStudio-ADK"

**这是一团乱麻，之前的研究文档 `domestic-platforms.research.md` 已经做过一次纠错，结论在那里。**

| 断言 | 实际 | 状态 |
|---|---|---|
| "阿里百炼 Visual Studio" | ❌ **不是准确产品名**。官网叫 **"大模型服务平台百炼" / "Model Studio"**，功能模块叫"智能体应用"、"工作流"、"智能体编排" | ❌ |
| "ModelStudio-ADK" | ❌ **媒体造词**，不是阿里官方产品名（36kr、InfoQ 类比 Google ADK 造出来的） | ❌ |
| "AgentScope" | ✅ 真实存在，是阿里通义实验室的**开源代码框架**：[agentscope-ai/agentscope](https://github.com/agentscope-ai/agentscope) (23.4k stars)，**ReAct 架构**（不是 P&E） | ✅ |

**三者的真实关系**:
- **百炼 (Bailian / Model Studio)** = 阿里云的低代码 Agent 编排平台，对应 Coze/Dify 这一层（**第二类**）
- **AgentScope** = 阿里通义实验室的开源代码框架，对应 LangGraph/Mastra/Google ADK 这一层（**第三类**）
- **"ModelStudio-ADK"** = 不存在的产品名，**博客里必须删掉**

**验证**:
- [阿里云百炼官网 (bailian.aliyun.com)](https://www.aliyun.com/product/bailian)
- [智能体应用 - 大模型服务平台百炼 - 阿里云](https://www.alibabacloud.com/help/zh/model-studio/single-agent-application)
- 已有研究文档：`/Users/linguanguo/dev/llm-agent-research/domestic-platforms.research.md` (第 161-214 行详细纠错)

**博客建议修改**:
- 第二类里写：**"阿里云百炼 (Bailian / Model Studio)"** —— 阿里云的 Agent 低代码编排平台，支持智能体应用、工作流、智能体编排三种模式
- 第三类里写：**"AgentScope (阿里通义实验室, 23.4k stars)"** —— ReAct 架构的开源代码框架
- ❌ 删掉 "百炼 Visual Studio" 这个不准确的名字
- ❌ 删掉 "ModelStudio-ADK" 这个媒体造词

---

## 7. FastGPT (sealos/labring)

| 断言 | 验证 | 状态 |
|---|---|---|
| 真实存在 | ✅ [labring/FastGPT GitHub](https://github.com/labring/FastGPT) | ✅ |
| 定位 | ✅ 知识库 + 工作流编排 + RAG 检索 + 可视化 flow 设计 | ✅ |
| 开源 | ✅ Apache 2.0 协议 | ✅ |
| GitHub stars | 实测 **27,677 stars** (2026-04-11) | ✅ |
| 创建时间 | 2023-02-23 | ✅ |
| 归属 | labring / sealos 团队 | ✅ |

**验证**:
- [labring/FastGPT GitHub](https://github.com/labring/FastGPT) (27,677 stars, created 2023-02-23)

**博客可以写**: "**FastGPT (labring, 开源 27k+ stars)** —— 专注知识库+工作流垂直场景的开源 Agent 平台，不是通用 agent 编排器"

---

## 8. 国内产品第二类整体推荐写法

给博客"第二类：Agent 编排平台"这一节的建议版本：

| 产品 | 归属 | 发布时间 | 定位 | 开源 |
|---|---|---|---|---|
| **Dify** | langgenius | 2023-04 | 开源 agentic workflow builder，137k stars | ✅ |
| **扣子 Coze** | 字节 | 2024-02-01（国内版）/ 2023-12（海外版） | C 端 bot + 智能体，拖拽编排 | ❌ |
| **文心智能体平台 (AgentBuilder)** | 百度 | 2024-04-16（前身灵境矩阵 2023-09） | C 端创作者智能体，对标扣子 | ❌ |
| **千帆 AppBuilder** | 百度智能云 | 2023-12-20 | B 端企业级 agent 开发工作台 | ❌ |
| **阿里云百炼 (Bailian/Model Studio)** | 阿里云 | — | 低代码 agent 编排：支持智能体/工作流/智能体编排三模式 | ❌ |
| **腾讯元器** | 腾讯云 | 2024-05-17 | 基于混元的 C 端智能体平台，主打微信/QQ 分发 | ❌ |
| **FastGPT** | labring | 2023-02 | 知识库+工作流垂直场景，开源 27k stars | ✅ |

**不应放入第二类**:
- **火山引擎 AgentKit** — 是云原生 agent **基础设施**（SDK + 运行时），不是可视化编排平台。应单独列为"基础设施/底座"或放在"第三类代码框架的基础设施搭子"
- **百度秒哒 (MiaoDa)** — 是零代码应用生成器（lovable/v0 类），目标是"一句话生成应用"，不是给人编排 agent 的

---

## 9. 博客里其他技术断言的求证

### 9.1 Cognition *Don't Build Multi-Agents* Flappy Bird 例子 ✅

**验证成功**，[原文 cognition.ai/blog/dont-build-multi-agents](https://cognition.ai/blog/dont-build-multi-agents)：

> **作者**: Walden Yan
> **发布日期**: 2025-06-12
> **原文引用**: "Suppose your **Task** is 'build a Flappy Bird clone'. This gets divided into **Subtask 1** 'build a moving game background with green pipes and hit boxes' and **Subtask 2** 'build a bird that you can move up and down'."

原文后续描述：一个 subagent 错误地生成了 Super Mario Bros 风格背景；另一个 subagent 生成的鸟 "doesn't look like a game asset and it moves nothing like the one in Flappy Bird." Walden 用这个场景论证并行 subagent 因为缺少共享 context 会产生冲突。

**博客可以原文引用**这段话（带引号和作者署名 "Walden Yan, Cognition 2025-06-12"）。

### 9.2 LangChain 2023-05 Plan-and-Execute 博客 ✅

**验证成功**，URL 已重定向到 [blog.langchain.com/plan-and-execute-agents](https://blog.langchain.com/plan-and-execute-agents)：

> **发布日期**: 2023-05-10
> **作者**: LangChain (无具名作者)

**关键原话 (非完全逐字但可引用)**:
- "Action Agents" (ReAct 框架) "have worked well up until now, but several things are changing which present some cracks in this algorithm."
- "User objectives are becoming more complex" —— 开发者需要 agents "handle more complex requests yet also be more reliable."
- "As objectives are more complex, more and more past history is being included to keep the agent focused on the final objective while also allowing it to remember and reason about previous steps."
- 核心问题：复杂度和可靠性要求增长导致 "prompt sizes to increase"，因为开发者要塞入 "more instructions around how to use tools" 以及更多历史上下文，这在当时的大多数 LLM 上会出问题

**博客原本的 "complex objectives + reliability → prompt size unsustainable"** —— 是**精确的原意总结**，但不是原文逐字。可以改写为：
- *引用版*: > "User objectives are becoming more complex... prompt sizes to increase... this causes issues with most language models." (LangChain 2023-05-10)
- *或保留意译*: *LangChain 在 2023-05-10 的 plan-and-execute 博客里首次官方承认 ReAct 存在"随着目标变复杂和可靠性要求提升，prompt size 持续膨胀变得不可持续"的问题*

URL: https://blog.langchain.com/plan-and-execute-agents/ (注意：旧 `blog.langchain.dev` 会 308 重定向到 `blog.langchain.com`)

### 9.3 Anthropic *Building Effective Agents* ✅

**验证成功**，[anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)：

> **发布日期**: **2024-12-19** ✅
> **作者**: **Erik Schluntz and Barry Zhang** ✅
> **原话**: "Written by Erik Schluntz and Barry Zhang. This work draws upon our experiences building agents at Anthropic and the valuable insights shared by our customers, for which we're deeply grateful."

博客断言 "2024-12-19 发布，作者 Schluntz + Zhang" —— **完全正确**，可以直接引用。

### 9.4 BabyAGI 架构定性 ⚠️

**博客的声明**:
- BabyAGI 不是 ReAct，是 Plan-and-Execute + 任务队列
- 有三个 agent：execution / task-creation / prioritization
- 用 Pinecone 做记忆

**验证** (通过 [babyagi_archive GitHub README](https://github.com/yoheinakajima/babyagi_archive) 和搜索结果确认)：

| 断言 | 验证 |
|---|---|
| 非 ReAct，是 P&E + 任务队列 | ✅ README 原话："infinite loop" 执行 "Pulls the first task from the task list → execution agent → store result → create new tasks and reprioritize the task list based on the objective and the result" — 这是 plan-execute-replan 循环 |
| 三个 agent | ✅ README 显式列出：`execution_agent()` / `task_creation_agent()` / `prioritization_agent()` |
| Pinecone 记忆 | ✅ Yohei Nakajima 2023-03-28 原博客标题就叫 *"Task-driven Autonomous Agent Utilizing GPT-4, Pinecone, and LangChain for Diverse Application"*。最初版本用 Pinecone；archived 版本 README 同时支持 Chroma/Weaviate；最初确实是 Pinecone |
| 公开时间 | 2023-03-28 / 2023-03-29 |
| 最初代码量 | ~140 行 Python |

**博客三个断言全部正确**，但可以补细节：
- Pinecone **是最初版本**，后续版本扩展了 Chroma/Weaviate
- 公开发布日期是 **2023-03-28 (blog) / 2023-03-29 (GitHub)**
- 原 repo (`yoheinakajima/babyagi`) 2024-09 之后已经被作者**归档并替换**为 functionz 框架；原始架构的历史版本在 [yoheinakajima/babyagi_archive](https://github.com/yoheinakajima/babyagi_archive)

### 9.5 AutoGPT 架构定性 ✅ (部分)

**博客声明**:
- AutoGPT 是 ReAct 的产品化
- 用 thoughts JSON 格式
- 字段：text / reasoning / plan / criticism / speak + command

**验证**:
- ✅ AutoGPT 官方 prompt 模板确实要求 JSON 响应，`thoughts` 对象包含 `text` / `reasoning` / `plan` / `criticism` / `speak` 五个字段，外加 `command` 对象（含 `name` 和 `args`）
- ✅ 这个格式是 AutoGPT 最早版本的核心，在多个 issue 和 discussion 里被引用（如 `Significant-Gravitas/AutoGPT#7197`）
- ⚠️ "ReAct 的产品化" —— 这是**风格上的描述**。严格说 AutoGPT 用的是自己的 prompt schema，不是经典的 Thought/Action/Observation 三元组。但精神上一致（每步思考-行动-观察），可以叫"ReAct 变体"或"ReAct-style"

**博客可以保留**，建议小修：
- "ReAct 的产品化" → "**ReAct 风格的产品化**，用自定义的 thoughts JSON schema（text/reasoning/plan/criticism/speak + command）替代经典的 Thought/Action/Observation 三元组"

---

## 10. 快速清单：博客必须修改的地方

| # | 原写法 | 改成 | 严重度 |
|---|---|---|---|
| 1 | "百度 xxx（AppBuilder / 插件库 / 灵境矩阵 / 妙搭）" | **文心智能体平台 (AgentBuilder) + 千帆 AppBuilder**（两个产品，两个用户群） | ❌ 必改 |
| 2 | "妙搭" | 不存在。百度的是**秒哒 (MiaoDa)**，而且它不属于第二类 | ❌ 必改 |
| 3 | "灵境矩阵" | 如果提，要注明"已于 2024-04 升级为文心智能体平台 (AgentBuilder)" | ⚠️ 过时 |
| 4 | "阿里百炼 Visual Studio" | **阿里云百炼 (Bailian / Model Studio)** | ❌ 必改 |
| 5 | "ModelStudio-ADK" | 删掉 — 不是官方产品名 | ❌ 必改 |
| 6 | "火山引擎 AgentKit" 放在第二类 | 移出第二类，归为"基础设施/底座层"，和 Bedrock AgentCore 同类 | ⚠️ 类别错 |
| 7 | Dify "2023-03 首次 commit" | **2023-04 开源发布**（GitHub repo 创建于 2023-04-12） | ⚠️ 月份错 1 个月 |
| 8 | 腾讯元器"需求证" | 已验证：**2024-05-17 上线，基于混元，腾讯云智能体创作与分发平台**。不要和腾讯元宝 (C 端聊天 App) 搞混 | ✅ 可用 |
| 9 | 扣子 2024-02-01 | ✅ 验证正确，保持 | ✅ |
| 10 | Dify 137k+ stars | ✅ 实测 137,260 stars，保持 | ✅ |

---

## 11. 所有 URL 汇总 (可验证)

### 百度系
- https://agents.baidu.com/ — 文心智能体平台官网
- https://cloud.baidu.com/product-s/qianfan_home — 千帆大模型平台
- https://github.com/baidubce/app-builder — 千帆 AppBuilder SDK
- https://www.miaoda.cn/ — 秒哒官网
- https://cloud.baidu.com/doc/MIAODA/s/Sm88db6er — 秒哒公有云文档
- https://www.oschina.net/news/257981 — 灵境矩阵发布新闻 (2023-09)
- http://www.news.cn/tech/20241112/041eaddb008049fdb7a9b93d7713c4f1/c.html — 秒哒发布 (2024-11-12)

### 字节系
- https://www.coze.cn/ — 扣子国内版
- https://news.qq.com/rain/a/20240203A07KG700 — 扣子上线新闻
- https://www.volcengine.com/product/agentkit — 火山引擎 AgentKit
- https://www.volcengine.com/solutions/ai-cloud-native-agentkit — AI 云原生 Agent 套件

### 阿里系
- https://www.aliyun.com/product/bailian — 百炼官网
- https://www.alibabacloud.com/help/zh/model-studio/single-agent-application — 百炼智能体应用文档
- https://github.com/agentscope-ai/agentscope — AgentScope 开源框架

### 腾讯系
- https://tech.cnr.cn/techph/20240517/t20240517_526708228.shtml — 元器上线新闻

### 开源
- https://github.com/langgenius/dify — Dify (137,260 stars)
- https://github.com/labring/FastGPT — FastGPT (27,677 stars)
- https://github.com/yoheinakajima/babyagi_archive — BabyAGI 原始架构 archive

### 技术引用
- https://cognition.ai/blog/dont-build-multi-agents — Walden Yan 2025-06-12
- https://blog.langchain.com/plan-and-execute-agents/ — LangChain 2023-05-10 (原 `.dev` 已重定向到 `.com`)
- https://www.anthropic.com/engineering/building-effective-agents — Schluntz & Zhang 2024-12-19

---

## Summary

- **百度产品判决**: 博客应该写 **文心智能体平台 (AgentBuilder, 2024-04)** 作为第二类代表；如果要强调企业视角，补 **千帆 AppBuilder (2023-12)**。用户记的"灵境矩阵/插件库"是**对的但过时了**（灵境矩阵已于 2024-04 升级为 AgentBuilder），"妙搭"是错字（应为"秒哒"，且不属于第二类）。
- **阿里产品判决**: 博客应该写 **阿里云百炼 (Bailian / Model Studio)**，删掉 "百炼 Visual Studio" 和 "ModelStudio-ADK"。AgentScope 留在第三类（开源代码框架）。
- **火山引擎 AgentKit**: 不属于第二类，是基础设施层。
- **腾讯元器**: 真实存在，2024-05-17 上线，可用。
- **扣子、Dify、FastGPT**: 博客原有描述基本正确，只有 Dify 的"2023-03 首次 commit"应改为"2023-04"。
- **技术引用 (Cognition / LangChain / Anthropic / BabyAGI / AutoGPT)**: 全部求证完成，所有日期和作者名正确，可以直接引用。
