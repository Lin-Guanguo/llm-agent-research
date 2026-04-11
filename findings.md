# Cross-Project Findings

Last Updated: 2026-04-11 (corrected)

Findings that span multiple categories of Agent architecture research. Each finding cites the project(s) it was derived from.

> **Correction history (2026-04-11)**: Finding 7 originally claimed Alibaba Bailian had a Plan-and-Execute code framework called "ADK / AgentScope." Direct GitHub verification showed: (a) Alibaba's actual open-source framework is AgentScope, not "ADK"; (b) "ModelStudio-ADK" was a media-only term from 36kr/InfoQ tech articles, not an official Alibaba product name; (c) AgentScope's README clearly states it uses ReAct as the core execution model, not P&E. Finding 7 has been rewritten. A new Finding 8 captures the corrected conclusion that the ④b P&E category is empty in public open source.

---

## Finding 1: The real dividing line is control-flow authority, not ReAct vs P&E

Industry debate frames the choice as "ReAct vs Plan-and-Execute" or "single vs multi-agent." These are secondary. The primary axis is:

> **Who decides the overall task flow — the LLM, a human-authored workflow, or a pre-execution Plan artifact?**

Once this question is answered, most other architectural decisions follow from it. A "multi-agent ReAct" (category ①②) and a "single-agent workflow node" (category ③) look nothing alike in practice, despite both being called Agents.

**Evidence**: Observed in every category — the confusion arises specifically because all three use "Agent" as a marketing label despite solving different problems. See `summary.md` "The Agent Map" for the full taxonomy.

---

## Finding 2: "Code framework" does not mean "autonomous agent" — Mastra is essentially Dify in TypeScript

One might assume that moving from visual workflow builders (Dify, Coze) to code frameworks (Mastra, LangGraph) means moving from "workflow" toward "autonomous agent." **This is false.**

Mastra's core execution model is:
- A **Workflow DAG** as the deterministic main line (written by the developer in TypeScript)
- **Agents as Steps within the DAG**, each internally running a ReAct loop with `stepCountIs(N)` as the default stop condition
- Sub-agents called via **explicit tool invocation**, not implicit delegation

This is structurally identical to Dify's "Workflow with Agent nodes" — the only difference is that Dify's authoring surface is drag-and-drop while Mastra's is TypeScript. **The control-flow authority is still the developer at design time, not the LLM at runtime.**

**Evidence**:
- `mastra.research.md` sections 2 (Execution Model), 3 (Workflow Concept), 4 (Agent × Workflow Organization)
- `domestic-platforms.research.md` Finding 1 ("国内平台的 Agent 本质是可视化工作流的增强")

**Implication**: When a framework calls itself an "Agent framework," look at whether the top-level composition primitive is a **Workflow graph** or a **Plan artifact**. If it's a Workflow graph, the framework is philosophically in the low-code family regardless of its authoring surface.

---

## Finding 3: Coze officially admits planning is the hardest problem — and chose to avoid it

From Coze official documentation and product analysis (cited in `domestic-platforms.research.md`):

> "规划是最难以工程化的，用工作流编排方式，但这与大模型的开放性产生本质冲突。"
> *("Planning is the hardest thing to engineer. Using workflow orchestration works, but it fundamentally conflicts with the openness of LLMs.")*

This is a rare public admission from the largest Chinese Agent platform that their "Agent mode" is not solving the core autonomous planning problem. Instead, Coze's "Agent" means **automatic plugin selection within a workflow node** — not task decomposition.

**Why this matters for the taxonomy**: It confirms that category ③ (low-code platforms) is not a naive or immature version of category ①② — it is a deliberate product decision to stay in the workflow domain rather than attempt autonomous planning.

---

## Finding 4: The "5% rule" — industry consensus that most scenarios don't need autonomous Agents

Across all seven Chinese Agent platforms studied, a consistent refrain appears:

> "95% of scenarios can be handled by a Workflow. Only 5% need an Agent's dynamic decision-making."

