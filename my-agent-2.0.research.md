---
description: Research note on the production Dayfold Agent architecture as programmatic Plan-and-Execute with typed dataflow.
last_updated: 2026-05-21
---

# Production Dayfold Agent Architecture

## Sources

Primary source code and internal docs:

- `/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/architecture.md:7` describes the system as a LangGraph workflow engine that receives HTTP requests, uses an LLM planner, executes capability nodes, and streams events.
- `/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/architecture.md:15` identifies the two defining choices: every user turn is replanned, and node connections are produced by the planner through typed `input_bindings`.
- `/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/tech-review.md:11` frames the system as an LLM-driven workflow engine with Pydantic contracts and multi-layer plan validation.
- `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/planning/planner_node.py:140` is the planner runtime.
- `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/coordinator/coordinator_node.py:148` is the deterministic scheduler and interruption point.
- `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/capabilities/dispatcher.py:188` is the capability execution boundary.
- `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/planning/validator.py:1221` is the plan validation orchestrator.
- `/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/routers/process_compat/adapter.py:113` records policy prompt defaults for onboarding and chat requests.
- `/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/workflow/planning/planner_node.py:291` resolves the active planner prompt id per turn.
- `/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/workflow/workflow_langgraph.py:319` loads planner few-shot references by planner policy.
- `/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/prompt/profiles/planning_global_chat.yaml:1` documents the global-chat planner policy and legal plan shapes.
- `/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/workflow/nodes/chat_creation_ready.py:1` defines the chat submit materialization node.
- `/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/docs/plans/119-global-chat-process-interface-design.md:490` documents the chat-to-story handoff through a backend-generated story request.

Research context:

- `/Users/linguanguo/dev/llm-agent-research/summary.md:13` defines the control-flow taxonomy by asking who decides execution flow.
- `/Users/linguanguo/dev/llm-agent-research/summary.md:23` places public Plan-and-Execute frameworks mostly in experimental, middleware, or historical positions.
- `/Users/linguanguo/dev/llm-agent-research/planning-spectrum-reading-notes.md:18` records the OpenAI Manager framing.
- `/Users/linguanguo/dev/llm-agent-research/planning-spectrum-reading-notes.md:38` records the Anthropic Orchestrator-workers framing.
- `/Users/linguanguo/dev/llm-agent-research/agent-paradigm-survey.reference.md:90` compares dependency-expression mechanisms, including typed dataflow.
- `/Users/linguanguo/dev/llm-agent-research/agent-paradigm-survey.reference.md:238` positions Agent 2.0 as Plan-and-Execute plus typed dataflow plus programmatic validation.
- `/Users/linguanguo/dev/llm-agent-research/multi-agent-complexity.reference.md:251` argues that typed dataflow removes some handoff failures structurally but leaves verification and single-call reasoning failures unresolved.

This note is intentionally desensitized. It focuses on architecture, control primitives, typed contracts, validation, and tradeoffs. Domain examples are kept generic.

## Overview

The production Dayfold Agent is best described as **programmatic Plan-and-Execute with typed dataflow**.

It is not a fixed workflow: every user turn invokes a planner that can produce a new typed execution plan. It is also not ordinary ReAct: once a plan exists, deterministic code, not the model, owns dependency readiness, scheduling, input binding, execution fan-out, error routing, and interruption checks.

The architecture is therefore a hybrid:

| Layer | Authority | Main artifact |
|---|---|---|
| Intent decomposition | LLM planner | `ExecutionPlan` / `PlannerOutputModel` |
| Runtime control | Programmatic coordinator | pending/completed step state, ready-step selection |
| Data movement | Typed port system | `input_bindings`, `PortValue`, Pydantic I/O |
| Capability behavior | Specialized node implementations | per-capability input/output models |
| Recovery and observability | Runtime infrastructure | checkpoints, events, run state |

This makes the system closer to a **compiled Manager** or **compiled Orchestrator-workers** design than to a normal multi-agent system. The planner decides the structure once per turn; the runtime compiles that structure into deterministic execution waves.

## Control Primitives

The system combines five control primitives:

