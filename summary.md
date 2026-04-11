# LLM Agent Architecture — Research Synthesis

Last Updated: 2026-04-12 (final after 6 source-level framework analyses)

Central question: **"What is an Agent?"** — specifically, who controls the flow of execution, and how the answer to that question separates the ecosystem into distinct architectural families.

> **Research completion note (2026-04-12)**: Six frameworks have now been analyzed at source-code level with file:line citations: Mastra, Google ADK, AgentScope, Pydantic AI, LangGraph, LangChain, and CrewAI. The biggest update to the working taxonomy: **④b is not empty**. CrewAI's `src/crewai/experimental/agent_executor.py` is a genuine Plan-and-Execute implementation extending `Flow[AgentExecutorState]` with typed `TodoList`, isolated `StepExecutor`, and `PlannerObserver` — explicitly citing arxiv 2503.09572 (Plan-and-Act). However, ④b exists **only in experimental / middleware / historical forms** across all major frameworks. The industry pattern is: evaluate P&E → classify as experimental or middleware → keep ReAct as the default production recommendation. The user's ⑤ remains distinctive not because P&E is a novel concept but because making P&E + typed dataflow the **default production architecture** is a path the rest of the industry has deliberately stepped away from.

> **Correction history**: An initial version placed "Alibaba ADK / AgentScope" in ④b based on media articles. Direct README verification showed AgentScope is ReAct-based. Source-level analysis later refined this: AgentScope has genuine `Plan`/`SubTask` Pydantic models (closest ④a to ④b), but execution is ReAct. "ModelStudio-ADK" was a media term, not an official product name.

---

## The Agent Map (Working Taxonomy)

All currently-observed Agent systems fall into one of these categories, distinguished by **who decides control flow** and **whether there is a Plan artifact**:

| # | Category | Representatives | Control Flow Authority | Plan Artifact | Validation |
|---|---|---|---|---|---|
| ① | **Coding CLI Agent** | Claude Code, Codex, Cursor, Aider | LLM in ReAct loop | None | Tool API shape |
| ② | **Cloud Autonomous Agent** | Manus, Devin, Claude Co-worker, GenSpark | LLM in ReAct loop (sandboxed) | None | Same as ① |
| ③ | **Low-Code Workflow Platform** | Dify, Coze, Yuanqi, FastGPT, Baidu AppBuilder, Alibaba Bailian Visual Studio | **Human-drawn workflow**, LLM at nodes | None | Node boundary schemas |
| ④a | **Code Framework (ReAct school)** | Mastra, Google ADK (18.9k ⭐), AgentScope (23.4k ⭐), Pydantic AI, LangGraph (`create_agent`), LangChain 1.0, Vercel AI SDK, CrewAI default | **Developer code** wraps Workflow/graph; LLM in Step-level ReAct | None | Zod/Pydantic boundary schemas + middleware |
| ④b | **Code Framework (P&E school) — experimental/middleware only** | **CrewAI `experimental/agent_executor.py`** (opt-in, extends `Flow[AgentExecutorState]`, cites arxiv 2503.09572), **LangChain `experimental.plan_and_execute`** (removed in 1.0), **LangGraph P&E tutorials** (redirected to `langchain/middleware/built-in#to-do-list`) | Pre-execution Plan generation + structured todo tracking + isolated step execution | **Yes, typed** | Framework built-in |
| ⑤ | **Custom Production System — P&E + typed dataflow as default** | Personal Agent 2.0 | Structured Plan layer + typed flow + per-capability gating, production-default | **Yes, first-class artifact** | Flow-level types |

**Key observations**:

- **① and ②** are architecturally the same (differ only in deployment).
- **③ and ④a** are philosophically the same (differ only in visual vs. code). Mastra is TypeScript Dify, Google ADK is Python Dify, AgentScope is Alibaba's Dify. The entire ④a category is "code-version of Dify."
- **④b exists but is gated**: every major framework has evaluated P&E and classified it as experimental, middleware, or historical. The pattern is crystal clear across LangChain (dropped), LangGraph (relocated), CrewAI (experimental), AgentScope (plan as data but execution as ReAct).
- **⑤ is distinctive not because P&E is rare** but because **making P&E + typed dataflow the default production architecture** is a path the industry has repeatedly explored and deliberately chosen not to promote.

## The Central Insight

**The real dividing line in the Agent ecosystem is not "multi-agent vs single-agent" or "ReAct vs Plan-and-Execute" — it is:**

> **Who has authority over the overall task flow — the LLM, the developer at design time, or a pre-execution plan artifact?**

This single question places every observed Agent system into a clear position:

- **LLM has authority** → ① ② (Coding/Cloud Agents)
- **Developer or drag-and-drop author has authority** → ③ ④a (Low-code and code-first Workflow platforms)
- **Pre-execution Plan artifact has authority** → ④b ⑤ (P&E frameworks and custom production) — **but ④b appears empty in public open source, so in practice this category contains only custom production systems like ⑤**

Everything else (tool calls, memory choice, multi-agent handoff, etc.) is secondary to this axis.

## Why the Industry Discourse Is Confused

**All three families use "Agent" as the label**, because all three have an LLM in the loop somewhere. But the three families solve fundamentally different problems:

| Family | Primary Problem | Success Metric |
|---|---|---|
| ① ② Coding/Cloud Agent | "Can the LLM accomplish an open-ended task autonomously?" | Task completion rate on benchmarks |
| ③ ④a Workflow Platform | "Can non-experts (or devs) compose LLM apps quickly?" | Time-to-first-demo, ease of iteration |
| ④b ⑤ P&E / Production | "Can we serve paying customers with predictable cost and auditability?" | SLA compliance, cost predictability, audit coverage |

Most public "Agent" discourse (Twitter, conference talks, hype cycles) focuses on family ① ②. Most enterprise spend goes to family ③. The ④b ⑤ corner is barely discussed publicly — in fact, ④b appears to be **essentially empty in public open source**, leaving ⑤ (custom production systems) as the only real representative of "Plan-artifact-driven Agents." This is a significant finding: when the user's Agent 2.0 was initially described as "rare," the actual situation is closer to "nearly unique in the public landscape."

## Key Cross-Project Findings

See [findings.md](./findings.md) for the full list. Highlights:

1. **Mastra is essentially "Dify in TypeScript"** — despite being a "code framework," its design philosophy is the same as Dify's: deterministic Workflow main line, LLM agents as nodes within the workflow. The surface is different (code vs. UI), the architecture is identical.

2. **Coze officially admits that planning is the hardest problem** — and they chose to avoid it. Their "Agent mode" is only "plugin auto-selection within a workflow node," not task decomposition.

3. **The "5% rule"** — industry consensus among Chinese platforms: 95% of scenarios can be handled by pure Workflow; only 5% need real Agent autonomy. This is used to justify the workflow-centric design of all low-code platforms.

4. **Mastra replicates Cognition's "Don't Build Multi-Agents" failure mode by default** — its agent-to-agent handoff passes only the final text output of sub-agents, losing intermediate reasoning. This is exactly what Cognition warned against.

5. **Typed dataflow is not a common abstraction** — Mastra, LangGraph, and other ④a frameworks use runtime boundary validation (Zod schemas), but none provide flow-level type guarantees. The user's Agent 2.0 system appears to be rare in combining P&E with typed dataflow.

