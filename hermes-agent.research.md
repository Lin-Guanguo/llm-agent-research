# Hermes Agent Architecture Research

Last Updated: 2026-05-28

## Sources

**Core Agent Loop**

- `/Users/linguanguo/dev/hermes-agent/run_agent.py:327` defines `AIAgent`, the central agent object that owns conversation flow, tool execution, and response handling.
- `/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:775` enters the bounded model/tool loop.
- `/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:3512` branches on assistant tool calls, validates tool names, and lets the model self-correct hallucinated tool calls.
- `/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:4278` computes turn completion, persists the session, logs turn-exit diagnostics, and builds the final result object.
- `/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:4486` syncs external memory and starts best-effort background memory/skill review after the response is delivered.

**Tools, Toolsets, And Execution**

- `/Users/linguanguo/dev/hermes-agent/tools/registry.py:151` defines the singleton `ToolRegistry`.
- `/Users/linguanguo/dev/hermes-agent/tools/registry.py:234` registers tools with schemas, handlers, check functions, async flags, and dynamic schema overrides.
- `/Users/linguanguo/dev/hermes-agent/tools/registry.py:337` returns OpenAI-format tool schemas after availability checks.
- `/Users/linguanguo/dev/hermes-agent/tools/registry.py:390` dispatches tool handlers and converts exceptions into JSON errors.
- `/Users/linguanguo/dev/hermes-agent/model_tools.py:264` resolves model-visible tool definitions through enabled and disabled toolsets.
- `/Users/linguanguo/dev/hermes-agent/toolsets.py:31` lists the shared Hermes core tools.
- `/Users/linguanguo/dev/hermes-agent/toolsets.py:606` recursively resolves composable toolsets.
- `/Users/linguanguo/dev/hermes-agent/agent/tool_dispatch_helpers.py:103` decides whether a batch of tool calls is safe to parallelize.
- `/Users/linguanguo/dev/hermes-agent/agent/tool_executor.py:65` executes parallel tool batches while preserving original tool-result order.
- `/Users/linguanguo/dev/hermes-agent/agent/tool_executor.py:469` executes sequential tool calls for single or interactive paths.

**Prompt, Memory, Skills, Plugins**

- `/Users/linguanguo/dev/hermes-agent/agent/system_prompt.py:1` documents the three-tier system prompt: stable, context, and volatile.
- `/Users/linguanguo/dev/hermes-agent/agent/system_prompt.py:60` assembles prompt parts and caches the system prompt for prompt-cache stability.
- `/Users/linguanguo/dev/hermes-agent/agent/system_prompt.py:274` adds memory, user profile, external memory provider blocks, and timestamp/model/provider metadata as volatile prompt content.
- `/Users/linguanguo/dev/hermes-agent/tools/memory_tool.py:1` describes the built-in file-backed memory model.
- `/Users/linguanguo/dev/hermes-agent/tools/memory_tool.py:653` defines the model-visible `memory` tool schema.
- `/Users/linguanguo/dev/hermes-agent/agent/memory_manager.py:258` registers memory providers and enforces a single external provider.
- `/Users/linguanguo/dev/hermes-agent/agent/memory_manager.py:318` builds memory system-prompt blocks.
- `/Users/linguanguo/dev/hermes-agent/agent/memory_manager.py:339` prefetches recall context from providers.
- `/Users/linguanguo/dev/hermes-agent/agent/memory_manager.py:371` syncs completed turns to providers.
- `/Users/linguanguo/dev/hermes-agent/agent/prompt_builder.py:983` builds a cached skill index for the system prompt.
- `/Users/linguanguo/dev/hermes-agent/agent/skill_commands.py:263` scans skills into slash commands.
- `/Users/linguanguo/dev/hermes-agent/agent/skill_commands.py:428` turns a skill invocation into a loaded instruction message.
- `/Users/linguanguo/dev/hermes-agent/tools/skills_tool.py:1491` exposes `skills_list` and `skill_view`.
- `/Users/linguanguo/dev/hermes-agent/tools/skill_manager_tool.py:1` describes agent-managed skill creation and editing.
- `/Users/linguanguo/dev/hermes-agent/hermes_cli/plugins.py:1` describes plugin discovery sources and plugin tool registration.
- `/Users/linguanguo/dev/hermes-agent/hermes_cli/plugins.py:128` lists lifecycle hook names.
- `/Users/linguanguo/dev/hermes-agent/hermes_cli/plugins.py:317` lets plugins register tools, slash commands, context engines, and other extension points.
- `/Users/linguanguo/dev/hermes-agent/hermes_cli/plugins.py:1535` invokes plugin hooks defensively.

