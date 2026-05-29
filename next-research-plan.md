# Next Research Plan: Existing Agent Case Studies

Last Updated: 2026-05-21

## Purpose

This document records the next research stage for the internal Agent sharing
document. The goal is still not to write the final document yet. The goal is to
learn along the intended outline:

1. What existing agents look like.
2. What behavior patterns those agents use.
3. Why the Dayfold Agent chose a programmatic typed-plan architecture.

The previous planning-spectrum reading plan has been archived at
`archive/next-research-plan-2026-05-19.md`.

## Working Outline

The internal sharing document should likely follow this progression:

1. **Existing agents in practice**
   - Use a few representative systems to create shared context.
   - Avoid turning the section into a product encyclopedia.

2. **Behavior patterns behind existing agents**
   - Extract reusable primitives from public systems and the 17-pattern article.
   - Focus on control flow, plan shape, state, tools, evaluation, and safety.

3. **Plan representation as the core technical thread**
   - Compare attention-oriented plans with executable plans.
   - Clarify why most public implementations keep plans soft.

4. **Dayfold Agent positioning**
   - Start from business/runtime constraints.
   - Then explain the planner, validator, coordinator, capability, and typed
     dataflow design.

## Primary Study Queue

1. **Claude Code / Codex-style coding agents**
   - Question: how do general coding agents organize long tasks?
   - Focus areas: todo lists, tool loops, file-system actions, terminal feedback,
     tests as evaluators, and user interruption.
   - Output: `coding-agents-control-flow.case.md` records the source-level case
     note on coding-agent control flow and plan semantics.

2. **OpenClaw / personal-agent products**
   - Question: how do personal agents connect to real systems and side effects?
   - Focus areas: app/tool surface, permissions, action safety, autonomy limits,
     and whether planning is explicit or mostly model-mediated.
   - Expected output: a case note on personal-agent behavior patterns and safety
     boundaries.

3. **LangChain DeepAgents TodoListMiddleware**
   - Question: what does mainstream todo-mediated planning look like in a
     general-purpose agent framework?
   - Focus areas: todo as working memory, completion as model-declared state,
     context injection, and why this differs from executable plans.
   - Expected output: a note positioning TodoListMiddleware as attention-plan
     infrastructure.

## Supplementary References

The following systems are useful background, but they should not become primary
case studies for the internal sharing document unless a specific gap appears.

1. **CrewAI Planning / AgentScope PlanNotebook**
   - Question: how do framework-level plan objects work when execution remains
     ReAct-like or role-agent-driven?
   - Focus areas: plan appended to task description, mutable plan notebook,
     subtask state, plan management tools, and the boundary between plan-as-hint
     and plan-as-runtime-contract.
   - Expected output: a comparison note for soft structured plans.

2. **Magentic-One / Microsoft Magentic orchestration**
   - Question: what does manager-led planning look like in a multi-agent system?
   - Focus areas: task ledger, progress ledger, manager authority, worker
     delegation, stall detection, and replan behavior.
   - Expected output: a note on ledger plans and why this is different from
     typed executable plans.

3. **Google ADK Workflow Agents / LangGraph workflow**
   - Question: what does deterministic workflow control look like in modern
     agent frameworks?
   - Focus areas: developer-authored workflows, sequential/parallel/loop agents,
     graph state, reducers, checkpointing, and where LLMs sit inside the graph.
   - Expected output: a note on code-authored workflow as programmatic plan.

## Dayfold Comparison

1. **Dayfold Agent comparison table**
   - Question: how does Dayfold Agent differ from the above patterns?
   - Focus areas: LLM-authored typed plan, programmatic validation, coordinator
     scheduling, capability contracts, `input_bindings`, port values, and
     semantic-validation gaps.
   - Expected output: a table comparing Dayfold Agent against coding agents,
     personal agents, todo-mediated agents, and selected supplementary
     references when needed.

## Questions To Answer

For every case study, answer the same questions:

1. Who decides the next step?
2. What is the plan representation?
3. Is the plan for model attention, user visibility, manager coordination, or
   program execution?
4. Who marks a step as complete?
5. Where does state live?
6. What is the tool/action boundary?
7. How are errors detected and recovered?
8. How are side effects gated?
9. What would be hard to reproduce in a business-specific production agent?
10. What does this imply for Dayfold Agent?

## Expected Outputs

Before drafting the internal document, produce:

1. A short case note for each primary item in the study queue.
2. A plan-shape comparison table:
   - text plan
   - todo list
   - plan notebook
   - ledger plan
   - developer-authored workflow
   - executable plan
   - typed executable plan
3. A behavior-primitives table extracted from the 17-pattern article.
4. A Dayfold Agent comparison table.
5. A final internal sharing outline.

## Non-Goals

- Do not write the final internal sharing document yet.
- Do not present the 17 patterns as mutually exclusive architecture categories.
- Do not expose private Dayfold implementation details beyond abstract
  architecture and already-documented runtime concepts.
- Do not frame Dayfold Agent as universally better than ReAct, todo-mediated
  agents, or fixed workflow.

## Working Thesis

Most public general-purpose agents keep planning soft because soft plans preserve
flexibility and reduce framework-specific engineering burden. Business-specific
production agents can justify executable plans when they need lower token cost,
more predictable execution, auditable intermediate state, and stronger one-shot
success.

Dayfold Agent should be presented as one production answer to that tradeoff:
an LLM-authored typed plan, validated and executed by a deterministic runtime.
