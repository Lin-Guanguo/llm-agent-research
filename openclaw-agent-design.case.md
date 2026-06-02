# Case: OpenClaw Personal Agent Design

Last Updated: 2026-06-02

## Purpose

This note re-reads previous OpenClaw research from the perspective of **agent
design and control primitives**, not memory or context management.

The previous detailed research lives in `llm-memory-research`. This file keeps
only the conclusions needed for the internal Agent sharing document.

## Sources

- `/Users/linguanguo/dev/llm-memory-research/openclaw.research.md:11` records
  the context-management architecture, including ContextEngine, runtime
  pipeline, system prompt construction, and subagent model.
- `/Users/linguanguo/dev/llm-memory-research/openclaw.research.md:25`
  describes the pre-LLM context assembly pipeline:
  sanitize, provider validation, truncation, ContextEngine assembly, then model
  call.
- `/Users/linguanguo/dev/llm-memory-research/openclaw.research.md:60`
  documents the ContextEngine lifecycle methods.
- `/Users/linguanguo/dev/llm-memory-research/openclaw.research.md:117`
  summarizes the large system prompt surface, including tooling, safety,
  memory recall, authorized senders, messaging, workspace, and sandbox sections.
- `/Users/linguanguo/dev/llm-memory-research/openclaw.research.md:155`
  documents gateway-mediated subagents and bidirectional parent-child control.
- `/Users/linguanguo/dev/llm-memory-research/openclaw-memory.research.md:33`
  documents file-based memory as the source of truth.
- `/Users/linguanguo/dev/llm-memory-research/openclaw-memory.research.md:70`
  documents hybrid memory retrieval.
- `/Users/linguanguo/dev/llm-memory-research/openclaw-memory.research.md:171`
  documents pre-compaction memory flush.
- `/Users/linguanguo/dev/CyberMnema/timeline/2026/01/W05/Clawdbot调研-2026-01-27.md:12`
  records the original product framing: self-hosted personal AI assistant.
- `/Users/linguanguo/dev/CyberMnema/timeline/2026/01/W05/Clawdbot调研-2026-01-27.md:16`
  records key product traits: multi-platform integration, task execution,
  CLI/Web UI, and extensibility.
- `/Users/linguanguo/dev/CyberMnema/timeline/2026/01/W05/Clawdbot调研-2026-01-27.md:81`
  records session isolation across DMs, groups, and channels.
- `/Users/linguanguo/dev/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:1487`
  assembles the OpenClaw tool surface for an embedded Pi run.
- `/Users/linguanguo/dev/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:1651`
  builds the embedded system prompt.
- `/Users/linguanguo/dev/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:1807`
  splits tools before creating the Pi agent session.
- `/Users/linguanguo/dev/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:1836`
  creates the Pi agent session with built-in tools, custom tools, session
  manager, settings manager, and resource loader.
- `/Users/linguanguo/dev/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:2075`
  sanitizes, validates, truncates, repairs, and optionally reassembles session
  context before the model call.
- `/Users/linguanguo/dev/openclaw/src/agents/pi-embedded-runner/tool-split.ts:4`
  states that OpenClaw always passes tools as custom tools so policy filtering,
  sandbox integration, and extended tools stay consistent.
- `/Users/linguanguo/dev/openclaw/src/agents/openclaw-tools.ts:139`
  lists OpenClaw-specific tools: browser, canvas, nodes, cron, messaging, TTS,
  gateway, session tools, subagent tools, search/fetch, image, and PDF tools.