1. **Replanning per turn.** The docs state that each user input causes a new plan to be generated from current capabilities and completed steps, instead of relying on a fixed DAG (`/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/architecture.md:19`).
2. **Plan artifact.** The plan is a Pydantic model whose steps contain `step_id`, `capability_id`, and `input_bindings` (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/contracts/plan_contract.py:13`).
3. **Dataflow scheduling.** The coordinator expands the plan, computes pending steps, selects ready steps, and uses `Send` for parallel fan-out (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/coordinator/coordinator_node.py:163`, `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/coordinator/coordinator_node.py:287`).
4. **Typed capability dispatch.** The dispatcher resolves the capability, materializes bindings, validates input with the capability's Pydantic input model, runs the handler, and writes outputs back as port values (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/capabilities/dispatcher.py:261`, `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/capabilities/dispatcher.py:290`, `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/capabilities/dispatcher.py:373`).
5. **Human/run interruption.** The coordinator checks for new user input before moving to the next wave and routes back toward turn preparation/replanning when a new message arrives (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/coordinator/coordinator_node.py:179`).

This set of primitives matters because it separates **semantic choice** from **runtime authority**. The LLM chooses the plan; the runtime enforces the plan contract.

## Planner Policy State Machine

The newer branch adds an outer planner-policy layer on top of the original typed Plan-and-Execute runtime. This is not a separate graph-level `StateMachine` class; it is a per-turn policy state machine distributed across request adaptation, prompt selection, planner few-shots, validation, and fallback behavior.

The simplified policy model is:

```text
incoming request / continuation state
  -> planner policy
      story      -> planner id: planning
      onboarding -> planner id: planner_onboarding
      chat       -> planner id: planner_global_chat
  -> policy-specific prompt + few-shots + capability constraints
  -> typed plan
  -> deterministic runtime execution
```

```text
request_type=chat
        |
        v
+------------------------------------------------+
| chat                                           |
| planner = planner_global_chat                  |
| reply / refine / update creation form          |
+--------------------------+---------------------+
        ^                  |
        | next chat turn   | final creation form / ready
        +------------------+ caller auto-calls request_type=story
                           v
request_type=story/default
        |
        v
+------------------------------------------------+
| story                                          |
| planner = planning                             |
| normal story generation/edit rounds            |
+--------------------------+---------------------+
        ^                  |
        | normal continuation
        +------------------+

request_type=onboarding
        |
        v
+------------------------------------------------+
| onboarding                                     |
| planner = planner_onboarding                   |
| onboarding/edit rounds                         |
| continuation restores policy from metadata     |
+--------------------------+---------------------+
        ^                  |
        | onboarding continuation
        +------------------+
```

The request adapter injects policy prompt ids from the product entry point. Onboarding requests set `capability_prompt_ids["planner"] = "planner_onboarding"` and also override selected downstream prompts; chat requests set `capability_prompt_ids["planner"] = "planner_global_chat"` (`/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/routers/process_compat/adapter.py:113`, `/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/routers/process_compat/adapter.py:214`). The planner then resolves the active prompt id from the latest turn and uses it to choose planner behavior (`/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/workflow/planning/planner_node.py:291`). Few-shot examples are also keyed by policy: `planning`, `planner_onboarding`, and `planner_global_chat` (`/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/workflow/workflow_langgraph.py:319`).

This changes the original architecture in an important way. The planner is no longer just "one planner that sees all capabilities." It is now a policy-selected planner surface:

| Policy | Primary role | Runtime implication |
|---|---|---|
| `planning` | Normal story/edit generation. | Full story pipeline is available under standard validation. |
| `planner_onboarding` | Onboarding-specific first run and edit continuation. | Uses onboarding topology and persistent prompt overrides for selected downstream capabilities. |
| `planner_global_chat` | Conversational collection and confirmation before generation. | Allows only chat-safe materialization shapes and blacklists render/audio/story-editing nodes. |

The chat policy is especially explicit. The prompt profile documents four legal shapes: `[story_writer, chat_creation_form]`, `[chat_creation_form]`, `[chat_creation_ready]`, or empty `execution_steps` with a `reply_message` (`/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/prompt/profiles/planning_global_chat.yaml:13`). The validator reinforces this with chat-only rules: blacklist story-pipeline producers, enforce form/ready mutual exclusion, enforce singleton form/ready, and require a user-visible output (`/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/workflow/planning/validator.py:1197`). Chat mode also caps replan attempts lower than story mode and falls back to text instead of a story pipeline when the chat plan cannot be repaired (`/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/workflow/planning/planner_node.py:368`, `/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/workflow/planning/planner_node.py:393`).

This is best described as a **planner policy state machine** rather than a new agent architecture. It keeps the same typed plan/runtime contract, but the available planner behavior is selected by product mode.

## Chat-To-Story Handoff

The chat mode introduces a second, product-facing state machine outside the Agent runtime.

In chat mode, the Agent may stay in `planner_global_chat` for multiple turns: replying, asking for details, refining the draft, or updating the creation form. When the chat state is ready to hand off, the Agent emits the final creation form/ready payload. That form is not merely UI data. It is also a machine-readable handoff payload for the next automatic story request.

```text
chat request
  -> planner_global_chat
  -> chat_creation_form / chat_creation_ready
  -> backend reads final creation form
  -> backend automatically calls /process request_type=story
  -> story planner receives fields derived from the final form
```

The `chat_creation_ready` node is a pure materialization step. It receives the planner's final `story_text` and `characters` snapshot, emits a `chat_creation_ready` business message, and does not run an LLM (`/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/src/agent/workflow/nodes/chat_creation_ready.py:1`). The interface design states that after ready, the backend re-calls `/process` with `request_type=story`, using the approved chat form content as the source for the next story request (`/Users/linguanguo/dev/dayfold_webapp-fix-2/agent/docs/plans/119-global-chat-process-interface-design.md:490`).

This means the full system has two nested state machines:

| Layer | Owner | State transition |
|---|---|---|
| Agent planner policy | Agent service | request/policy state selects `planning`, `planner_onboarding`, or `planner_global_chat`. |
| Product workflow | Caller/backend | final chat form triggers review/precheck and an automatic `request_type=story` call. |

This is a useful but leaky boundary. The caller/backend must understand the creation form as a handoff contract and translate it into the next story request. That puts some product workflow semantics outside the Agent runtime, even though the handoff payload is authored by the Agent. The leakage is acceptable if the boundary is kept stable: business lifecycle checks and approval can live outside the Agent, while executable-plan legality, capability constraints, and typed dataflow validation should remain inside the Agent.

## Plan / Runtime Contract

The core contract is small:

```python
ExecutionStep:
    step_id: str
    capability_id: str
    input_bindings: Mapping[str, BindingSource]

ExecutionPlan:
    execution_steps: tuple[ExecutionStep, ...]
    output_summary: str
```

The important part is not the object shape alone. It is the contract around that shape:

- A step names a capability, not an arbitrary agent persona.
- Input bindings are explicit data dependencies, not hidden conversation context.
- A plan may be rejected before execution.
- A plan can be normalized after validation to repair known model mistakes.
- Runtime state distinguishes plan, pending steps, completed steps, active step, port values, context messages, and step errors (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/graph_builder.py:130`).

