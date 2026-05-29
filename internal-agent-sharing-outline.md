# Internal Agent Sharing Outline

Last Updated: 2026-05-21

## Purpose

This is a working outline for an internal sharing document about Agent
architecture and the Dayfold Agent design.

The goal is not to produce a broad Agent taxonomy. The goal is to build a
shared mental model for why modern Agent systems converge on similar control
primitives, and why the Dayfold Agent chooses a programmatic typed-plan runtime
instead of a purely attention-driven plan.

## Working Thesis

Agent development is less about choosing a named pattern and more about deciding
where to place state, control flow, action boundaries, and evaluation.

General-purpose agents often keep planning soft because they need flexibility.
Business-specific production agents can justify a more programmatic plan when
they need lower token cost, stronger one-shot stability, clearer auditability,
and recoverable execution.

## 1. What Existing Agents Are Trying To Solve

### Section Goal

Create shared context before discussing our own architecture.

This section should explain that the boundary between "workflow" and "agent" is
not sharp. When a workflow starts to plan, call tools, observe results, recover
from failure, and decide when to stop, it begins to resemble an agent. The
important question is not whether a system deserves the label "agent"; the
important question is who controls each part of execution.

### Topics To Cover

- Why Agent and Workflow are hard to separate cleanly.
- OpenAI and Anthropic's public framing of agents and workflows.
- Three representative external systems:
  - Claude Code / Codex-style coding agents.
  - OpenClaw / personal-agent products.
  - LangChain DeepAgents and todo-mediated agents.
- The common production question: how can an LLM reliably complete multi-step
  tasks with tools, state, and feedback?

### Claims To Test

- Existing agents differ less in whether they are "agents" and more in where
  they put control authority.
- Coding agents tolerate flexible loops and higher token use because the task
  space is open-ended.
- Business agents often need more constrained execution because the action
  surface and success criteria are product-specific.

### Materials To Add Later

- Short case notes from the current research plan.
- Quotes from OpenAI and Anthropic official agent guides.
- A compact comparison table of the three representative external systems.

## 2. Implementation Patterns As Control Primitives

### Section Goal

Use the 17-pattern article as a useful entry point, but avoid treating those
patterns as mutually exclusive architecture categories.

The section should reframe agent architectures as combinations of higher-level
control primitives.

### Core Primitives

| Primitive | Question It Answers |
|---|---|
| State | What does the system explicitly remember? |
| Planner | Who decides what should happen next? |
| Router / Scheduler | Who selects the next executable step? |
| Tool / Capability | What actions can the system take? |
| Evaluator / Validator | Who decides whether the result is valid? |
| Loop | When does the system continue, retry, stop, or escalate? |
| Memory | How does previous history affect the current task? |
| Human / Side-Effect Gate | When must a human or policy boundary approve action? |

### Common Implementation Families

| Family | Plan Shape | Control Owner | Typical Strength | Typical Weakness |
|---|---|---|---|---|
| ReAct / Tool-use Agent | Mostly implicit | LLM | Flexible exploration | Higher token cost and weaker predictability |
| Todo / Text Plan Agent | Text or todo list | LLM or manager | Better task attention and progress tracking | Completion is often model-declared |
| Manager / Orchestrator Agent | Ledger, task list, or delegation state | Manager agent | Handles multiple workers or tools | Manager reasoning stays token-heavy |
| Workflow / Graph Agent | Developer-authored graph | Program | Stable and inspectable | Dynamic behavior must be designed upfront |
| Programmatic Plan-and-Execute | Structured executable plan | LLM plans, program executes | Lower runtime ambiguity | Requires domain schema and runtime validation |

### Claims To Test

- Many named patterns are combinations of the same primitives.
- A "new architecture" is only meaningful if it adds a new explicit state,
  routing rule, evaluator, side-effect boundary, or recovery path.
- Plan is one important primitive, but not the only one.

### Materials To Add Later

- The existing `agent-control-primitives.reference.md` table.
- Examples from TodoListMiddleware, PlanNotebook, Magentic ledgers, and
  workflow graphs.