**Context Compression And Session Store**

- `/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:454` defines `ContextCompressor` as the default lossy context engine.
- `/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:614` triggers compression based on token thresholds with anti-thrashing.
- `/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:640` prunes and summarizes old tool outputs before LLM summarization.
- `/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:914` generates a structured context checkpoint summary.
- `/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:1239` sanitizes tool-call/result pairs after compression.
- `/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:1495` is the main compression entry point.
- `/Users/linguanguo/dev/hermes-agent/hermes_state.py:220` defines SQLite session and message tables.
- `/Users/linguanguo/dev/hermes-agent/hermes_state.py:290` adds FTS5 message search.
- `/Users/linguanguo/dev/hermes-agent/hermes_state.py:1483` appends messages and updates message/tool counters.
- `/Users/linguanguo/dev/hermes-agent/hermes_state.py:2154` searches messages with FTS5, CJK trigram, and fallback paths.

**Delegation, Cron, Gateway, TUI, ACP**

- `/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:1` describes isolated child `AIAgent` instances.
- `/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:44` blocks dangerous or recursive child tools.
- `/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:569` builds the child system prompt.
- `/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:878` constructs child agents with inherited or overridden provider credentials and restricted toolsets.
- `/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:1918` implements `delegate_task` single and batch modes.
- `/Users/linguanguo/dev/hermes-agent/cron/jobs.py:531` creates scheduled jobs.
- `/Users/linguanguo/dev/hermes-agent/cron/scheduler.py:1204` executes cron jobs and supports no-agent script jobs.
- `/Users/linguanguo/dev/hermes-agent/cron/scheduler.py:1625` constructs an `AIAgent` for LLM-backed cron jobs.
- `/Users/linguanguo/dev/hermes-agent/gateway/run.py:1656` defines `GatewayRunner`.
- `/Users/linguanguo/dev/hermes-agent/gateway/run.py:16741` reuses cached `AIAgent` instances per session.
- `/Users/linguanguo/dev/hermes-agent/gateway/run.py:17204` calls `agent.run_conversation` from gateway message handling.
- `/Users/linguanguo/dev/hermes-agent/gateway/platforms/base.py:3278` queues or bypasses messages while a session is active.
- `/Users/linguanguo/dev/hermes-agent/tui_gateway/server.py:2004` creates TUI `AIAgent` instances.
- `/Users/linguanguo/dev/hermes-agent/tui_gateway/server.py:3339` handles TUI prompt submission.
- `/Users/linguanguo/dev/hermes-agent/acp_adapter/server.py:445` wraps Hermes as an ACP agent.
- `/Users/linguanguo/dev/hermes-agent/acp_adapter/server.py:1243` runs prompts through Hermes for editor clients.
- `/Users/linguanguo/dev/hermes-agent/acp_adapter/session.py:186` manages ACP sessions backed by `AIAgent` instances and `SessionDB`.

**Guardrails And Safety**

- `/Users/linguanguo/dev/hermes-agent/agent/tool_guardrails.py:63` defines per-turn repeated-tool-call guardrail configuration.
- `/Users/linguanguo/dev/hermes-agent/agent/tool_guardrails.py:224` tracks repeated failures and no-progress loops.
- `/Users/linguanguo/dev/hermes-agent/tools/checkpoint_manager.py:575` manages automatic filesystem checkpoints.
- `/Users/linguanguo/dev/hermes-agent/agent/file_safety.py:28` defines write-denied sensitive paths.
- `/Users/linguanguo/dev/hermes-agent/agent/file_safety.py:165` defines read-blocking rules for Hermes secrets and project `.env` files.
- `/Users/linguanguo/dev/hermes-agent/tools/terminal_tool.py:1583` executes terminal commands through environment, timeout, background, approval, and workdir checks.