The planner code also has production hardening that pure pattern descriptions usually omit: predefined-plan override handling, fallback plan generation, structured output unwrapping, alias normalization, literal binding coercion, and per-attempt validation feedback (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/planning/planner_node.py:200`, `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/planning/planner_node.py:456`, `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/planning/planner_node.py:631`, `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/planning/planner_node.py:736`).

## Typed Dataflow

The data plane is the distinctive part of the design. The planner does not only choose tools; it wires fields between typed producers and consumers.

`PortMapper.materialize()` supports literal values, direct step-field references, index/slice expressions, multi-source merge, and type coercion through the registry (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/coordinator/port_mapper.py:156`, `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/coordinator/port_mapper.py:186`). Capability output fields become port keys such as `{step_id}.{field_name}` (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/capabilities/dispatcher.py:373`).

The docs explicitly position this as a port-based dataflow system: step outputs become `PortValue` entries and downstream steps reference them through `input_bindings` (`/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/architecture.md:352`). They also describe Protocol-based structural consumption so producers can preserve provenance while consumers accept an interface shape (`/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/architecture.md:380`).

The list-operation subsystem shows the same philosophy at a finer granularity: `BatchListOp` carries both a snapshot and operation hints, with the invariant that snapshot is truth and ops are hints (`/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/architecture.md:447`, `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/schemas/list_op.py:1`). This is an important production concession. The system wants incremental edit semantics, but it does not trust noisy structural deltas more than the canonical resulting view.

Relative to the earlier research taxonomy, this maps to the strictest dependency-expression mechanism: **field-level typed dataflow**, not ReWOO-style string substitution, not LLMCompiler-style positional variables, and not plain JSON tasks with whole-object dependencies (`/Users/linguanguo/dev/llm-agent-research/agent-paradigm-survey.reference.md:151`).

## Validation And Recovery

The plan validator is no longer just a structural checker. It encodes both generic graph rules and production-specific invariants.

Examples include unknown capability, missing binding, invalid source references, field/type mismatch, dynamic output conflicts, dependency cycles, duplicate capability in a plan, reused completed step IDs, and required pipeline-pair rules (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/planning/validator.py:43`, `/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/planning/validator.py:1192`).

