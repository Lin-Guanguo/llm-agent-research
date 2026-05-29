---
description: Note on interpreting Dayfold Agent as a workflow-centric agent system with typed LLM subagents rather than a top-level LLM manager.
last_updated: 2026-05-29
---

# Workflow-Centric Agent System

## Core Finding

The current Dayfold architecture is better described as a **workflow-centric agent system** than as a top-level LLM manager agent.

The top-level control authority is a programmatic workflow runtime: a state machine / coordinator that owns validation, scheduling, execution, repair, fallback, and state updates. LLM calls participate as typed decision or artifact producers inside that runtime.

The high-level shape is:

```text
Workflow runtime
-> Planner LLM: structured ExecutionPlan
-> Validator: validate / repair / reject
-> Coordinator: deterministic scheduling
-> Worker or subagent: typed input -> typed output
-> State update
-> Replan / continue / finish
```

One concise formulation:

```text
A workflow-centric agent system: deterministic runtime owns control flow,
while LLM subagents produce typed artifacts under validated contracts.
```

## Tool Call vs Structured Output

A common agent shape is:

```text
LLM -> tool call -> observation -> LLM -> tool call -> ...
```

This is the natural shape for coding agents and open-ended research agents. The model stays in the outer control loop and repeatedly decides the next action.

Dayfold has a different emphasis:

```text
LLM -> structured plan / structured result
Runtime -> validate / execute / schedule / repair
```

The difference is not whether one system is "more agentic." The difference is where control authority lives.

| Pattern | Control owner | Model output | Best fit |
|---|---|---|---|
| Tool-call agent | LLM loop | next tool and arguments | open-ended exploration, coding, research |
| Workflow-centric agent | programmatic runtime | typed artifact or typed decision | business processes, low token cost, high stability |

Tool calls have stronger protocol-level action semantics: the model says "call this tool now." Structured outputs are declarative artifacts: their business semantics come from the runtime contract.

The useful distinction is:

```text
Tool call = imperative action
Structured output = declarative artifact
Executable plan = macro tool argument
```

## Plan As Macro Tool Argument

A typed executable plan can be viewed as the argument to a macro tool:

```text
Planner LLM
-> ExecutionPlan
-> execute_plan(plan)
-> final / partial state
```

The executor is conceptually a large tool, but it does not have to be expressed through the API-level tool-call protocol. The key is that the plan is a typed object that the runtime can validate, compile, schedule, and recover.

This explains the Dayfold tradeoff:

- The system is not avoiding tools; it is consolidating many low-level actions behind a high-level business executor.
- The model supplies the plan parameters.
- The runtime owns execution and recovery.
- Token cost drops because not every step has to return to the planner.
- Reliability improves because scheduling and validation are deterministic.
- Engineering complexity rises because the plan schema, binding system, validators, and repair policies must be explicit.

## Subagent Boundary

The best boundary is not "all tool call" or "all structured output." It is layered:

```text
External contract: structured input/output
Internal behavior: optional tool-call loop
```

Subagents should expose typed contracts to the workflow. Internally, they may use tool calls when they need exploration, retrieval, API access, or observation-driven decisions.

| Component | Recommended form |
|---|---|
| Planner | Structured output: `ExecutionPlan` |
| Coordinator | Programmatic logic; avoid free LLM control |
| Worker / capability | Typed input -> typed output |
| Researcher / explorer | Internal tool-call loop, final structured summary |
| Validator / reviewer | Structured verdict and error list |
| Summarizer | Text or structured output depending on downstream consumption |

This keeps both strengths:

- tool calls for open-ended exploration;
- structured output for validated handoff and runtime execution.

## Global State vs Prompt Context

In a workflow-centric agent system, the global context should be treated as runtime-owned state, not as a default prompt payload for every subagent.

The important distinction is:

```text
Global state != prompt context
```

Global state can hold canonical runtime facts: user request, plan, step outputs, run metadata, artifacts, errors, and checkpoints. It does not necessarily consume tokens while it stays in the runtime or storage layer. However, it still affects system complexity because it may later be selected, projected, or interpreted by a subagent.