**OpenClaw Comparison Baseline**

- `/Users/linguanguo/dev/llm-agent-research/openclaw-agent-design.case.md:113` summarizes OpenClaw as a personal agent shell with soft, model-mediated planning.
- `/Users/linguanguo/dev/llm-agent-research/openclaw-agent-design.case.md:137` lists OpenClaw's source-level layers: agent loop, tool boundary, runtime identity, context state, delegation, long-running entry points, and action gates.
- `/Users/linguanguo/dev/llm-agent-research/openclaw-agent-design.case.md:191` describes OpenClaw's control model as a long-running personal assistant rather than typed workflow runtime.
- `/Users/linguanguo/dev/llm-agent-research/openclaw-agent-design.case.md:317` describes OpenClaw's gateway-mediated subagent design.
- `/Users/linguanguo/dev/llm-agent-research/openclaw-agent-design.case.md:367` states that OpenClaw and coding-agent loops are not radically different at the inner-loop level.
- `/Users/linguanguo/dev/llm-agent-research/openclaw-agent-design.case.md:401` gives the OpenClaw takeaways around explicit primitives and attention-oriented planning.

## Overview

Hermes is best understood as a **personal-agent runtime shell around a standard LLM-driven tool loop**.

The inner loop is conventional: `AIAgent` sends messages and tool schemas to a model, the model either answers or emits tool calls, the runtime executes those calls, appends tool results, and repeats until a final response, interruption, failure, or iteration budget ends the turn. The source signal is direct: `AIAgent` is the core object in `/Users/linguanguo/dev/hermes-agent/run_agent.py:327`, the bounded loop starts in `/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:775`, tool-call processing begins at `/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:3512`, and final result construction happens at `/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:4278`.

The runtime around that loop is much larger than the loop itself. Hermes implements prompt caching, toolset resolution, plugin and MCP-style extensibility, memory providers, session search, context compression, subagents, cron jobs, gateway adapters, TUI and ACP frontends, approval bridges, checkpoints, file safety, and repeated-tool guardrails. Those mechanisms make the system feel more like an agent product than a thin ReAct demo, but they do not change the central control contract into a typed executable plan.

## 1. Product Center: Runtime Shell, Not Plan Runtime

Hermes' architecture center is the `AIAgent` plus a family of runtime surfaces:

| Layer | Hermes Owns | Source Signal |
|---|---|---|
| Agent turn | Bounded model/tool loop, interrupt handling, final result object. | `/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:775`, `/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:4278` |
| Tool boundary | Registry, toolsets, check functions, dynamic schemas, sequential/concurrent dispatch. | `/Users/linguanguo/dev/hermes-agent/tools/registry.py:151`, `/Users/linguanguo/dev/hermes-agent/model_tools.py:264`, `/Users/linguanguo/dev/hermes-agent/agent/tool_executor.py:65` |
| Persistent state | SQLite sessions/messages, FTS search, memory files, external memory providers. | `/Users/linguanguo/dev/hermes-agent/hermes_state.py:220`, `/Users/linguanguo/dev/hermes-agent/hermes_state.py:290`, `/Users/linguanguo/dev/hermes-agent/tools/memory_tool.py:1` |
| Context management | Cached system prompt and lossy compression of old turns/tool results. | `/Users/linguanguo/dev/hermes-agent/agent/system_prompt.py:1`, `/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:454` |
| Delegation | Child `AIAgent` construction, restricted toolsets, batch execution, depth controls. | `/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:1`, `/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:878`, `/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:1918` |
| Long-running entry points | Cron jobs, no-agent watchdog scripts, gateway cron ticker. | `/Users/linguanguo/dev/hermes-agent/cron/jobs.py:531`, `/Users/linguanguo/dev/hermes-agent/cron/scheduler.py:1204`, `/Users/linguanguo/dev/hermes-agent/gateway/run.py:18196` |
| Interaction surfaces | Gateway, TUI, ACP, approvals, queueing, streaming, session reload. | `/Users/linguanguo/dev/hermes-agent/gateway/run.py:1656`, `/Users/linguanguo/dev/hermes-agent/tui_gateway/server.py:3339`, `/Users/linguanguo/dev/hermes-agent/acp_adapter/server.py:1243` |
| Safety and repair | Tool guardrails, checkpoints, file safety, terminal approval and background rules. | `/Users/linguanguo/dev/hermes-agent/agent/tool_guardrails.py:224`, `/Users/linguanguo/dev/hermes-agent/tools/checkpoint_manager.py:575`, `/Users/linguanguo/dev/hermes-agent/tools/terminal_tool.py:1583` |