6. **④b is not empty — it exists only in experimental / middleware / historical forms**: After source-level analysis of 6 frameworks, CrewAI's `src/crewai/experimental/agent_executor.py` was found to be a genuine Plan-and-Execute implementation (typed `TodoList`, isolated `StepExecutor`, `PlannerObserver`, citing arxiv 2503.09572). LangChain historically had `plan_and_execute` in `langchain_experimental` (dropped in 1.0). LangGraph's P&E tutorials have been redirected to `langchain/middleware/built-in#to-do-list`. AgentScope has `Plan`/`SubTask` Pydantic models but executes via ReAct with plan-as-hint-injection. **Every major framework has evaluated P&E and classified it as experimental, middleware, or historical.**

7. **The industry "experimental gating" pattern**: Across LangChain (experimental → dropped), LangGraph (tutorials → middleware), CrewAI (experimental/), AgentScope (data model only), there is a consistent pattern. P&E is explored, built to partial completeness, and then kept behind opt-in gates or relocated to a higher layer. The user's ⑤ (P&E as production default with typed dataflow) is not rare because the concept is novel — it is rare because **the industry has made P&E vs ReAct a settled decision**, and ReAct won as the default.

8. **Beware of media-sourced claims**: The earlier "Alibaba ADK supports Plan-and-Execute" claim came from 36kr and InfoQ tech articles, not from Alibaba's official product documentation or AgentScope's README. Always verify framework claims against actual README or source code. Media tends to conflate "supports planning as a feature" with "is a Plan-and-Execute framework" — these are different things.

9. **Multi-agent handoff trace loss is the most consistent anti-pattern**: Across Mastra (`AgentTool`), Google ADK (`AgentTool`), CrewAI (`DelegateWorkTool` returns plain `str`), Pydantic AI (agent-as-tool default), and AgentScope (sub-agent via tool), sub-agent delegation consistently returns only the final text output — intermediate reasoning is discarded. **This is exactly the failure mode Cognition's "Don't Build Multi-Agents" warned about.** Only Google ADK's `transfer_to_agent` (preserves full context via shared `InvocationContext`) avoids this. LangGraph's subgraph pattern avoids it via shared state channels.

## Cross-References to Other Research

- **`/Users/linguanguo/dev/CyberMnema/timeline/2026/04/W15/Agent架构调研-2026-04-07.md`** — Initial survey of 8 academic Agent paradigms (ReAct, Reflexion, ToT, P&E, ReWOO, LLMCompiler, LATS, P-t-E), 6 empirical Agent types, and the 50-year typed dataflow theoretical lineage.
- **`/Users/linguanguo/dev/CyberMnema/timeline/2026/04/W15/Agent复杂度取舍-2026-04-08.md`** — Triangulation of multi-agent complexity debate (Cognition "Don't Build Multi-Agents" + Berkeley MAST 14 failure modes + personal production experience).
- **`/Users/linguanguo/dev/llm-memory-research/context.summary.md`** — Memory/context analysis of 7 Coding CLI Agents (orthogonal axis).
- **`/Users/linguanguo/dev/llm-memory-research/anthropic-context-engineering.research.md`** — Anthropic's official position on Agent context engineering.

## Open Questions

- ~~**Does Google ADK fit into ④a or ④b?**~~ — **Answered**: ④a (ReAct-based). See `google-adk.research.md`.
- **Is LangGraph ④a or ④b?** — Has both a ReAct prebuilt path (`create_react_agent`) and lower-level graph primitives. The prebuilt is what most users touch, so LangGraph is effectively ④a by default. But the graph primitives *could* be used to build a P&E system — whether anyone does this in practice is an open question.
- **Does OpenAI Assistants API belong anywhere?** — It's closer to a hosted execution environment than an architecture. May not fit the taxonomy at all.
- **Is there any truly public ④b framework at all?** — Worth searching for: academic implementations of ReWOO / LLMCompiler / Plan-then-Execute, as well as any vertical domain frameworks. If the answer is definitively "none," the blog's main thesis gets significantly sharper.
- **Is there a ⑥ archetype we haven't seen yet?** — Research hasn't covered the "agent OS" direction (MemGPT-style or Letta-style systems that blur memory and control flow).