This is used to justify the workflow-centric design of all low-code platforms. It is also used to justify the Agent-as-workflow-node pattern (including Mastra's).

**Nuance**: The "5%" is real for enterprise workflow applications (customer service, order processing, knowledge Q&A, RPA replacement). It is not real for open-ended development tasks (coding, research, exploration) — which is why Coding Agents (① ②) and enterprise workflows (③ ④a) appear to be solving different problems and can both be right.

**Implication for writing**: A blog that claims "Agents are hyped but actually just workflows" is correct for enterprise contexts and wrong for coding contexts. The answer depends on which 95% you're in.

---

## Finding 5: Mastra replicates the exact failure mode Cognition warned against

Mastra's multi-agent handoff is implemented by **wrapping sub-agents as tools**. The parent agent calls `subAgentTool.execute(query)` and receives only the **final text output** of the sub-agent. The intermediate reasoning, tool calls, and state are discarded.

This is **exactly** the failure pattern described in Cognition's *Don't Build Multi-Agents*:
> "Every agent handoff is an opportunity to lose implicit assumptions that were never explicitly encoded."

And it matches **8 out of 14** failure modes catalogued in Berkeley's MAST 2025 study of multi-agent systems.

**Verification**: `mastra.research.md` section 6 ("Multi-Agent Support")
- Code reference: `packages/core/src/agent/agent.ts:278-279` and `agent/types.ts:276-279`
- The `agents` config and `listAgents()` method, tool-based delegation

**Implication**: Mastra's "multi-agent support" is not a solved problem — it's a documented anti-pattern. A production system using it needs to either avoid multi-agent composition or manually flatten sub-agent traces into parent context.

---

## Finding 6: Typed dataflow is rare even among "typed" frameworks

All major ④a frameworks (Mastra, LangGraph, Vercel AI SDK) advertise type safety via Zod schemas. But this type safety is **boundary-level**, not **flow-level**:

| Framework | What's typed | What's NOT typed |
|---|---|---|
| Mastra | Agent I/O, Tool I/O, Step I/O | Internal dataflow between Agent turns |
| LangGraph (assumed) | Node I/O | Edge data contracts |
| Vercel AI SDK (assumed) | Generate/stream I/O | Tool chain continuity |

Only **compile-time flow-level typing** (tracking what data is available at each point in the execution graph) provides guarantees that the next step's input will match the previous step's output structure. None of the ④a frameworks do this.

**Implication**: The personal Agent 2.0 system's "typed dataflow" abstraction may be genuinely novel in the public-framework landscape. Worth a dedicated research file (`my-agent-2.0.research.md`).

---

## Finding 7: Alibaba Bailian provides two separate products (low-code + code framework), but both are still ReAct-philosophy

Alibaba Bailian genuinely does offer two distinct products for two user profiles, which most Chinese vendors don't:

- **Bailian Visual Studio (low-code)** — a drag-and-drop workflow builder aimed at non-developers. Sits in ③.
- **AgentScope (open-source code framework)** — the actual open-source framework from Tongyi Lab (`github.com/agentscope-ai/agentscope`, 23.4k ⭐). Aimed at developers.

**Previously claimed (incorrect)**: That the code framework is "Plan-and-Execute," making Alibaba the only Chinese vendor with a ④b framework.

**Actually true (verified against AgentScope README)**: AgentScope is explicitly built around **ReAct** as its core primitive. The README's "Hello AgentScope" example uses `ReActAgent`. There is a "Meta Planner Agent" as an optional/example pattern, but it is not the core execution model. **AgentScope belongs in ④a, not ④b.**

**Also incorrect**: The name "ModelStudio-ADK" is not used in AgentScope's README or Alibaba Cloud's official product documentation. It appeared in secondary-source tech articles (36kr, InfoQ) and was picked up as if it were an official product name. The correct reference is "AgentScope" or "Alibaba Bailian's open-source code framework."

**Evidence**: Direct WebFetch of `github.com/agentscope-ai/agentscope` README (2026-04-11). The framework's executive summary says: "AgentScope is a production-ready, easy-to-use agent framework... built-in ReAct agent."

**Corrected implication**: Alibaba does offer a more thoughtful product split than Coze or Dify (low-code + real code framework), but it is not a representative of ④b. The ④b slot still has no verified Chinese public representative.

---

## Finding 8: Category ④b (P&E code framework) is empty in the public open-source landscape

After direct verification of the candidate frameworks, **no publicly-maintained, open-source Agent framework ships with Plan-and-Execute as its primary architectural primitive**.

**Verified by direct README/README-adjacent source inspection**:

| Framework | Stars | Verified execution model | Category |
|---|---|---|---|
| Mastra | (from `mastra.research.md`) | Pure ReAct + Workflow DAG overlay | ④a |
| Google ADK (`google/adk-python`) | 18.9k | Pure ReAct + Sequential/Parallel/LoopAgent orchestration | ④a |
| AgentScope (`agentscope-ai/agentscope`) | 23.4k | Built-in ReAct, Plan is optional example | ④a |
| LangChain `plan-and-execute` | (deprecated) | — | ❌ removed from LangChain ecosystem |

**Also relevant (not yet verified but expected to be ④a)**: LangGraph, Vercel AI SDK, CrewAI, AutoGen.

**What this means**:

- Plan-and-Execute as a paradigm exists in academic papers (ReWOO, LLMCompiler, Plan-then-Execute arxiv 2509.08646)
- But no one has packaged it into a well-maintained, adoptable open-source framework
- LangChain tried and **deprecated** its implementation
- Every major framework that might have been ④b turned out to be ReAct-based when checked against its actual README

**Implication for the user's blog**: The "I'm doing Plan-and-Execute in production" story is not "rare compared to one or two other frameworks" — it is "rare compared to nothing else verifiable in public." This significantly sharpens the blog's central argument.

**Caveat**: We have not yet exhaustively searched for vertical-domain P&E frameworks (e.g., for medical, legal, or financial agents). The "empty" claim applies to general-purpose public open source. There may be proprietary enterprise systems (like the user's) that also do P&E but are not visible to this research.

---

## Open Questions (Become Findings When Answered)

- ~~**Q1**: Is Google ADK in category ④a or ④b?~~ — **Answered**: ④a (see `google-adk.research.md`).
- **Q2**: Does LangGraph's prebuilt `create_react_agent` define what "most people using LangGraph" actually do, or is it a minor API surface? If it dominates usage, LangGraph is effectively ④a even if its lower-level graph primitives could theoretically support P&E.
- **Q3**: Is there ANY public open-source P&E framework that is (a) maintained, (b) has Plan as a first-class primitive, and (c) is general-purpose? Exhaustive search not yet conducted. Academic implementations of ReWOO / LLMCompiler / Plan-then-Execute should be looked at.
- **Q4**: What happens when a ④a framework adds a Plan layer as an example? Does this turn it into ④b? (Answer: probably not — Plan-as-example is different from Plan-as-primitive. AgentScope's Meta Planner pattern is the test case.)
- **Q5**: Is there a ⑥ archetype we haven't seen yet? Research hasn't covered the "agent OS" direction (MemGPT-style or Letta-style systems that blur memory and control flow).