The important distinction is that these are runtime controls around a model-mediated loop. I did not find a central typed `ExecutionPlan`, DAG scheduler, typed dataflow binding layer, or step-state machine that acts as the primary task contract. Hermes can ask the model to use `todo`, `delegate_task`, `cronjob`, `skill_manage`, or other tools, but the model remains the local decision-maker for what to do next inside each turn.

## 2. Core Model/Tool Loop

`run_conversation` prepares memory prefetch, then enters a bounded loop that runs while API-call and iteration budgets remain. The loop checks user interrupt state before each model call and exits with `interrupted_by_user` when needed (`/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:775`).

When the model emits tool calls, Hermes validates and repairs tool names before execution. Unknown tool names are reported back as tool errors for model self-correction, and repeated invalid tool calls can stop the turn as partial (`/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:3512`). This is a hardened ReAct loop: it anticipates model errors and repairs or reflects them back, but the next action still comes from the model.

The final phase saves trajectories, cleans resources, persists session state, logs turn-exit diagnostics, optionally appends a file-mutation failure footer, runs plugin output transforms, and returns a structured result with tokens, cost, model/provider, interruption state, and guardrail metadata (`/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:4278`). After that, external memory sync and background memory/skill review run outside the user-visible critical path (`/Users/linguanguo/dev/hermes-agent/agent/conversation_loop.py:4486`).

## 3. Tool System: Registry, Toolsets, Concurrency, Guardrails

Hermes' tool layer is explicit and modular. The singleton `ToolRegistry` stores schemas, handlers, availability checks, async flags, descriptions, emoji, result-size limits, and dynamic schema overrides (`/Users/linguanguo/dev/hermes-agent/tools/registry.py:151`, `/Users/linguanguo/dev/hermes-agent/tools/registry.py:234`). `get_definitions` filters tools through availability checks and returns model-visible OpenAI-format schemas (`/Users/linguanguo/dev/hermes-agent/tools/registry.py:337`). Dispatch catches exceptions and returns JSON errors instead of letting tool exceptions break the model loop (`/Users/linguanguo/dev/hermes-agent/tools/registry.py:390`).

Tool exposure is controlled through composable toolsets. The shared Hermes core tool list includes web, terminal/process, file, vision/image, skills, browser, TTS, todo/memory, session search, clarify, code execution, delegation, cron, messaging, Home Assistant, kanban, and computer-use tools (`/Users/linguanguo/dev/hermes-agent/toolsets.py:31`). `model_tools.get_tool_definitions` resolves enabled/disabled toolsets and asks the registry for final schemas (`/Users/linguanguo/dev/hermes-agent/model_tools.py:264`). `toolsets.resolve_toolset` recursively expands includes and supports the special all-tools aliases (`/Users/linguanguo/dev/hermes-agent/toolsets.py:606`).

Execution is also hardened. Hermes can run safe tool batches concurrently while preserving the tool-result order expected by the API (`/Users/linguanguo/dev/hermes-agent/agent/tool_executor.py:65`). Parallelization is gated by a helper that rejects unsafe batches, unknown argument shapes, overlapping path-scoped tools, and non-parallel-safe tools (`/Users/linguanguo/dev/hermes-agent/agent/tool_dispatch_helpers.py:103`). Sequential execution remains the path for single or interactive tools and includes plugin pre-call blocks, guardrail decisions, checkpointing, memory-provider routing, context-engine tool routing, and normal registry dispatch (`/Users/linguanguo/dev/hermes-agent/agent/tool_executor.py:469`).

