# Domestic (Chinese) Agent Platforms Research

Last Updated: 2026-04-11

> **Research Methodology**: Public-sources only — official documentation, technical blogs, open-source READMEs, large-vendor engineering sharings, media reports, community discussions. Code-level analysis not possible for closed-source platforms.

## Scope

This file covers **seven mainstream Chinese Agent platforms** that together represent nearly all enterprise "Agent development" activity in China:

1. Dify (open source + cloud)
2. Coze / 扣子 (ByteDance)
3. Alibaba Bailian (Alibaba Cloud Model Studio)
4. Tencent Yuanqi (腾讯元器)
5. FastGPT (open source)
6. Baidu Qianfan AppBuilder
7. Volcengine AgentKit

The research question: **When Chinese enterprises say "doing Agent development," what are they actually building — autonomous decision-making agents, or visual workflows with LLM nodes?**

**Short answer**: Overwhelmingly the latter. The only partial exception is Alibaba Bailian's high-code ADK/AgentScope layer.

---

## Sources (Key URLs)

### Dify
- [langgenius/dify](https://github.com/langgenius/dify) (137,000+ stars) (accessed: 2026-04-11)
- [Dify Official Docs](https://docs.dify.ai) (accessed: 2026-04-11)
- [Dify 2.0 Introduction (Zhihu)](https://zhuanlan.zhihu.com/p/1951665390498354250) (accessed: 2026-04-11)
- [Dify Agent vs Workflow (cnblogs)](https://www.cnblogs.com/lightsong/p/18927748) (accessed: 2026-04-11)

### Coze / 扣子
- [coze.cn Official Site](https://www.coze.cn/) (accessed: 2026-04-11)
- [Coze 2.0 Deep Analysis (Geekpark)](https://www.geekpark.net/news/359437) (accessed: 2026-04-11)
- [Coze Multi-Agent Mode Deep Dive (CSDN)](https://blog.csdn.net/weixin_68755187/article/details/141020176) (accessed: 2026-04-11)

### Alibaba Bailian
- [Bailian Platform](https://bailian.aliyun.com) (accessed: 2026-04-11)
- [High-code Era Arrives (36kr)](https://www.36kr.com/p/3484631535033224) (accessed: 2026-04-11)
- [Full-stack Agent Capability (InfoQ)](https://www.infoq.cn/article/ya6zml7irki6ph3c56hr) (accessed: 2026-04-11)

### Cross-Platform Analysis
- [Four Agent Platforms Comparison (Woshipm)](https://www.woshipm.com/evaluating/6204580.html) (accessed: 2026-04-11)
- [Domestic Agent Platforms Deep Review (cnblogs)](https://www.cnblogs.com/ExMan/p/18727491) (accessed: 2026-04-11)
- [Agent vs Workflow Essential Difference (Zhihu)](https://www.zhihu.com/question/1896707093580448857) (accessed: 2026-04-11)

### Theory
- [Lilian Weng: LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/)
- [DataWhale Agent Tutorial](https://github.com/datawhalechina/agent-tutorial)

---

## 1. Dify (langgenius/dify)

### Product Positioning

Open-source LLM application development platform. Positions itself as a "production-grade Agentic workflow development platform." Target users: developers and AI application builders, supporting a "no-code to low-code" continuum.

### Core Abstractions

Dify offers five application types:

| Type | Definition | Execution Characteristics |
|------|------|---------|
| **Chatbot** | Conversation with prompt + knowledge base | No workflow, single-turn |
| **Chatflow** | Chatbot + visual flow | Mixes conversation with workflow logic |
| **Workflow** | Pure predefined task flow | Sequential, conditional branching, fully predictable |
| **Agent** | Workflow where one node has LLM autonomous decision | ReAct or Function Calling within node |
| **Completion** | Single input-output transformation | No multi-turn |

**Critical design insight**: In Dify, **Agent is not a standalone application type — it is a node type within a Workflow**. When a step in a flow requires complex reasoning, the developer replaces that step with an Agent node, preserving the overall flow's controllability.

### Execution Model

Dify's Agent node supports two strategies:

- **ReAct**: Thought → Tool selection → Observation → Repeat (loop reasoning)
- **Function Calling**: Direct user-intent-to-function mapping, no iterative reasoning

User chooses which strategy fits the use case. Dify defaults to encouraging Workflow over Agent for production cost control.

### Dify's Definition of "Agent"

From Dify official documentation: Agent is "the moment when a workflow learns to think autonomously." Formally defined as:

> **Agent = LLM + Tool Calling + Autonomous Reasoning** (within a workflow node)

Official guidance: "95% of scenarios are handled by Workflow. Only 5% require Agent's dynamic decision-making."

### Production Features

- Version management, publishing, rollback
- Logging, cost monitoring, model call tracing
- Canary/gray-scale releases
- Queue-based scheduling engine with complex dependency support (Dify 2.0 addition)

### Key Finding

**Dify's "Agent" is fundamentally a "workflow node with extended intelligence," not an autonomous system.** The design philosophy is "predictability first, autonomy as accent."

---

## 2. Coze / 扣子 (ByteDance)

### Product Positioning

ByteDance's one-stop AI Bot development platform. Target users: non-technical business people (product managers, operations staff). Marketing emphasis: "build a Bot in 5 minutes."

### Core Abstractions

Coze's unit is the **Bot** (conversational application), not strictly an Agent. In late 2024, official terminology shifted from "Bot" to "智能体" (intelligent agent), but the underlying architecture is unchanged.

| Layer | Component | Description |
|------|------|------|
| **Model & Persona** | LLM + Prompt | Defines Bot's personality and capabilities |
| **Workflow** | Visual flow nodes | Fixed task orchestration |
| **Knowledge Base** | Document library + vector retrieval | External knowledge source |
| **Plugins** | Tools and APIs | External system connectors |
| **Multi-Agent** | Agent collaboration | Multiple Bots can call each other |

### Design Philosophy

Coze explicitly articulates a **"Workflow vs Agent" dichotomy**:

- **Workflow mode**: For repetitive tasks with fixed flows (customer service, order processing)
- **Agent autonomous mode**: With strong reasoning models, the Agent can self-plan and auto-select tools

But in the actual implementation, **Coze's "Agent mode" is still realized through workflow nodes**. Official statements acknowledge:

> **"规划是最难以工程化的。用工作流编排方式，但这与大模型的开放性产生本质冲突。"**
> *(Planning is the hardest thing to engineer. Using workflow orchestration works, but it fundamentally conflicts with the openness of LLMs.)*

This is a rare public admission from the largest Chinese Agent platform that they did **not** solve the core autonomous planning problem — they chose to side-step it.

### Tool Calling Mechanism

- Plugins are designed for "conversational Agents"
- The AI automatically selects plugins based on user intent
- Supports automatic parameter filling and dynamic invocation

### Deployment & Production Features

- Multi-channel publishing: WeChat, QQ, Feishu, Web
- Built-in database capability (Bot Memory)
- Multi-Agent collaboration mode (inter-Agent calls)
- Official documentation is thin on cost control and gray-scale testing

### Coze's Definition of "Agent"

Agent = **LLM persona + Workflow + Knowledge Base + Memory**

Notably, this is different from Lilian Weng's canonical definition. There is **no mention of Planning** — emphasis is on "complete application composition."

### Key Finding

**Coze's "Agent" is effectively "a Bot that can chat," and its core capability is "automatic plugin selection" rather than "autonomous task decomposition."**

---

## 3. Alibaba Bailian (Alibaba Cloud Model Studio)

### Product Positioning

Alibaba Cloud's strategy: "Provide the soil where real AI Agents can grow." Attempting to upgrade from "model provider" to "full Agent development stack."

### Two-Tier Strategy

Unlike all other platforms in this survey, Alibaba explicitly provides **two separate products** for two different user profiles:

| Tier | Product | Target User | Nature |
|------|---------|-------------|--------|
| **Low-code** | Visual Studio (drag-and-drop) | Non-developers | Visual workflow builder |
| **High-code** | ModelStudio-ADK / AgentScope | Developers | **Real Agent framework with Plan-and-Execute** |

This two-tier split is the **clearest public statement from any Chinese Agent vendor that "Agent platform" and "Agent framework" are different products solving different problems**.

### ADK Framework Features (high-code, 2024)

Characteristics that distinguish it from the low-code Visual Studio:

| Feature | Description |
|------|------|
| **Autonomous planning** | Agent can decompose complex tasks, dynamically adjust strategy |
| **Multi-turn reflection** | Loop execution, result feedback, self-correction |
| **Tool calling** | 200+ model APIs, knowledge bases, databases, sandboxes |
| **Memory management** | Short-term context + long-term knowledge base |
| **Observability** | Full-link logging, cost tracing, auditing |
| **Dynamic reasoning** | Based on Qwen3 with 90% decision success rate |

### Execution Model

Based on the open-source AgentScope framework, ADK supports:
- **Plan-and-Execute mode** (plan first, then execute)
- **ReAct loop**
- **Multi-Agent collaboration**

### Alibaba's Definition of "Agent"

Official emphasis: Agent = **Autonomous Decision-Making + Dynamic Reflection + Loop Execution**. This is the most ambitious Chinese vendor statement, closest to the Lilian Weng canonical definition and directly parallel to "Coding Agent"-style autonomous execution.

### Key Finding

**Alibaba is the only Chinese vendor that openly distinguishes "workflow application platform" from "autonomous Agent framework."** The Visual Studio serves enterprise buyers needing controllable workflow + AI nodes; the ADK serves development teams building real autonomous systems. This is the most honest product positioning in the Chinese Agent ecosystem.

**Note for future research**: ADK / AgentScope deserves a dedicated source-level analysis — planned as `alibaba-adk.research.md`.

---

## 4. Tencent Yuanqi (腾讯元器)

### Product Positioning

Hunyuan-model-based "one-stop intelligent agent creation platform." Two layers:
- **C-end: Yuanqi** (lightweight, individual developers)
- **B-end: Tencent Cloud Intelligent Agent Development Platform** (enterprise)

### Core Abstractions

- **Prompt mode**: Direct prompt configuration, no workflow
- **Workflow mode**: Visual orchestration with plugins + knowledge base + model nodes

Execution nodes: conditional branching, Python processing, multi-step orchestration.

### Positioning Characteristics

- Strong emphasis on "publishing channel advantages" (WeChat, QQ ecosystem)
- Compared to Coze, narrower feature coverage
- Lighter positioning, suitable for lightweight applications

### Key Finding

**Tencent Yuanqi is essentially a "Coze competitor" — architecturally similar, positioned lighter, with a smaller ecosystem.**

---

## 5. FastGPT (sealos/labring, open source)

### Product Positioning

Open-source enterprise-grade AI knowledge base and Agent building platform. Differentiator: local deployability.

### Primary Use Cases

- Knowledge base Q&A systems
- Retrieval-augmented conversational Agents
- Complex workflow orchestration

### Core Abstractions

- **Flow module**: Visual workflow node orchestration
- **Knowledge base**: Document processing + vector retrieval + RAG
- Node types: conversation, API, conditional branching

### Technical Features

- Hybrid retrieval (BM25 + vector) + reranking
- Multi-model integration
- Graphical Flow design, no code required

### Key Finding

**FastGPT is focused on the "knowledge base + workflow" vertical, not a general-purpose Agent platform.** Its "Agent" capability is specifically retrieval + workflow, not autonomous decision-making.

---

## 6. Baidu Qianfan AppBuilder

### Product Positioning

Baidu's enterprise-grade "large model application development platform." Covers model calling, application development, and operations full lifecycle.

### Core Abstractions

- **Model & Workflow**: Two operation modes
- **Workflow**: Node + connection visual composition of knowledge bases, APIs, and LLMs

### Workflow Agent Types

- Workflow Agents (ticket ordering, fitness assistants, etc.)
- Interactive writing Agents
- Multi-agent collaboration

### Characteristics

- Simple structure, short development cycle
- Lacks model-depth management capability
- Complete enterprise loop (model center + application center + data center + operation center)

### Key Finding

**Baidu AppBuilder architecturally resembles Coze/Dify but has the most complete enterprise-grade feature set.** It is a workflow platform with enterprise polish, not a different architectural category.

---

## 7. Volcengine AgentKit (ByteDance Cloud)

### Product Positioning

ByteDance's enterprise AI cloud service platform. Offers:
- Yunque large model (130B params, Chinese-optimized)
- AgentKit: enterprise-grade Agent development and deployment toolchain

### Three-Layer Architecture

Technology productization + capability servitization + scenario industrialization

### AgentKit Coverage

- Agent development (model + workflow + tools)
- Deployment and operations
- Cost optimization and security isolation

### Key Finding

**Volcengine is primarily an "infrastructure provider" rather than a building platform.** Its positioning differs from the other visual Agent platforms — it offers the underlying compute, model serving, and deployment infrastructure, and Agent development is built on top by the customer.

---

## Cross-Platform Comparison Matrix

| Dimension | Dify | Coze | Alibaba Bailian | Tencent Yuanqi | FastGPT | Baidu AppBuilder | Volcengine |
|---|---|---|---|---|---|---|---|
| **Users** | Developers + creators | Business people | Developers + enterprise | Light developers | Developers | Enterprise dev | Enterprise tech |
| **Core Paradigm** | Visual workflow + Agent node | Bot + workflow + plugins | Dual: low-code + high-code ADK | Prompt / workflow | Workflow + KB | Workflow orchestration | Cloud service |
| **Agent Definition** | Workflow's autonomous reasoning node | Bot's auto plugin selection | **Autonomous planning + loop execution (ADK only)** | Prompt-driven | Workflow node | Workflow orchestration | N/A |
| **ReAct Support** | ✓ Official | ✗ Workflow primarily | ✓ | ✗ | ✗ | ✗ | ✓ |
| **Tool Calling** | Function Calling + custom | Plugin protocol | Standard API | Plugins + API | Nodes | Component nodes | Standard API |
| **Visual Authoring** | Drag-first | Fully visual | Low-code visual | Low-code | Fully visual | Fully visual | Code framework |
| **Code Extension** | ✓ Python nodes | ✓ Limited | ✓ Full (ADK) | ✓ Limited | ✗ | ✗ | ✓ Full |
| **Version Management** | ✓ Full | ✓ | ✓ | ✓ Simplified | ✓ | ✓ | Cloud native |
| **Multi-Agent** | ✓ Workflow call | ✓ Native | ✓ | ✗ | ✗ | ✓ | ✓ |
| **Publishing Channels** | Web API | Multi-social | Cloud-native | WeChat/QQ/Web | Self-hosted | Web API | Cloud API |
| **Open Source** | ✓ 137k stars | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ |

---

## Core Findings

### Finding 1: Chinese platforms' "Agent" is essentially "enhanced visual workflow"

**Observation**: All seven platforms (except Alibaba ADK) claim to support "Agent," but the implementation is remarkably uniform.

**The actual architecture**:
- Agent node = LLM + tool selection + limited multi-turn reasoning
- Overall execution order, branching logic, and error handling are still controlled by visual flow
- The LLM's autonomy is constrained within specific nodes — it cannot re-plan the entire task flow

**Contrast with Coding Agents (Claude Code, Devin, Manus)**:
- Coding Agent: Given a goal → LLM autonomously decomposes → step-by-step execution → self-reflection → dynamic plan adjustment
- Chinese platform Agent: Follow designed flow → some steps let LLM decide → continue following the flow

### Finding 2: "Planning" is the hardest thing to engineer — and most platforms gave up on it

**Coze's admission**: "Planning is the hardest thing to engineer. Using workflow orchestration, but this conflicts fundamentally with the openness of LLMs."

**Result**: All platforms adopt a compromise:
- Let the LLM choose **within predefined node boundaries**
- Not let the LLM **re-plan the task flow itself**

**Only Alibaba ADK is an exception**: Plan-and-Execute mode is provided, allowing the Agent to make arbitrary decisions at the code level — but this requires developer programming ability.

### Finding 3: All use cases are in the "workflow" category

Actual applications (from survey data):

- **Customer service bots**: Knowledge retrieval + fixed response flow → Workflow, no Agent needed
- **Order processing**: API calls + conditional branching → Workflow
- **Knowledge Q&A**: Vector retrieval + LLM answer → RAG pipeline, not Agent
- **RPA replacement**: Structured task + tool call → Workflow
- **Content creation**: Multi-step editing → Workflow

**Scenarios truly requiring Agent autonomy are rare.** Industry consensus on the "5% rule": 95% of scenarios are served by Workflow; only 5% need Agent dynamic decision-making.

### Finding 4: The Dify / Coze / Alibaba three-way split

| Platform | Design Philosophy | Target User | Implementation Path |
|------|---------|---------|---------|
| **Dify** | Developer-friendly + open ecosystem | AI creatives + SMBs | Open source + cloud service |
| **Coze** | Simplest usability | Business people + creators | Closed source + social integration |
| **Alibaba Bailian** | Enterprise full-stack | Large enterprise + deep dev | Dual strategy (low-code + high-code) |

**Selection logic**:
- Want fast onboarding + community support → Dify
- Want publishing to WeChat/QQ, social ecosystem integration → Coze
- Want to build real autonomous Agent systems + have dev team → Alibaba ADK

### Finding 5: Workflow vs Agent distinction is systemically ignored

**Theoretical definition** (Lilian Weng):
```
Agent = LLM + Memory + Planning + Tool Use
```

**Actual Chinese implementation**:
```
Agent = LLM + Tool Selection + Bounded Reasoning
```

What's missing: **Planning** — the ability to self-plan task decomposition.

In most Chinese platforms, "Agent planning" is in reality **a flow that a human dragged out in a UI.**

---

## Why Is It Like This?

### Technical Difficulty
- Autonomous planning requires strong LLM reasoning. In 2023-2024, mainstream Chinese models weren't strong enough
- Only recently (2024-2025) with DeepSeek-R1, Qwen3, etc. have strong reasoning models emerged, enabling change

### Commercial Considerations
- Enterprise customers demand "explainable, controllable, predictable"
- Fully autonomous Agent systems are high-risk, high-cost, and uncontrollable
- Visual workflow + bounded Agent nodes = best "usability + controllability" trade-off

### Engineering Cost
- Autonomous planning requires multi-turn reasoning, 4-15x token consumption, high cost
- Response time 3-15 seconds, poor for real-time interaction
- Error rates are higher, hard for enterprise production to accept

---

## Difference from Overseas Coding Agents

### Claude Code / Cursor / Devin
- **Design goal**: Autonomously complete code work
- **Execution model**: Goal → autonomous decomposition → multi-turn reasoning → reflection → correction
- **UX**: User only tells AI "what to do," AI decides "how to do"
- **Cost & time**: High (but resolves complex problems in one go)
- **Use cases**: Code writing, bug fixing, small feature development

### Chinese Agent Platforms (Dify / Coze / Alibaba ADK)
- **Design goal**: Provide application development tools
- **Execution model**: Human designs flow → LLM makes choices within nodes
- **UX**: Need to configure flow, but LLM also participates in decisions
- **Cost & time**: Low (fast iteration, high controllability)
- **Use cases**: Customer service, office assistants, content generation, knowledge Q&A

**Essential distinction**:
- **Coding Agent** = "Autonomous agent worker"
- **Chinese Agent platforms** = "Workflow automation tool with LLM nodes"

---

## One-Sentence Conclusion

**When Chinese enterprises say "doing Agent development," they are essentially building "visual workflow engines + LLM nodes," not "autonomous decision-making Agent systems."** This is not a regression; it is a rational choice for enterprise needs. In most scenarios, **a controllable workflow with AI at certain steps** provides more business value than a fully autonomous Agent.

---

## Position on the Agent Architecture Spectrum

From the `summary.md` taxonomy, these platforms map as follows:

| Category | Chinese Platforms |
|---|---|
| **③ Low-Code Workflow Platform** | Dify, Coze, Yuanqi, FastGPT, Baidu AppBuilder, Alibaba Visual Studio |
| **④b Code Framework (P&E school)** | **Alibaba ADK / AgentScope** — the only Chinese representative |
| **Infrastructure (not a platform)** | Volcengine AgentKit |

The absence of any Chinese representative from category ④a (ReAct-based code framework like Mastra or LangGraph) is worth noting. The domestic ecosystem skipped directly from visual workflow (③) to P&E code framework (④b), while the Western ecosystem has strong representation in ④a.

---

## Postscript

This research covered the seven most mainstream Chinese Agent platforms through 50+ technical articles, official documents, and product reviews. The goal was to provide an empirical base for the "Enterprise Agent" layer of the Agent Map.

**The most important finding**: What Chinese platforms do is fundamentally different from what overseas Coding Agents do. This is not a "stage of development" lag — it is a rational choice for **different application scenarios**. Understanding this is essential for correctly distinguishing what is an Agent, what is a Workflow, and what is a Chatbot.
