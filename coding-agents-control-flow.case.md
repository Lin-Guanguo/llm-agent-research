# Case: Claude Code / Codex-Style Coding Agents

Last Updated: 2026-05-21

## Purpose

This note studies Claude Code and Codex-style coding agents from the perspective
of **agent control flow**, not memory management or product UX.

The main question is:

> In coding agents, is the plan an executable runtime contract, or is it a
> model-attention and collaboration artifact?

The short answer from the local source review is: coding agents have explicit
plans, todos, plan modes, subagents, tool boundaries, and verification
mechanisms, but their main execution model remains a model-driven tool loop.
The plan usually helps the model and user coordinate; it does not become a
deterministic workflow IR that the runtime schedules step by step.

## Sources

Codex source:

- `/Users/linguanguo/dev/codex/codex-rs/core/src/session/turn.rs:120`
  documents the main turn loop: model output is either tool calls or assistant
  messages; tool calls are executed and fed back into the next sampling request.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/session/turn.rs:432`
  constructs the next model input from conversation history before each
  sampling request.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/session/turn.rs:461`
  uses `needs_follow_up` from model/tool results and pending user input to decide
  whether the loop continues.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/session/turn.rs:1835`
  handles the streaming sampling request and records tool futures and assistant
  messages.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/stream_events_utils.rs:202`
  queues tool execution when a completed model output item is a tool call.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/tools/handlers/plan.rs:45`
  implements `update_plan` as a tool that sends a `PlanUpdate` event.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/tools/handlers/plan.rs:74`
  rejects `update_plan` in Codex Plan mode, explicitly separating plan mode from
  the todo/checklist tool.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/tools/handlers/plan_spec.rs:6`
  defines the `update_plan` schema as a list of `{ step, status }` items plus
  optional explanation.
- `/Users/linguanguo/dev/codex/codex-rs/core/gpt_5_2_prompt.md:38`
  frames `update_plan` as progress tracking and user-visible collaboration.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/tools/handlers/multi_agents.rs:1`
  implements the collaboration tool surface for subagents.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/codex_delegate.rs:59`
  starts interactive sub-Codex threads, with approval requests handled through
  the parent session.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/tools/handlers/shell.rs:108`
  routes shell-like tool calls through permission and sandbox checks.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/tools/runtimes/shell.rs:1`
  describes the shell runtime as approval-aware and sandbox-aware.

Claude Code local sourcemap source:

- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/query.ts:219`
  exports `query()` and delegates to `queryLoop`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/query.ts:241`
  defines `queryLoop` with mutable cross-iteration state and an infinite loop.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/query.ts:659`
  calls the model with messages, system prompt, tools, active agents, MCP tools,
  and runtime options.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/query.ts:1380`
  executes tool-use blocks through a streaming executor or `runTools`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/query.ts:1714`
  builds the next loop state by appending assistant messages and normalized tool
  results to the message history.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/TodoWriteTool/TodoWriteTool.ts:31`
  defines `TodoWriteTool` as a strict tool that updates session todos.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/TodoWriteTool/TodoWriteTool.ts:65`
  stores todos in app state keyed by agent id or session id.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/TodoWriteTool/prompt.ts:3`
  describes the todo list as structured task management for the current coding
  session.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/TodoWriteTool/prompt.ts:144`
  defines todo states and completion rules.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts:36`
  defines the tool for entering read-only planning mode.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts:77`
  changes the permission mode to `plan`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts:92`
  exposes SDK-facing plan content and plan file path after normalization.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts:147`
  defines exit-plan-mode approval before coding.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/AgentTool.tsx:90`
  defines the Agent tool schema, including named agents, team mode, isolation,
  cwd, and model override.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/AgentTool.tsx:239`
  spawns subagents or teammates from model tool calls.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/built-in/planAgent.ts:21`
  defines a read-only planning specialist.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/built-in/verificationAgent.ts:10`
  defines a verification specialist that actively tries to break the
  implementation.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/FileEditTool/FileEditTool.ts:86`
  defines the file edit tool with strict schema and write permission checks.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/FileWriteTool/FileWriteTool.ts:94`
  defines the file write tool with strict schema and write permission checks.

Related prior memory/context research:

- `/Users/linguanguo/dev/llm-memory-research/codex-context.research.md`
- `/Users/linguanguo/dev/llm-memory-research/claude-code-context.research.md`
- `/Users/linguanguo/dev/llm-memory-research/claude-code-sourcemap.research.md`
- `/Users/linguanguo/dev/llm-memory-research/claude-code-swarm.research.md`

## Short Position

Claude Code and Codex-style agents are best understood as:

> Model-driven coding loops with structured progress artifacts, guarded action
> tools, optional subagents, and external evaluators such as tests, builds,
> diffs, user review, and verifier agents.