The guardrail layer catches repetitive failure modes. The controller tracks exact repeated failures, same-tool failures, and idempotent calls that return the same result, with warnings by default and optional hard stops (`/Users/linguanguo/dev/hermes-agent/agent/tool_guardrails.py:63`, `/Users/linguanguo/dev/hermes-agent/agent/tool_guardrails.py:224`). File-mutating and destructive terminal operations can trigger automatic checkpoints before execution (`/Users/linguanguo/dev/hermes-agent/agent/tool_executor.py:103`, `/Users/linguanguo/dev/hermes-agent/tools/checkpoint_manager.py:575`).

## 4. Prompt, Memory, Skills, Plugins

Hermes is especially deliberate about prompt-cache stability. `agent/system_prompt.py` states that the prompt is assembled once per session and reused across turns, with three tiers: stable identity/tool guidance/skills, context files, and volatile memory/user/timestamp data (`/Users/linguanguo/dev/hermes-agent/agent/system_prompt.py:1`). The assembly function returns those tiers and caches the joined prompt on the agent (`/Users/linguanguo/dev/hermes-agent/agent/system_prompt.py:60`). Volatile content includes built-in memory, user profile, external memory provider blocks, and a date/session/model/provider line (`/Users/linguanguo/dev/hermes-agent/agent/system_prompt.py:274`).

The built-in memory tool is file-backed and intentionally frozen into the system prompt at session start. The memory module describes `MEMORY.md` and `USER.md`, durable mid-session writes, and the frozen snapshot pattern (`/Users/linguanguo/dev/hermes-agent/tools/memory_tool.py:1`). The model-visible tool schema asks the model to save durable facts and avoid temporary task progress (`/Users/linguanguo/dev/hermes-agent/tools/memory_tool.py:653`). External memory providers are plugged through `MemoryManager`, which registers one external provider, builds provider prompt blocks, prefetches recall context, syncs completed turns, and routes provider-specific tools (`/Users/linguanguo/dev/hermes-agent/agent/memory_manager.py:258`, `/Users/linguanguo/dev/hermes-agent/agent/memory_manager.py:318`, `/Users/linguanguo/dev/hermes-agent/agent/memory_manager.py:339`, `/Users/linguanguo/dev/hermes-agent/agent/memory_manager.py:371`).

Skills are both prompt-indexed and tool-addressable. Hermes builds a cached skill index for the system prompt, with local and external skill directories plus snapshot caching (`/Users/linguanguo/dev/hermes-agent/agent/prompt_builder.py:983`). It scans skills into slash commands (`/Users/linguanguo/dev/hermes-agent/agent/skill_commands.py:263`), converts invoked skills into loaded instruction messages (`/Users/linguanguo/dev/hermes-agent/agent/skill_commands.py:428`), exposes `skills_list` and `skill_view` tools (`/Users/linguanguo/dev/hermes-agent/tools/skills_tool.py:1491`), and lets the model create or edit user skills through `skill_manage` (`/Users/linguanguo/dev/hermes-agent/tools/skill_manager_tool.py:1`, `/Users/linguanguo/dev/hermes-agent/tools/skill_manager_tool.py:1018`).

Plugins extend Hermes at multiple points. The plugin manager discovers bundled, user, opt-in project, and pip entry-point plugins (`/Users/linguanguo/dev/hermes-agent/hermes_cli/plugins.py:1`). Valid hooks include pre/post tool, terminal/tool result transforms, LLM call hooks, session hooks, subagent stop, gateway dispatch, and approval lifecycle hooks (`/Users/linguanguo/dev/hermes-agent/hermes_cli/plugins.py:128`). Plugin contexts can register tools, slash commands, context engines, and other capabilities (`/Users/linguanguo/dev/hermes-agent/hermes_cli/plugins.py:317`). Hook invocation is defensive: each callback is isolated so plugin errors do not break the core agent loop (`/Users/linguanguo/dev/hermes-agent/hermes_cli/plugins.py:1535`).