- A short note on why "plan generates workflow" is less common in open-source
  defaults than in business-specific runtimes.

## 3. Our Business Problem And Agent Design

### Section Goal

Explain why the Dayfold Agent does not simply use a general-purpose ReAct or
todo-mediated agent loop.

This section should start from product/runtime constraints, then explain the
architecture as a response to those constraints.

### Problem Constraints

- Low token cost.
- High one-shot success rate.
- Stable execution over multi-step business tasks.
- Auditable intermediate state.
- Recoverable failures.
- Clear tool and business-action boundaries.
- Ability to reuse product-specific capability contracts.

### Design Position

The Dayfold Agent can be described as:

> Programmatic Plan-and-Execute with Typed Dataflow.

The model is responsible for selecting and composing business capabilities. The
runtime is responsible for validating, scheduling, binding, executing, and
recovering from that plan.

### Control Primitives In Our Design

| Primitive | Dayfold Interpretation |
|---|---|
| State | Explicit workflow state, step state, and typed port values. |
| Planner | LLM generates a typed execution plan. |
| Scheduler | Program determines dependency readiness and execution waves. |
| Capability | Business actions are exposed as typed capability nodes. |
| Binding | Plan declares how upstream outputs flow into downstream inputs. |
| Validator / Evaluator | Structural validation, type validation, and execution feedback. |
| Loop | Replanning or continuation happens around the deterministic runtime. |

### Claims To Test

- This design shifts repeated reasoning out of the model and into the runtime.
- It is closer to a compiled Manager / Orchestrator-workers design than to a
  free-form multi-agent chat.
- Typed dataflow improves auditability and handoff reliability, but does not
  eliminate semantic correctness problems.

### Materials To Add Later

- Desensitized excerpts from `my-agent-2.0.research.md`.
- A Dayfold comparison table against coding agents, todo agents,
  manager-orchestrator agents, and fixed workflows.
- A simple diagram showing planner, coordinator, capabilities, bindings, and
  result aggregation.

## 4. Tradeoffs And Retrospective

### Section Goal

Avoid presenting the design as universally better. Make the tradeoffs explicit
and connect them to what could be improved if the system were built again.

### Comparisons

| Compared With | Dayfold Advantage | Dayfold Cost |
|---|---|---|
| Todo / Text Plan Agent | More deterministic execution and stronger auditability. | Less open-ended flexibility. |
| Manager / Orchestrator Agent | Lower repeated manager-token cost and clearer runtime authority. | Requires a stronger runtime contract. |
| Fixed Workflow | More dynamic than a hand-authored DAG. | Planner and schema complexity increase. |
| Multi-Agent Role Framework | Focuses on typed execution rather than role count. | Less natural for open-ended collaborative reasoning. |

### Retrospective Points

- Research public agent patterns earlier.
- Define the plan intermediate representation earlier.
- Build an evaluation harness earlier.
- Separate soft plan, executable plan, and runtime state earlier.
- Add plan-template fast paths for common high-frequency tasks.
- Add stronger semantic validation, not only structural and type validation.

### Claims To Test

- The main cost of programmatic P&E is not the plan object itself; it is the
  surrounding contract: schema, bindings, validators, capability signatures,
  recovery rules, and observability.
- Evaluation becomes a first-class engineering problem once the runtime is
  expected to be stable and low-token.

### Materials To Add Later

- Failure-mode examples, desensitized.
- Metrics candidates: one-shot success, token cost, step failure rate, replan
  rate, human-intervention rate, and semantic error rate.
- Concrete "what I would do differently" notes.

## Expansion Plan

1. Expand section 1 with the three representative external systems: coding
   agents, OpenClaw personal agents, and LangChain todo-mediated agents.
2. Expand section 2 with a compact primitives table and plan-shape comparison.
3. Expand section 3 with a desensitized Dayfold architecture explanation.
4. Expand section 4 with measured or anecdotal tradeoffs and retrospective
   lessons.

The document should stay focused. If a topic becomes too large, move it into a
separate case note and summarize only the conclusion here.
