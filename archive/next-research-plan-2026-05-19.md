# Next Research Plan: Planning Spectrum

Last Updated: 2026-05-19

## Purpose

This document records the next research stage for the Agent control-flow study.
The immediate goal is not to write a blog post yet. The goal is to read a small
set of high-signal public sources, then refine the taxonomy around planning:

> Is planning kept inside the model's attention, or externalized into a
> programmatically interpreted execution contract?

## Current Stage

The current stage is a reading and calibration pass led by Lin Guanguo.

The research should stay lightweight until the key sources are read. Do not
update the main synthesis documents yet unless a source clearly invalidates an
existing finding.

Progress update (2026-05-19): the first reading pass has started. Several
sources have been read, with the OpenAI practical guide and Anthropic's
*Building effective agents* currently providing the strongest framing value.

## Working Hypothesis

The next part of the research is likely centered on a planning spectrum rather
than a hard Workflow-vs-Agent boundary:

| Position | Planning Authority | Execution Authority | Representative Pattern |
|---|---|---|---|
| Fixed workflow | Developer-authored program | Program | Deterministic workflow / DAG |
| Programmatic P&E | LLM generates structured plan | Program validates and executes | Typed Plan-and-Execute |
| Todo-mediated ReAct | LLM maintains soft plan | LLM decides next action each turn | Todo list / scratchpad agent |
| Pure ReAct | LLM reasons step-by-step | LLM decides next action each turn | Tool-calling loop |

The likely writing angle:

> Todo lists are the mainstream planning interface for general-purpose agents.
> Programmatic Plan-and-Execute is a business-runtime strategy for low token
> cost, predictable execution, and higher one-shot success rates.

## Reading Queue

Primary sources to read first:

1. OpenAI, *A practical guide to building agents*
   - URL: https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/
   - Why: product-oriented definition of agents, orchestration, guardrails, and when deterministic automation is enough.

2. Anthropic, *Building effective agents*
   - URL: https://www.anthropic.com/engineering/building-effective-agents
   - Why: clean workflow/agent distinction and the argument for using the simplest architecture that works.

3. LangChain, *Deep Agents* documentation
   - URL: https://docs.langchain.com/oss/python/deepagents/overview
   - Why: public evidence that todo-list planning is becoming a mainstream general-agent pattern.

4. LangChain, To-do list middleware documentation
   - URL: https://docs.langchain.com/oss/python/langchain/middleware
   - Why: concrete implementation of todo-mediated planning as attention management rather than executable contract.

5. Google ADK, Workflow Agents
   - URL: https://google.github.io/adk-docs/agents/workflow-agents/
   - Why: official framing of deterministic workflow control in an agent SDK.

6. HumanLayer, *12 Factor Agents*
   - URL: https://www.humanlayer.dev/blog/12-factor-agents
   - Why: engineering argument for owning control flow while using LLMs for structured decisions.

7. arXiv, *Web Agents Should Adopt the Plan-Then-Execute Paradigm*
   - URL: https://arxiv.org/abs/2605.14290
   - Why: recent argument that task-level programmatic plans can improve safety and reliability for web agents.

8. arXiv, *Architecting Resilient LLM Agents: Secure Plan-then-Execute*
   - URL: https://arxiv.org/abs/2509.08646
   - Why: connects Plan-then-Execute with predictability, control-flow integrity, and prompt-injection resistance.

Secondary sources after the first pass:

- Zhoumo Programmer, *从0开发大模型的17种Agent架构演进详细拆解*
  - URL: https://mp.weixin.qq.com/s/5f0I2apY4oFsHrttANBOJg
  - Why: Chinese synthesis that reframes agent architecture as control-flow evolution, using consistent questions about state, topology, router, failure modes, and upgrade boundaries.
- FareedKhan-dev, *all-agentic-architectures*
  - URL: https://github.com/FareedKhan-dev/all-agentic-architectures
  - Why: Source project behind the 17-pattern article; useful as a runnable pattern catalog, not as primary evidence for industry defaults.
- Agno workflow documentation
  - URL: https://docs.agno.com/workflows/running-workflows
  - Why: Provides a compact vocabulary for Step, Router, Loop, Parallel, and Workflow, useful for describing control-flow primitives independent of LangGraph.
- Cognition, *Multi-Agents: What's Actually Working*
- AutoGen Magentic-One documentation
- CrewAI planning and experimental executor source
- AgentScope plan examples and plan model source
- LangGraph plan-and-execute tutorial / current replacement path

## Questions To Answer While Reading

1. What does each source mean by "planning"?
2. Is the plan an internal model aid, a user-visible artifact, or a runtime contract?
3. Who decides the next step after the plan exists: the model or the program?
4. What problem is the planning layer optimizing for: flexibility, cost, stability, safety, auditability, or UX?
5. Does the source recommend planning as a default path, a middleware/helper, an experimental feature, or a domain-specific technique?
6. What does each source imply about token cost and one-shot success?
7. Where does programmatic P&E become worth its engineering complexity?
8. What state, router, evaluator, or side-effect gate does each architecture add?

## Expected Outputs

After the reading pass, produce:

1. A short source note for each primary reading.
2. A comparison table: Todo-mediated ReAct vs programmatic P&E vs deterministic workflow.
3. A correction list for existing repository claims, especially where old "P&E is empty" or "P&E is deprecated" wording needs nuance.
4. A blog outline only after the above materials are stable.

## Non-Goals

- Do not write the blog before the reading pass.
- Do not expose private implementation details from the production Agent system.
- Do not frame programmatic P&E as a universal replacement for todo-list agents.
- Do not update the main taxonomy until the new sources have been read and reconciled with existing source-level research.

## Research Boundary

This part is about control flow and planning semantics. Memory and context
management should remain cross-referenced to `/Users/linguanguo/dev/llm-memory-research`
unless a source directly ties memory to planning behavior.