## 5. Session Store And Context Compression

Hermes stores session metadata and messages in SQLite. The schema tracks source, user, model, prompt, parent session, token/cost counters, titles, and handoff state (`/Users/linguanguo/dev/hermes-agent/hermes_state.py:220`). Messages are indexed into FTS5 tables for search, with a trigram table for CJK substring matching (`/Users/linguanguo/dev/hermes-agent/hermes_state.py:290`). Message append increments per-session message and tool-call counts (`/Users/linguanguo/dev/hermes-agent/hermes_state.py:1483`). Search supports FTS5 ranking, time sorting, CJK trigram, and LIKE fallback paths (`/Users/linguanguo/dev/hermes-agent/hermes_state.py:2154`).

The default context engine is a lossy `ContextCompressor` rather than a typed state reducer. Its documented algorithm prunes old tool results, protects head and tail messages, summarizes middle turns, and iteratively updates prior summaries (`/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:454`). It decides whether to compress based on token thresholds and backs off when repeated compressions save too little (`/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:614`). It prunes duplicate and old tool results before spending an LLM call on summarization (`/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:640`), uses a structured checkpoint summary prompt (`/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:914`), and sanitizes orphaned tool-call/result pairs after compression (`/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:1239`). The main `compress` method assembles a compressed message list with a summary marker and strips historical media payloads (`/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:1495`).

This is strong context hygiene. It preserves continuity, API validity, and prompt-cache economics, but it is still context state rather than execution-plan state.

## 6. Delegation And Subagents

Hermes implements "agents as tools" via `delegate_task`. The module describes child `AIAgent` instances with fresh conversation, own task ID, restricted toolset, focused prompt, and parent-visible summary only (`/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:1`). Children are blocked from recursive delegation by default, clarify, memory writes, cross-platform messaging, and execute-code (`/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:44`).

Child prompts are focused on a delegated goal and optional context, with an orchestrator role that can spawn further workers when config and depth allow it (`/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:569`). Child agent construction intersects requested toolsets with parent capability, strips blocked tools, optionally re-adds delegation for orchestrators, supports provider/model overrides, inherits fallbacks and credential pools, skips parent context files and memory, and gives each child a fresh budget (`/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:878`).

`delegate_task` supports single and batch modes, enforces a spawn pause switch, depth limits, max concurrent children, config-owned max iterations, credential resolution, parallel batch execution, interrupt-aware polling, memory-provider delegation notifications, plugin `subagent_stop` hooks, and cost rollup (`/Users/linguanguo/dev/hermes-agent/tools/delegate_tool.py:1918`).

This is manager-worker orchestration, not a typed DAG. The parent decides to delegate through the model loop, each child runs its own model loop, and the parent receives summaries.

## 7. Cron, Gateway, TUI, ACP: Product Runtime Surfaces

Cron turns Hermes into a scheduled agent. Jobs live under `~/.hermes/cron`, support duration/interval/cron/timestamp schedules, can pin model/provider/base URL, scripts, context chaining, toolsets, workdir, profile, and no-agent mode (`/Users/linguanguo/dev/hermes-agent/cron/jobs.py:1`, `/Users/linguanguo/dev/hermes-agent/cron/jobs.py:209`, `/Users/linguanguo/dev/hermes-agent/cron/jobs.py:531`). The scheduler can run pure script watchdog jobs without creating an `AIAgent` (`/Users/linguanguo/dev/hermes-agent/cron/scheduler.py:1204`) or construct a cron-scoped `AIAgent` for LLM jobs (`/Users/linguanguo/dev/hermes-agent/cron/scheduler.py:1625`). It applies cron-specific disabled toolsets, prompt-injection scanning, inactivity timeout, at-most-once advancement, parallel execution for safe jobs, and sequential execution for jobs that mutate process-global workdir/profile state (`/Users/linguanguo/dev/hermes-agent/cron/scheduler.py:60`, `/Users/linguanguo/dev/hermes-agent/cron/scheduler.py:1658`, `/Users/linguanguo/dev/hermes-agent/cron/scheduler.py:1889`).