Prompt context is a derived view:

```text
global state -> context projector -> scoped typed input -> subagent
```

This means the safer pattern is not to let every subagent see the full global context. Each subagent should receive a narrow typed input that contains exactly what the node needs for its local responsibility.

| Layer | Purpose | Default LLM visibility |
|---|---|---|
| Global state | Canonical runtime state and cross-step facts | No |
| Artifact store | Large pages, images, history, generated assets, provenance | No |
| Typed input | The local data required by a specific subagent | Yes |
| Prompt projection | A selected, compressed view of relevant state | Yes |
| Telemetry / logs | Debugging, evaluation, repair records, rejected outputs | No |

This is necessary because broad global context can quietly undo the benefit of typed workflow design. If every node sees the same large implicit context, the system drifts back toward a prompt-driven agent where nodes infer too much from ambient information.

Good context hygiene asks:

- Who consumes this field?
- Is it canonical truth or only a model hypothesis?
- Is it turn-scoped, step-scoped, or persistent?
- Does it duplicate a typed input field?
- Should it be visible to the model, or only to the runtime?

The working rule is:

```text
SubagentInput = exactly what this node needs
SubagentOutput = exactly what downstream/runtime consumes
GlobalState = runtime-owned source of truth
PromptContext = derived view, not raw global dump
```

## Product Process State Machine

The current `request_type=chat` design should be understood as a product-level
process policy, not as a final claim that all process transitions must live
outside the planner forever.

The reason for the split is practical: the story-start path already has
business logic outside the planner. Chat mode collects and confirms intent,
emits a form/ready handoff artifact, and then an external caller can start a
new `request_type=story` turn with form-derived input.

The current shape is:

```text
/process request_type=chat
-> planner_global_chat
-> reply_message / chat_creation_form / chat_creation_ready
-> external caller observes ready/form artifact
-> /process request_type=story with form-derived payload
-> story planner
-> story pipeline
```

This means the planner does not always see the full action set. Instead, each
process state exposes a smaller action subset:

| Process state | Planner surface | Main responsibility |
|---|---|---|
| `chat` | reply, form, ready | Collect intent and produce a confirmed handoff artifact. |
| `story` | story pipeline capabilities | Execute the formal story generation/editing workflow. |
| `onboarding` | onboarding-specific planner and prompt overrides | Run the onboarding creation/edit chain. |

The transition can still be model-triggered in practice: the chat planner
chooses `chat_creation_ready`, but the actual switch into the story workflow is
performed by the external product/backend call. That gives two useful
properties:

1. **Smaller planner prompt and action space.** The planner only needs the
   actions relevant to the current state. Capabilities belonging to other
   states can be hidden; the planner only needs to know how to emit the
   transition artifact.
2. **Externally visible state transitions.** Because the product/backend sees
   the state switch, it can run hooks before entering the next stage. For
   example, entering story generation can trigger billing, quota checks,
   eligibility checks, review gates, or other product lifecycle logic.

The tradeoff is that the workflow boundary becomes split. Some transitions are
inside the Agent runtime, while some are product protocol transitions around
the Agent. That split is acceptable when the handoff artifact is typed and
stable, but it should be treated as a deliberate process contract rather than
as ordinary prompt context.

## Why This Matters

For a general coding agent, a top-level LLM tool loop is often the right shape because the problem is open-ended and the environment must be inspected step by step.

For a business workflow agent, the stronger requirement is often predictable execution:

- low token consumption
- higher one-shot success rate
- auditable plan artifacts
- recoverable execution state
- validated business contracts
- deterministic repair
- controlled side effects

This is why Dayfold should be positioned as a workflow-centric agent system rather than as a conventional ReAct-style tool-call agent.

The main caveat is that structured output does not automatically make the system correct. It only makes the artifact parseable. The runtime still needs business validators, deterministic repair, retry/fallback policy, and telemetry.