The validator is paired with a replan policy. It retries planning up to a bounded number of attempts, feeding validation issues back into the next planner call (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/planning/replan.py:25`). If planning still fails, the planner can emit a fallback plan rather than leaving the run without a path (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/planning/planner_node.py:330`).

This is a useful distinction from ReAct-style retry. The retry loop is not "try the same action again"; it is "repair the plan artifact before runtime execution." That makes failures inspectable and turns validation into a planner training signal.

## Position Relative To Common Patterns

| Pattern | Similarity | Key Difference |
|---|---|---|
| Fixed workflow | Uses deterministic runtime edges, typed nodes, checkpoints, and programmatic scheduling. | The logical DAG is not authored once by developers. It is generated per turn by the planner. |
| OpenAI Manager | Resembles a manager delegating to specialized workers, as summarized in `/Users/linguanguo/dev/llm-agent-research/planning-spectrum-reading-notes.md:24`. | Manager authority is compiled into a plan; workers are capabilities with typed contracts, not peer agents with open-ended autonomy. |
| Anthropic Orchestrator-workers | Resembles central decomposition into worker tasks, as summarized in `/Users/linguanguo/dev/llm-agent-research/planning-spectrum-reading-notes.md:47`. | Mid-run coordination is programmatic. The orchestrator does not repeatedly decide every delegation step after execution starts. |
| Todo-mediated ReAct | Shares the idea that a plan/todo artifact externalizes task state. | Todo ReAct keeps the LLM in the action loop; this design externalizes executable dependencies into a typed DAG and lets code schedule it. |
| Classical Plan-and-Execute | Shares the plan-then-run split. | The plan is not only a list of steps. It is a typed dataflow graph with field bindings, validation, checkpointable state, and parallel wave execution. |
| Multi-agent role frameworks | Has specialized workers. | It avoids most peer-agent handoff semantics. Capability nodes are typed functions/services, not autonomous agents passing summaries to each other. |

The concise placement:

> Dayfold Agent is **runtime-authored workflow**: the LLM authors the DAG at turn start, then deterministic code executes the DAG with typed dataflow semantics.

This matches the research note that programmatic P&E can be read as a compiled variant of Manager / Orchestrator-workers: an LLM creates the plan, while a programmatic runtime validates, schedules, and executes specialized workers (`/Users/linguanguo/dev/llm-agent-research/planning-spectrum-reading-notes.md:32`).

## What The Design Gets Right