Gateway is the multi-platform runtime. `GatewayRunner` manages adapters, session store, delivery, running agents, pending messages, queued events, busy acknowledgements, and a per-session `AIAgent` cache (`/Users/linguanguo/dev/hermes-agent/gateway/run.py:1656`). Session keys isolate DMs, groups, channels, threads, and users according to configured rules (`/Users/linguanguo/dev/hermes-agent/gateway/session.py:600`). Incoming messages are guarded by active-session logic that can bypass critical commands, route clarify answers, queue follow-ups, debounce text, or spawn background processing (`/Users/linguanguo/dev/hermes-agent/gateway/platforms/base.py:3278`). Gateway reuses cached agents when config signatures match, preserving frozen system prompts and prompt-cache hits (`/Users/linguanguo/dev/hermes-agent/gateway/run.py:16741`). It bridges clarify prompts, execution approvals, stream callbacks, long-running notifications, interrupts, session split handling after compression, and final delivery around `agent.run_conversation` (`/Users/linguanguo/dev/hermes-agent/gateway/run.py:16874`, `/Users/linguanguo/dev/hermes-agent/gateway/run.py:17204`, `/Users/linguanguo/dev/hermes-agent/gateway/run.py:17452`).

The TUI creates the same kind of `AIAgent`, but wraps it in JSON-RPC session state, history locks, streaming render events, approval notifications, slash workers, process-completion notifications, and subagent controls (`/Users/linguanguo/dev/hermes-agent/tui_gateway/server.py:118`, `/Users/linguanguo/dev/hermes-agent/tui_gateway/server.py:2004`, `/Users/linguanguo/dev/hermes-agent/tui_gateway/server.py:3339`, `/Users/linguanguo/dev/hermes-agent/tui_gateway/server.py:3055`). It persists history returned from `run_conversation`, syncs session keys after compression, emits final message events, and can continue an active goal after each completed turn (`/Users/linguanguo/dev/hermes-agent/tui_gateway/server.py:3604`).

The ACP adapter exposes Hermes to editor clients. `HermesACPAgent` advertises commands, session modes, model state, session load/resume/fork/list, cancellation, and prompt handling (`/Users/linguanguo/dev/hermes-agent/acp_adapter/server.py:445`, `/Users/linguanguo/dev/hermes-agent/acp_adapter/server.py:1086`, `/Users/linguanguo/dev/hermes-agent/acp_adapter/server.py:1243`). ACP sessions are backed by `AIAgent` plus the shared `SessionDB`; they can be restored from DB, forked, persisted, and recreated with working directory and runtime provider metadata (`/Users/linguanguo/dev/hermes-agent/acp_adapter/session.py:186`, `/Users/linguanguo/dev/hermes-agent/acp_adapter/session.py:423`, `/Users/linguanguo/dev/hermes-agent/acp_adapter/session.py:563`).

These surfaces explain much of the likely "better experience" claim: Hermes puts significant engineering into session continuity, active-run handling, streaming, approvals, status updates, and product surfaces around the same inner loop.

## 8. OpenClaw Comparison

OpenClaw's existing research baseline says it is also a personal agent shell with durable identity, channels, session isolation, tools, memory, and long-running entry points, while its plan remains soft and model-mediated (`/Users/linguanguo/dev/llm-agent-research/openclaw-agent-design.case.md:113`). Its source-level layers match the same broad categories: agent loop, tool boundary, runtime identity, context state, delegation, long-running entry points, and action gates (`/Users/linguanguo/dev/llm-agent-research/openclaw-agent-design.case.md:137`). The OpenClaw note explicitly says the model remains the main local decision-maker and the runtime does not turn the model plan into a typed DAG or typed dataflow contract (`/Users/linguanguo/dev/llm-agent-research/openclaw-agent-design.case.md:191`).

So the clean comparison is not "Hermes has a different agent architecture." It is closer to:

| Dimension | Hermes | OpenClaw Baseline |
|---|---|---|
| Inner control loop | `AIAgent` model/tool loop with retries, tool validation, interrupts, persistence. | Personal-agent loop delegated to Pi agent sessions, with context assembly and tool wrapping. |
| Planning model | Soft planning in model attention, `todo`, subagent prompts, skills, and session history. | Soft planning in model attention, conversation, system prompt, and subagent task prompts. |
| Tool boundary | Central registry, composable toolsets, dynamic schemas, plugin tools, parallel-safe batches. | Pi tools plus OpenClaw policy, sandbox, channel, exec, and plugin wrapping. |
| State model | SQLite sessions/messages with FTS, memory files, external memory providers, cached system prompt, compression. | Persistent sessions, workspace memory, context engine, memory search/get, channel state. |
| Delegation | Child `AIAgent` instances, restricted toolsets, batch parallel execution, depth and pause controls. | Gateway-mediated child sessions, spawn limits, parent steering/list/read/terminate, completion pushback. |
| Long-running work | Cron jobs, no-agent watchdogs, gateway ticker, background process notifications. | Cron, wake, heartbeat, persistent sessions, subagents. |
| UX/runtime hardening | Agent cache, active-session queueing, clarify/approval bridges, TUI, ACP, long-running notifications, auto-title, compression split handling. | Channel routing, session isolation, subagents, owner-only tools, sandboxing, approvals, heartbeat. |
| Architectural center | Productized personal-agent runtime shell around an LLM tool loop. | Personal-agent shell around an LLM tool loop. |

OpenClaw's note also says its subagent design is manager-worker orchestration where the dependency structure stays mostly in model attention and session messages (`/Users/linguanguo/dev/llm-agent-research/openclaw-agent-design.case.md:317`). Hermes' `delegate_task` implementation matches that class of primitive: it makes child agents practical and observable, but it does not create a typed dependency graph.

The plausible reason Hermes may feel better is therefore implementation maturity and product breadth, not a fundamentally different control-flow primitive. Source-level examples include per-session agent caching for prompt-cache hits (`/Users/linguanguo/dev/hermes-agent/gateway/run.py:16741`), SQLite FTS session search (`/Users/linguanguo/dev/hermes-agent/hermes_state.py:2154`), path-safe concurrent tool execution (`/Users/linguanguo/dev/hermes-agent/agent/tool_dispatch_helpers.py:103`), structured context compression with tool-pair repair (`/Users/linguanguo/dev/hermes-agent/agent/context_compressor.py:1239`), gateway active-session queue/bypass logic (`/Users/linguanguo/dev/hermes-agent/gateway/platforms/base.py:3278`), ACP/TUI surfaces (`/Users/linguanguo/dev/hermes-agent/acp_adapter/server.py:1243`, `/Users/linguanguo/dev/hermes-agent/tui_gateway/server.py:3339`), and cron no-agent watchdog mode (`/Users/linguanguo/dev/hermes-agent/cron/scheduler.py:1204`).

## 9. Architecture Position

Hermes should be classified as a **productized personal-agent shell with a hardened ReAct/tool-calling core**.

It implements many explicit primitives:

- durable identity and session routing;
- cached prompt assembly;
- tool registry and toolset governance;
- parallel-safe and sequential tool execution paths;
- file, terminal, approval, checkpoint, and repeated-tool guardrails;
- built-in and external memory providers;
- searchable SQLite transcripts;
- lossy context compression;
- skill discovery, invocation, and model-managed skill writing;
- plugin hooks and plugin tools;
- child-agent delegation;
- cron and no-agent scheduled watchdogs;
- gateway, TUI, and ACP frontends.

But its execution state is still mostly conversation state, context state, runtime state, and tool observations. The runtime validates, repairs, gates, compresses, persists, dispatches, resumes, interrupts, and delivers. It does not make a typed executable plan the central object that the scheduler owns.

**Conclusion:** Hermes and OpenClaw appear to occupy the same architecture family: long-running personal-agent systems around model-driven tool loops. Hermes' likely advantage is not a new control-flow theory, but a more aggressively engineered runtime envelope: cache discipline, session/search infrastructure, tool-call hygiene, UI/ACP/gateway handling, cron/no-agent support, and operational guardrails.