They are not plan-free. They simply do not treat the plan as the main executable
runtime contract.

This distinction matters for Dayfold Agent:

| System Type | Plan Role | Runtime Role |
|---|---|---|
| Coding agent | Attention, progress, user collaboration, plan approval. | Run a model-tool-observation loop under sandbox and permission boundaries. |
| Dayfold Agent | Typed executable contract generated by the model. | Validate, schedule, bind, execute, and recover from a structured plan. |

## Shared Control Model

Both systems converge on the same high-level loop:

```text
user request / pending input
  -> model sampling
  -> assistant message or tool call
  -> guarded tool execution
  -> tool result / observation
  -> append to history
  -> model sampling again if follow-up is needed
```

The next step is usually chosen by the model at each turn from:

- current conversation history
- filesystem observations
- tool results
- todo or plan state
- system prompt rules
- user interruptions or pending input
- subagent or verifier feedback

The runtime does not normally read a todo item, pick the corresponding operation,
and execute it as a deterministic scheduler. The model sees the todo/plan and
continues deciding what to do next.

## Codex Source Notes

Codex's main loop is explicit in `run_turn`.

The runtime constructs prompt input from session history, streams a model
response, records assistant messages, dispatches tool calls, and repeats when a
follow-up is required. Tool calls are detected from completed output items,
turned into runtime calls by the tool router, executed, and fed back as tool
outputs.

The important control-flow property is:

> `needs_follow_up` is driven by tool calls, pending input, and turn state, not
> by walking through an executable plan graph.

`update_plan` is intentionally lightweight:

- The schema is a list of plan items with `step` and `status`.
- The handler parses arguments and emits a `PlanUpdate` event.
- The output back to the model is only a success marker.
- It is rejected in Codex Plan mode, which confirms that Codex separates
  collaboration/planning mode from the checklist-progress tool.

Codex also has subagent machinery. The collaboration handlers can spawn,
message, wait on, resume, and close subagents. Subagents inherit important
runtime state such as provider, approval policy, sandbox, and cwd. This gives
Codex a multi-agent collaboration surface, but the parent loop still treats
subagent interaction as tool-mediated conversation and event handling rather
than a typed workflow scheduler.

## Claude Code Source Notes

Claude Code's local sourcemap shows the same broad shape.

`queryLoop` keeps cross-iteration state, calls the model with tools and active
agent definitions, executes tool-use blocks, normalizes tool results into user
messages, appends assistant messages and tool results, and continues into the
next turn.

`TodoWriteTool` stores a structured list in app state. It is strict, visible to
the runtime, and session- or agent-scoped. The prompt gives strong operational
rules:

- Use todos for complex multi-step work.
- Keep exactly one task in progress.
- Mark tasks complete immediately.
- Do not mark tasks complete while tests fail, implementation is partial, errors
  remain unresolved, or necessary files/dependencies are missing.

This is stronger than a free-text plan because completion status is represented
as structured state. But it is still mostly model-declared state. The runtime
stores and renders the todo list; it does not execute todo items as typed tasks.

Claude Code's Plan mode is a separate boundary. Entering Plan mode changes the
permission mode and instructs the model to explore and design without editing
files. Exiting Plan mode presents the plan for approval before coding. This is
best understood as a read-only design and human approval boundary, not an
executable plan runtime.

Claude Code also has stronger subagent primitives:

- A general Agent tool can spawn specialized agents or teammates.
- Plan agents are read-only planning specialists.
- Verification agents independently test and attempt to break the
  implementation.
- Agent work can be isolated with worktrees in supported paths.

These are evaluator and delegation primitives. They improve reliability, but
they still report results back into the model-driven parent loop.

## Plan / Todo Semantics

The key distinction:

| Plan Shape | Claude Code / Codex Meaning | Dayfold Meaning |
|---|---|---|
| Free-text plan | Human-readable design or explanation. | Not enough for runtime execution. |
| Todo/checklist | Structured attention and progress state. | Useful but not the core plan IR. |
| Plan mode document | Read-only exploration output for approval. | Could inform planning, but still not executable. |
| Subagent task prompt | Delegation instruction to another model loop. | Could be a capability, but not typed dataflow by itself. |
| Typed executable plan | Mostly absent from coding-agent main loop. | Central runtime contract. |

For coding agents, "done" is usually declared by the model and reinforced by:

- tool observations
- file diffs
- tests and builds
- linters and type checks
- verifier agents
- user review
- permission and sandbox failures

For Dayfold, "done" should be represented in runtime state:

- step status
- typed outputs
- dependency readiness
- validation result
- recoverable error state
- final aggregation state

## Tool / Action Boundary

Coding agents make the action boundary explicit through tools:

- shell execution goes through sandbox, approval, timeout, cwd, and environment
  handling
- file edits and writes go through strict schemas and write permission checks
- plan mode can prohibit writes
- subagents can be read-only, isolated, or verifier-oriented
- approvals and hooks can block or redirect continuation

This is a strong safety architecture. But it is an action boundary, not a
business capability contract.

Dayfold's equivalent boundary is different:

```text
capability signature
  -> typed input schema
  -> typed output schema
  -> dependency binding
  -> runtime validation
  -> execution result
```

The coding-agent boundary answers:

> May this model perform this filesystem or shell action now?

The Dayfold boundary answers:

> May this business capability run with these typed inputs after these upstream
> dependencies have produced validated outputs?

## Evaluation And Recovery

Coding agents have a natural external evaluator: the software system itself.

They can run:

- tests
- builds
- linters
- type checks
- local servers
- browser checks
- command-line probes
- diff review

This is one reason a flexible ReAct-style loop works well for coding. The model
can explore, edit, observe failures, and try again. The filesystem and test
suite form a shared world model.

Claude Code's verification agent makes this even more explicit: it is a
separate evaluator role that tries to break the implementation and requires
command-backed evidence. That is a strong evaluator primitive, but it remains a
separate model loop, not a deterministic validator for every runtime step.

Dayfold needs a different evaluation strategy. Business task success may not
have an equivalent of `npm test`. Structural validation and type validation are
necessary but insufficient. Semantic validation has to be designed as part of
the product runtime or surrounding evaluator harness.

## Why Soft Plans Fit Coding Agents

Soft plans make sense for coding agents because:

- coding tasks are open-ended and often underspecified
- the relevant state is distributed across files, commands, tests, and runtime
  behavior
- the model often discovers the real plan while reading code
- recovery requires exploration, not just retrying a known step
- users often interrupt or redirect mid-task
- tests and diffs provide post-hoc evaluation
- the cost of extra model turns is often acceptable compared with developer time

This also explains why open-source coding agents tend to expose flexible loops
and todos rather than typed executable plans. A strict plan IR would have to
model too much of software engineering: file edits, dependency changes,
debugging, test selection, refactors, build failures, and user preference.

## Relevance To Dayfold Agent

The comparison supports the Dayfold design rather than invalidating it.

The lesson is not "todos are bad" or "ReAct is bad." The lesson is that plan
shape should match task shape.

| Dimension | Coding Agents | Dayfold Agent |
|---|---|---|
| Task space | Open-ended software work. | Product-specific business tasks. |
| Primary world model | Filesystem, terminal, tests, diffs. | Typed workflow state and business capability outputs. |
| Next-step decision | Model decides after each observation. | Runtime schedules ready typed steps after plan validation. |
| Plan artifact | Todo/checklist or approval plan. | Typed executable plan. |
| Completion | Model-declared, test/tool/user reinforced. | Runtime state plus validators and final synthesis. |
| Cost profile | More turns are acceptable for flexible coding. | Lower token cost and one-shot stability matter more. |
| Main risk | Missing edge cases or unsafe edits. | Wrong plan, wrong binding, semantic mismatch, failed business action. |

Dayfold's typed programmatic plan is reasonable when:

- capability surfaces are known
- step contracts are stable enough to type
- the business process has repeatable structure
- runtime auditability matters
- repeated manager reasoning would be expensive
- one-shot success is more important than open-ended exploration

## Takeaways

1. Claude Code and Codex use plans and todos, but those artifacts mainly support
   attention, collaboration, progress tracking, and approval.
2. Their main execution loop is model-driven: observe, choose a tool, execute,
   observe again, and continue until the model/runtime decides the turn is done.
3. Tests, builds, linters, file diffs, and verifier agents act as evaluators.
   This makes flexible loops practical for coding.
4. Subagents are delegation and evaluation primitives, not automatically typed
   workflow primitives.
5. Dayfold Agent should be positioned as a business-specific response to a
   different set of constraints: low token cost, one-shot stability, typed
   handoff, auditability, and deterministic scheduling.

## Open Questions

- Should the internal sharing document compare Codex `update_plan` and Claude
  Code `TodoWriteTool` as one shared category, or separate them because Claude
  Code's todo schema carries `activeForm` and stronger prompt-side completion
  rules?
- How much should the document mention Claude Code Plan mode? It is highly
  relevant to plan semantics, but it may distract from the main todo-mediated
  control loop.
- Should verifier agents be treated as an evaluator primitive in section 2, or
  only as a coding-agent-specific reliability pattern?
- What Dayfold metrics can play the role that tests/builds play in coding
  agents: one-shot success rate, replan rate, step failure rate, semantic error
  rate, human-intervention rate, or business acceptance rate?
