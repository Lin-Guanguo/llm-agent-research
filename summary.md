# LLM Agent Architecture — Research Synthesis

Last Updated: 2026-04-11

Central question: **"What is an Agent?"** — specifically, who controls the flow of execution, and how the answer to that question separates the ecosystem into distinct architectural families.

---

## The Agent Map (Working Taxonomy)

All currently-observed Agent systems fall into one of these categories, distinguished by **who decides control flow** and **whether there is a Plan artifact**:

| # | Category | Representatives | Control Flow Authority | Plan Artifact | Validation |
|---|---|---|---|---|---|
| ① | **Coding CLI Agent** | Claude Code, Codex, Cursor, Aider | LLM in ReAct loop | None | Tool API shape |
| ② | **Cloud Autonomous Agent** | Manus, Devin, Claude Co-worker, GenSpark | LLM in ReAct loop (sandboxed) | None | Same as ① |
| ③ | **Low-Code Workflow Platform** | Dify, Coze, Yuanqi, FastGPT, Baidu AppBuilder | **Human-drawn workflow**, LLM at nodes | None | Node boundary schemas |
| ④a | **Code Framework (ReAct school)** | Mastra, LangGraph, Vercel AI SDK | **Developer code** wraps Workflow DAG; LLM in Step-level ReAct | None | Zod boundary schemas + processor middleware |
| ④b | **Code Framework (P&E school)** | Alibaba Bailian ADK/AgentScope | Developer code with P&E primitives | Optional | Framework built-in |
| ⑤ | **Custom Production System** | Personal Agent 2.0 (P&E + typed dataflow) | Structured Plan layer + typed flow + per-capability gating | **Yes, first-class artifact** | Flow-level types |

**Key observation**: ① and ② are architecturally the same (differ only in deployment). ③ and ④a are philosophically the same (differ only in visual vs. code). ④b and ⑤ are rare — real Plan-and-Execute with structured planning is uncommon in public discourse.

## The Central Insight

**The real dividing line in the Agent ecosystem is not "multi-agent vs single-agent" or "ReAct vs Plan-and-Execute" — it is:**

> **Who has authority over the overall task flow — the LLM, the developer at design time, or a pre-execution plan artifact?**

This single question places every observed Agent system into a clear position:

- **LLM has authority** → ① ② (Coding/Cloud Agents)
- **Developer or drag-and-drop author has authority** → ③ ④a (Low-code and code-first Workflow platforms)
- **Pre-execution Plan artifact has authority** → ④b ⑤ (P&E frameworks and custom production)

Everything else (tool calls, memory choice, multi-agent handoff, etc.) is secondary to this axis.

## Why the Industry Discourse Is Confused

**All three families use "Agent" as the label**, because all three have an LLM in the loop somewhere. But the three families solve fundamentally different problems:

| Family | Primary Problem | Success Metric |
|---|---|---|
| ① ② Coding/Cloud Agent | "Can the LLM accomplish an open-ended task autonomously?" | Task completion rate on benchmarks |
| ③ ④a Workflow Platform | "Can non-experts (or devs) compose LLM apps quickly?" | Time-to-first-demo, ease of iteration |
| ④b ⑤ P&E / Production | "Can we serve paying customers with predictable cost and auditability?" | SLA compliance, cost predictability, audit coverage |

Most public "Agent" discourse (Twitter, conference talks, hype cycles) focuses on family ① ②. Most enterprise spend goes to family ③. The ④b ⑤ corner is barely discussed publicly despite serving real production traffic.

## Key Cross-Project Findings

See [findings.md](./findings.md) for the full list. Highlights:

1. **Mastra is essentially "Dify in TypeScript"** — despite being a "code framework," its design philosophy is the same as Dify's: deterministic Workflow main line, LLM agents as nodes within the workflow. The surface is different (code vs. UI), the architecture is identical.

2. **Coze officially admits that planning is the hardest problem** — and they chose to avoid it. Their "Agent mode" is only "plugin auto-selection within a workflow node," not task decomposition.

3. **The "5% rule"** — industry consensus among Chinese platforms: 95% of scenarios can be handled by pure Workflow; only 5% need real Agent autonomy. This is used to justify the workflow-centric design of all low-code platforms.

4. **Mastra replicates Cognition's "Don't Build Multi-Agents" failure mode by default** — its agent-to-agent handoff passes only the final text output of sub-agents, losing intermediate reasoning. This is exactly what Cognition warned against.

5. **Typed dataflow is not a common abstraction** — Mastra, LangGraph, and other ④a frameworks use runtime boundary validation (Zod schemas), but none provide flow-level type guarantees. The user's Agent 2.0 system appears to be rare in combining P&E with typed dataflow.

## Cross-References to Other Research

- **`/Users/linguanguo/dev/CyberMnema/timeline/2026/04/W15/Agent架构调研-2026-04-07.md`** — Initial survey of 8 academic Agent paradigms (ReAct, Reflexion, ToT, P&E, ReWOO, LLMCompiler, LATS, P-t-E), 6 empirical Agent types, and the 50-year typed dataflow theoretical lineage.
- **`/Users/linguanguo/dev/CyberMnema/timeline/2026/04/W15/Agent复杂度取舍-2026-04-08.md`** — Triangulation of multi-agent complexity debate (Cognition "Don't Build Multi-Agents" + Berkeley MAST 14 failure modes + personal production experience).
- **`/Users/linguanguo/dev/llm-memory-research/context.summary.md`** — Memory/context analysis of 7 Coding CLI Agents (orthogonal axis).
- **`/Users/linguanguo/dev/llm-memory-research/anthropic-context-engineering.research.md`** — Anthropic's official position on Agent context engineering.

## Open Questions

- **Does Google ADK fit into ④a or ④b?** — Pending research on `github.com/google/adk-python`. Early signal: framework name suggests P&E-ready, but actual primitives TBD.
- **Is LangGraph ④a or ④b?** — Has both a ReAct prebuilt path and lower-level graph primitives. Need to determine whether real-world usage is primarily ReAct-style or P&E-style.
- **Does OpenAI Assistants API belong anywhere?** — It's closer to a hosted execution environment than an architecture. May not fit the taxonomy at all.
- **Is there a ⑥ archetype we haven't seen yet?** — Research hasn't covered the "agent OS" direction (MemGPT-style or Letta-style systems that blur memory and control flow).