- OpenClaw `0f1f1a1f`, `src/agents/tool-catalog.ts:39`
  ([GitHub](https://github.com/openclaw/openclaw/blob/0f1f1a1fd7b9087f49e7efeb899bf18f651bfebe/src/agents/tool-catalog.ts#L39))
  defines the current OpenClaw core tool sections: files, runtime, web, memory,
  sessions, UI, messaging, automation, nodes, agents, and media.
- OpenClaw `0f1f1a1f`, `src/agents/tool-catalog.ts:53`
  ([GitHub](https://github.com/openclaw/openclaw/blob/0f1f1a1fd7b9087f49e7efeb899bf18f651bfebe/src/agents/tool-catalog.ts#L53))
  lists the current core catalog entries, including file, runtime, web, memory,
  session, messaging, automation, agent-state, and media tools.
- OpenClaw `0f1f1a1f`, `src/agents/tool-catalog.ts:356`
  ([GitHub](https://github.com/openclaw/openclaw/blob/0f1f1a1fd7b9087f49e7efeb899bf18f651bfebe/src/agents/tool-catalog.ts#L356))
  defines the `minimal`, `coding`, `messaging`, and `full` tool profiles.
- OpenClaw `0f1f1a1f`, `src/agents/openclaw-tools.ts:400`
  ([GitHub](https://github.com/openclaw/openclaw/blob/0f1f1a1fd7b9087f49e7efeb899bf18f651bfebe/src/agents/openclaw-tools.ts#L400))
  assembles the current OpenClaw runtime tools for nodes, cron, message,
  heartbeat, transcripts, media, gateway, goal, session, subagent, web, image,
  PDF, and plugin surfaces.
- OpenClaw `0f1f1a1f`, `src/agents/agent-tools.ts:724`
  ([GitHub](https://github.com/openclaw/openclaw/blob/0f1f1a1fd7b9087f49e7efeb899bf18f651bfebe/src/agents/agent-tools.ts#L724))
  constructs the base coding tools and rewrites shell execution into `exec`,
  `process`, and optional `apply_patch`.
- OpenClaw `0f1f1a1f`, `src/agents/tool-search.ts:22`
  ([GitHub](https://github.com/openclaw/openclaw/blob/0f1f1a1fd7b9087f49e7efeb899bf18f651bfebe/src/agents/tool-search.ts#L22))
  defines the Tool Search control tools: `tool_search_code`, `tool_search`,
  `tool_describe`, and `tool_call`.
- OpenClaw `0f1f1a1f`, `src/agents/tool-search.ts:423`
  ([GitHub](https://github.com/openclaw/openclaw/blob/0f1f1a1fd7b9087f49e7efeb899bf18f651bfebe/src/agents/tool-search.ts#L423))
  resolves Tool Search config and falls back from code mode to tools mode when
  the isolated Node child process is unavailable.
- OpenClaw `0f1f1a1f`, `src/agents/tool-search.ts:834`
  ([GitHub](https://github.com/openclaw/openclaw/blob/0f1f1a1fd7b9087f49e7efeb899bf18f651bfebe/src/agents/tool-search.ts#L834))
  compacts eligible tools behind the visible Tool Search control surface.
- OpenClaw `0f1f1a1f`, `src/agents/tool-search.ts:985`
  ([GitHub](https://github.com/openclaw/openclaw/blob/0f1f1a1fd7b9087f49e7efeb899bf18f651bfebe/src/agents/tool-search.ts#L985))
  implements the current lexical search scoring over tool name, id, label, and
  description.
- OpenClaw `0f1f1a1f`,
  `extensions/codex/src/app-server/dynamic-tool-profile.ts:3`
  ([GitHub](https://github.com/openclaw/openclaw/blob/0f1f1a1fd7b9087f49e7efeb899bf18f651bfebe/extensions/codex/src/app-server/dynamic-tool-profile.ts#L3))
  excludes Codex app-server-owned tools from the OpenClaw dynamic tool catalog.
- OpenClaw `0f1f1a1f`,
  `extensions/codex/src/app-server/thread-lifecycle.ts:1184`
  ([GitHub](https://github.com/openclaw/openclaw/blob/0f1f1a1fd7b9087f49e7efeb899bf18f651bfebe/extensions/codex/src/app-server/thread-lifecycle.ts#L1184))
  emits the deferred searchable OpenClaw dynamic tool manifest for Codex runs.
- `/Users/linguanguo/dev/openclaw/src/agents/pi-tools.ts:270`
  resolves effective tool policies, group policies, subagent policies, sandbox
  tool policies, and execution configuration.
- `/Users/linguanguo/dev/openclaw/src/agents/pi-tools.ts:410`
  constructs the exec tool with host, security, ask, node, sandbox, background,
  timeout, and notification options.
- `/Users/linguanguo/dev/openclaw/src/agents/pi-tools.ts:570`
  applies authorization and policy pipelines before returning model-visible
  tools.
- `/Users/linguanguo/dev/openclaw/src/agents/tool-policy.ts:31`
  marks fallback owner-only tools, including cron, gateway, and nodes.
- `/Users/linguanguo/dev/openclaw/src/agents/system-prompt.ts:394`
  adds safety guidance, including no independent goals and no long-term plans
  beyond the user's request.
- `/Users/linguanguo/dev/openclaw/src/agents/system-prompt.ts:422`
  frames the runtime as a personal assistant inside OpenClaw, lists tool
  guidance, and recommends subagent spawning for complex or long tasks.
- `/Users/linguanguo/dev/openclaw/src/agents/system-prompt.ts:38`
  defines the memory recall prompt section: use `memory_search`, then
  `memory_get`, for prior work, decisions, dates, people, preferences, or
  todos.
- `/Users/linguanguo/dev/openclaw/src/context-engine/types.ts:68`
  defines the ContextEngine lifecycle contract.
- `/Users/linguanguo/dev/openclaw/src/agents/tools/memory-tool.ts:76`
  defines `memory_search` as the model-visible semantic recall tool.
- `/Users/linguanguo/dev/openclaw/src/agents/tools/memory-tool.ts:132`
  defines `memory_get` as the model-visible snippet read tool.
- `/Users/linguanguo/dev/openclaw/src/agents/tools/sessions-spawn-tool.ts:68`
  defines `sessions_spawn`, which can start subagent or ACP sessions in one-shot
  or persistent mode.
- `/Users/linguanguo/dev/openclaw/src/agents/subagent-spawn.ts:334`
  enforces subagent spawn depth, active child limits, cross-agent allowlists, and
  sandbox constraints.
- `/Users/linguanguo/dev/openclaw/src/agents/subagent-spawn.ts:558`
  builds the child task message and starts the child run through the gateway.
- `/Users/linguanguo/dev/openclaw/src/agents/subagent-spawn.ts:695`
  registers the spawned child run for later completion, control, and cleanup.
- `/Users/linguanguo/dev/openclaw/src/agents/subagent-capabilities.ts:89`
  derives subagent roles and capabilities from spawn depth.
- `/Users/linguanguo/dev/openclaw/src/agents/tools/cron-tool.ts:210`
  exposes cron jobs and wake events as an owner-only tool.
- `/Users/linguanguo/dev/openclaw/src/infra/heartbeat-runner.ts:620`
  implements scheduled heartbeat execution with active-hours, queue, isolated
  session, delivery, and transcript-pruning gates.
- `/Users/linguanguo/dev/openclaw/src/infra/exec-approvals.ts:52`
  defines command approval plans for shell execution, not for agent workflow
  planning.
- `/Users/linguanguo/dev/openclaw/src/agents/bash-tools.exec-approval-request.ts:88`
  uses two-phase registration for exec approvals before waiting for a decision.
- `/Users/linguanguo/dev/openclaw/src/routing/session-key.ts:118`
  defines session-key construction for main, peer, channel, account, group, and
  direct-message scopes.

## Short Position

OpenClaw is best understood as a **personal agent shell**:

- It gives the model durable identity, channels, session isolation, tools,
  memory, and long-running entry points.
- Its runtime has many explicit control primitives, but it does not appear to
  make plan representation the main runtime contract.
- Its plan is mostly a soft, model-mediated plan inside the conversation,
  system prompt, and tool loop.

For the internal sharing document, OpenClaw is useful because it shows a
different production pressure from Dayfold Agent:

| System | Main Pressure | Architecture Response |
|---|---|---|
| OpenClaw | Personal continuity across channels and time. | Persistent sessions, workspace memory, channel integrations, context assembly, subagents. |
| Dayfold Agent | Stable business task execution with low token cost. | Typed executable plan, deterministic scheduler, typed capabilities, validation, dataflow bindings. |

## Source-Level Reading

After reading the current OpenClaw source, the central architecture looks like
this:

| Layer | What OpenClaw Owns | Source Signal |
|---|---|---|
| Agent loop | Delegates the core model/tool loop to Pi agent sessions. | `attempt.ts` creates OpenClaw tools, builds the system prompt, then calls `createAgentSession`. |
| Tool boundary | Wraps Pi coding tools with OpenClaw policy, sandbox, channel, exec, and plugin rules. | `pi-tools.ts`, `tool-policy.ts`, `tool-split.ts`. |
| Runtime identity | Resolves agent ID, session key, workspace, sender, channel, thread, and account context. | `get-reply.ts`, `session-key.ts`. |
| Context state | Sanitizes and validates history, limits turns, repairs tool/result pairing, then allows ContextEngine assembly. | `attempt.ts`, `context-engine/types.ts`. |
| Delegation | Exposes `sessions_spawn` and `subagents` for one-shot or persistent child sessions with depth and child-count limits. | `sessions-spawn-tool.ts`, `subagent-spawn.ts`, `subagent-capabilities.ts`. |
| Long-running entry points | Supports cron jobs, wake events, heartbeats, and isolated heartbeat sessions. | `cron-tool.ts`, `heartbeat-runner.ts`. |
| Action gates | Uses owner-only tool filtering, policy pipelines, sandbox constraints, and two-phase exec approvals. | `pi-tools.ts`, `tool-policy.ts`, `exec-approvals.ts`. |

This makes OpenClaw a real agent runtime, but the center of gravity is runtime
envelope design rather than executable plan design.

## Default Tools, Memory, And Skills

OpenClaw's default tool surface is not coding-agent-centered in the same way as
Claude Code. It includes Pi coding tools, but then adds personal-agent tools:
browser, canvas, messaging, sessions, subagents, cron/wake, gateway, nodes, TTS,
web search/fetch, image, PDF, channel tools, and plugin tools.

The current OpenClaw source makes this split more explicit. Its catalog defines
eleven core sections: files, runtime, web, memory, sessions, UI, messaging,
automation, nodes, agents, and media
(OpenClaw `0f1f1a1f`, `src/agents/tool-catalog.ts:39`).
The practical tool surface can be grouped as follows:

| Group | Representative Tools | Role |
|---|---|---|
| Workspace / coding | `read`, `write`, `edit`, `apply_patch`; session-level `bash`, `grep`, `find`, `ls` | Observe and modify the workspace. The main OpenClaw coding surface uses `exec` / `process` for shell work, while session tools still define a lighter `bash`-style set. |
| Runtime execution | `exec`, `process`, `code_execution` | Run commands, manage long-running command sessions, or use provider-backed analysis. |
| Web / memory | `web_search`, `web_fetch`, `x_search`, `memory_search`, `memory_get` | Fetch outside information and retrieve long-term memory. |
| Sessions / delegation | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status` | Inspect sessions, read history, send work to another session, spawn child agents, and wait for subagent results. |
| Messaging / UI / devices | `message`, `browser`, `canvas`, `nodes`, `gateway` | Send channel messages, control UI surfaces, and interact with Gateway or paired devices. |
| Automation / lifecycle | `cron`, `heartbeat_respond` | Schedule future work, delayed follow-ups, recurring jobs, wake events, and heartbeat outcomes. |
| Agent state / planning | `agents_list`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `skill_workshop` | List available agents, maintain current goal or short plan state, and propose reusable skills. |
| Media | `image`, `pdf`, `image_generate`, `music_generate`, `video_generate`, `tts` | Understand images/PDFs, generate media, and speak text. |

OpenClaw also has a large-catalog strategy. `tools.toolSearch` can compact the
eligible OpenClaw, MCP, plugin, and client tool catalog behind a smaller visible
surface. In code mode the model sees `tool_search_code`, an isolated JavaScript
bridge with `openclaw.tools.search`, `openclaw.tools.describe`, and
`openclaw.tools.call`; in tools mode it sees `tool_search`, `tool_describe`, and
`tool_call`
(OpenClaw `0f1f1a1f`, `src/agents/tool-search.ts:22` and
`src/agents/tool-search.ts:423`).
The current search algorithm is lexical rather than semantic: it tokenizes the
query and scores matches over tool name, catalog id, label, and description
(OpenClaw `0f1f1a1f`, `src/agents/tool-search.ts:985`).

This is similar in purpose to Claude Code's deferred tool search, but the
implementation shape is different. OpenClaw first computes the effective tool
catalog after normal policy filtering, then hides most eligible tools behind a
catalog bridge (OpenClaw `0f1f1a1f`, `src/agents/tool-search.ts:834`).
In the Codex extension, OpenClaw dynamic tools default to searchable exposure,
while app-server-owned tools such as `read`, `write`, `edit`, `apply_patch`,
`exec`, `process`, `update_plan`, and the search control tools are excluded to
avoid duplicate ownership
(OpenClaw `0f1f1a1f`,
`extensions/codex/src/app-server/dynamic-tool-profile.ts:3`).

Memory is also exposed differently from Claude Code. Claude Code memory is
mostly file/context mediated: selected memories are prefetched, and the model
can inspect files with normal file tools. OpenClaw exposes memory recall as
explicit tools:

```text
memory_search(query, maxResults?, minScore?)
memory_get(path, from?, lines?)
```

The internal memory subsystem can be complex: backend selection, embedding
search, fallback handling, citation mode, result-size limits, and file/snippet
reads. But from the model's perspective, ordinary recall is a tool interface.
The call timing is mostly prompt-driven: the system prompt tells the model to
run `memory_search` and then `memory_get` for prior work, decisions, dates,
people, preferences, or todos. It is not a hard scheduler gate in the normal
answering path.

There is a separate lifecycle mechanism called pre-compaction memory flush. That
is not the same as recall. It is a runtime-triggered run that writes important
session content to memory before compaction pressure loses detail.

Skills use another pattern. OpenClaw injects an `<available_skills>` listing and
asks the model to read the selected `SKILL.md` with the read tool. Claude Code's
skill path is more tool-mediated: a listing is shown, then `SkillTool` expands
the selected skill. In short:

| Capability | Claude Code Pattern | OpenClaw Pattern |
|---|---|---|
| Default tools | Coding workspace and development loop. | Personal-agent runtime and real-world surfaces. |
| Memory recall | File/context mediated, with runtime prefetch. | Explicit `memory_search` / `memory_get` tools. |
| Skill loading | `SkillTool` expands selected skill. | Prompt listing plus read selected `SKILL.md`. |
| Long-running work | Mostly task/session/background command oriented. | Cron, wake, heartbeat, persistent sessions, subagents. |

## Control Model

OpenClaw's control loop is closer to a long-running personal assistant than a
typed workflow runtime.

```text
channel / CLI / system event
  -> session resolution
  -> context assembly
  -> model turn
  -> tool / plugin / subagent calls
  -> transcript, memory, and channel state update
```

The model remains the main local decision-maker inside each turn. The runtime
adds durable state, context discipline, provider adaptation, channel handling,
and tool/plugin surfaces, but it does not turn the model's plan into a typed DAG
or a typed dataflow contract.

One important nuance: OpenClaw does have hard control flow. Examples include
queue behavior for active runs, heartbeat skip gates, cron payload constraints,
tool-policy filtering, subagent spawn limits, sandbox checks, and exec approval
registration. The distinction is that these controls protect and route the
agent runtime; they are not a domain workflow plan IR.

## Plan Representation

OpenClaw is a strong example of **soft planning**.

The relevant plan-like artifacts are:

- The model's implicit plan in the current turn.
- Any textual todo or reasoning state the model writes into the conversation.
- Subagent task prompts sent through the gateway-mediated subagent mechanism.
- Memory files and session history that influence future planning.

These artifacts are useful for attention and continuity, but they are not the
same as an executable typed plan. Step completion is mostly model-mediated or
tool-observation-mediated, not scheduler-mediated.

I also searched the agent/runtime source for plan and todo-like surfaces. The
notable `Plan` hits are local operational plans such as model config plans,
exec approval plans, sandbox filesystem command plans, reply-reference planning,
or tests. I did not find a central application-level `ExecutionPlan`, typed DAG,
or todo middleware that the runtime schedules as the agent's main task contract.

So the more precise claim is:

> OpenClaw is not "unstructured." It has structured runtime controls. What it
> does not appear to have is a typed executable plan as the central task state.

## State Model

OpenClaw externalizes a lot of state, but most of it is **context state** rather
than **execution-plan state**.

| State Surface | Role |
|---|---|
| Session key | Separates DM, group, channel, thread, or agent-specific context. |
| Session transcript | Stores the turn history for later reconstruction and retrieval. |
| Workspace files | Provide durable, user-auditable state. |
| `MEMORY.md` / `memory/*.md` | Store long-term and daily memories. |
| ContextEngine output | Assembles the subset of state sent to the model. |
| Subagent session | Gives child agents independent context with parent steering. |
| Cron / heartbeat state | Lets the agent wake later, run isolated work, or react to system events. |
| Exec approval state | Separates side-effect proposal, approval registration, and final decision. |

This is highly relevant for personal agents because continuity and identity are
core product requirements. For Dayfold, the corresponding requirement is
different: the runtime must know which business steps are pending, ready,
completed, failed, or waiting for user input.

## Tool And Action Boundary

OpenClaw exposes a broad personal-agent action surface:

- Channel integrations such as chat platforms.
- Tooling and skill surfaces.
- Workspace and filesystem interaction.
- Subagent spawning and steering.
- Memory search and memory reads.
- Cron, wake, heartbeat, and delayed execution.
- Shell execution with sandbox, allowlist, and approval controls.

The important design lesson is that personal agents need explicit boundaries
around identity, authorized senders, workspace, sandbox, tools, and channels.
OpenClaw's large system prompt surface reflects that pressure.

This is not the same kind of boundary as Dayfold's typed capability contract.
OpenClaw's boundary is more about "who is speaking, through which channel, with
which tools and sandbox." Dayfold's boundary is more about "which business
capability may run, with which typed inputs, after which dependencies are ready."

This is a useful distinction for the sharing document:

| Boundary Type | OpenClaw Example | Dayfold Analogue |
|---|---|---|
| Identity boundary | Sender, owner, channel, account, session key. | User / tenant / business context. |
| Tool boundary | Policy-filtered tools, owner-only tools, plugin allowlists. | Typed capabilities and capability allowlists. |
| Workspace boundary | Workspace guards, sandbox filesystem bridge, sandboxed exec. | Runtime-owned data scopes and typed port values. |
| Side-effect boundary | Exec approvals, cron ownership, message target checks. | Capability validation, approval points, write gates. |
| Progress boundary | Model/tool loop, subagent completion, heartbeat/cron events. | Scheduler step state and typed dependency readiness. |

## Evaluation And Recovery

OpenClaw has strong context-level safeguards:

- Provider-aware message validation.
- Tool-result pairing cleanup.
- Turn-history truncation.
- ContextEngine assembly under budget.
- Compaction and pre-compaction memory flush.

These are not domain-task evaluators. They protect the agent's conversation and
memory continuity, but they do not prove that a multi-step business task was
semantically completed.

For the internal document, this contrast is useful:

| Concern | OpenClaw Emphasis | Dayfold Emphasis |
|---|---|---|
| Context reliability | Strong. | Important, but secondary to execution state. |
| Memory continuity | Strong. | Useful, but not the main runtime contract. |
| Plan execution | Soft/model-mediated. | Typed/runtime-mediated. |
| Completion judgment | Model/tool observations. | Runtime state plus validators, with semantic gaps still remaining. |

## Subagent / Orchestration

OpenClaw's gateway-mediated subagent design is architecturally interesting:

- Child agents run in independent sessions.
- Spawn depth, active child count, cross-agent allowlists, and sandbox rules are
  checked before child creation.
- The parent can steer, list, read history, or terminate a child.
- Child completion is pushed back to the parent.
- ContextEngine lifecycle hooks can prepare and clean up subagent context.

This is closer to **personal-agent delegation** than typed business workflow
execution. It helps when the parent wants parallel or isolated reasoning, but it
does not by itself create a typed dependency graph or typed input bindings.

This maps to a common agent primitive:

```text
manager model
  -> spawn child session with task prompt
  -> child runs with isolated context and inherited workspace
  -> completion is announced back to parent
  -> parent decides next action
```

It is "agents as tools" / manager-worker orchestration, but the dependency
structure remains mostly in model attention and session messages.

## Relevance To Dayfold Agent

OpenClaw is a good contrast case because it is production-oriented but optimizes
for a different axis.

Dayfold should not be described as "OpenClaw but for business workflows." The
architectural center is different:

| Dimension | OpenClaw | Dayfold Agent |
|---|---|---|
| Primary domain | Personal assistant. | Product-specific business automation. |
| Main state | Sessions, memory, workspace, channels. | Workflow state, step state, typed port values. |
| Plan shape | Soft plan in model attention and subagent prompts. | LLM-authored typed executable plan. |
| Runtime role | Context assembly, channel/tool surface, persistence, subagent lifecycle. | Validation, scheduling, binding, dispatch, recovery. |
| Best lesson | Long-running personal continuity requires explicit state and boundaries. | Stable business execution requires explicit plan/runtime contracts. |

## Relevance To Current Article

OpenClaw should appear in the internal sharing document as the representative
case for **personal-agent products**, not as a direct competitor to Dayfold's
plan-and-execute design.

For the current Agent architecture research, OpenClaw is not radically different
from Claude Code-style agents at the inner-loop level. Both are still
model-driven tool loops with model-visible tool observations, soft planning, and
optional subagent delegation. The main difference is outside that loop:
OpenClaw adds a longer-lived personal-agent lifecycle around the loop.

That lifecycle mainly covers:

- session and identity routing;
- transcript, workspace, memory, and context continuity;
- per-run tool construction and policy filtering;
- side-effect gates such as sandboxing, owner-only tools, and exec approvals;
- subagent spawn, tracking, completion, steering, and cleanup;
- background entry points such as cron, wake, and heartbeat;
- delivery back to the correct channel, account, thread, or session.

UI and IM surfaces should not be overstated as agent intelligence. UI is mostly
interaction and observability. IM channels matter only insofar as they become
runtime boundaries for identity, authorization, session routing, and delivery.

The useful comparison is:

- Claude Code / Codex-style agents show interactive coding-agent loops.
- OpenClaw shows a personal-agent runtime shell: identity, channels, session,
  memory, tools, background triggers, subagents, and approval boundaries.
- LangChain DeepAgents / todo-mediated agents show todo-as-attention and
  model-visible progress tracking.
- Dayfold shows programmatic typed planning for stable business execution.

This keeps the external section small while still covering the main design
families that matter for the article.

## Takeaways

1. OpenClaw supports the claim that real agents are built from explicit
   primitives: state, tools, channels, memory, context assembly, and delegation.
2. Its most relevant primitive for this research is not memory itself, but the
   way it turns a chat model into a persistent personal agent with durable
   state and broad tool boundaries.
3. Its plan remains mostly attention-oriented, even though the runtime has many
   hard gates around tools, sessions, subprocesses, cron, heartbeat, and
   approvals. That makes it a useful contrast against Dayfold's executable
   typed plan.
4. The main product lesson is that personal-agent design starts from continuity
   and access boundaries; business-agent design starts from task contracts and
   execution reliability.

## Follow-Up Questions

- How much of OpenClaw's behavior is driven by system prompt policy versus hard
  runtime gates?
- Which parts of OpenClaw's channel/session model are worth mentioning in the
  internal sharing document, and which should remain out of scope?
- Should we mention OpenClaw's cron/heartbeat subsystem as an example of
  long-running personal-agent entry points, or keep the article focused on
  interactive agent execution?