1. **It puts structure where production needs it.** The plan, port values, events, checkpoints, and state reducers are explicit. That supports auditability, recovery, and debugging.
2. **It avoids peer-agent trace loss.** Workers do not negotiate through lossy natural-language handoffs. They consume typed inputs and publish typed outputs.
3. **It makes parallelism natural.** Once dependencies are explicit, ready steps can be fanned out through `Send`.
4. **It treats LLM output as fallible.** Planner output is schema-normalized, validated, retried, and sometimes replaced by fallback plans.
5. **It preserves incremental editing semantics.** The `prev_output` injection and `BatchListOp` snapshot/ops model give capabilities a way to reason about prior outputs without redoing everything blindly (`/Users/linguanguo/dev/dayfold_webapp-new/agent/src/agent/workflow/capabilities/dispatcher.py:307`, `/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/architecture.md:459`).

## Complexity Tax

The design pays real complexity in exchange for control:

1. **Planner correctness becomes a platform problem.** The validator already contains many generic and domain-invariant rules. That is a sign of production hardening, but also a sign that the LLM planner needs a guardrail ecosystem.
2. **Binding syntax becomes a language.** Literal prefixes, tuple merges, index expressions, `_latest` resolution, dynamic outputs, and protocol generation types are useful, but they are now a DSL that must be documented and tested.
3. **State growth is structural.** Append/merge reducers make recovery and parallelism workable, but docs already identify `port_values` and `context_messages` as unbounded growth risks (`/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/tech-review.md:23`, `/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/tech-review.md:455`).
4. **Semantic correctness is not guaranteed.** A structurally valid plan can still be the wrong plan. The tech review calls this out directly: the validator checks structure, not semantic fit (`/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/tech-review.md:406`).
5. **Production invariants leak into the planner layer.** Rules such as "only one step per capability per round" and paired capability requirements stabilize behavior, but they partially reintroduce fixed-workflow constraints on top of a flexible planner (`/Users/linguanguo/dev/dayfold_webapp-new/agent/docs/architecture.md:29`).
6. **Product workflow semantics leak across the service boundary.** The chat creation form is both user-facing UI state and the handoff contract for the automatic story request. This reduces ambiguity in the chat-to-story transition, but it means the caller/backend must understand part of the Agent's mode lifecycle.

The important conclusion is not that the complexity is bad. It is that the architecture should be evaluated as a trade: it buys reproducible data movement, auditability, parallel execution, and recoverability by accepting a planner DSL, validation surface, and state-management burden.

## What To Research Next

1. **Evaluation harness for planner quality.** Measure first-pass valid-plan rate, replan convergence rate, fallback frequency, semantic correction rate, and user-visible failure categories by scenario.
2. **Semantic validation.** Structural validation is mature. The next gap is detecting "valid but wrong" plans. Candidate methods: task-specific assertions, HITL review points, lightweight verifier models, and replay-based regression suites.
3. **State compaction policy.** Define when old port values and context messages become archival rather than active planner/runtime state.
4. **Plan template fast path.** For high-frequency flows, compare LLM-authored plans against prevalidated templates with parameter slots. This tests whether common paths can get fixed-workflow reliability without removing planner flexibility for novel requests.
5. **Todo ReAct comparison.** Run the same task suite against a Todo-mediated ReAct baseline. Compare latency, token cost, failure recoverability, audit quality, and user interruption behavior.
6. **Complexity budget.** Track how many validator rules, binding forms, and capability-specific exceptions are needed over time. If these grow faster than capability count, the planner contract may be too expressive or too underspecified.
7. **Handoff contract boundary.** Decide which chat-to-story responsibilities should remain in the caller/backend and which should be pulled into the Agent service. The key question is whether the creation form should stay as a shared product contract or become an internal Agent transition artifact.

## Conclusion

Dayfold Agent occupies a narrow and interesting point in the agent architecture space. It is not merely "an agent with tools," and it is not merely "a workflow with LLM nodes." It is a production system where the LLM authors a typed execution graph per turn, then deterministic runtime code validates and executes that graph.

That makes it a strong fit for domains where outputs are structured, intermediate artifacts matter, user edits are common, and production observability is valuable. It is a weaker fit for open-ended environments where the right next action depends heavily on fresh observations after every step.

The next research priority should not be more elaborate dataflow. The system already has enough structure to capture the benefits of typed P&E. The higher-leverage work is proving, with evaluation data, where this structure beats simpler Todo-mediated ReAct or fixed workflow baselines, and where its complexity tax is not justified.
