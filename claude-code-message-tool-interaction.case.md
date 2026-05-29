# Case: Claude Code Message and Tool Interaction

Last Updated: 2026-05-21

## Purpose

This note records how Claude Code structures model-visible messages and wires
tool results, subagents, and background notifications back into the main agent
context. The focus is the interaction contract, not the product UX:

- how the parent model invokes a subagent;
- how the subagent system prompt and initial user message are constructed;
- what the experimental fork path changes;
- how the subagent result returns to the parent context;
- how ordinary tool results, parallel tool calls, and background notifications
  enter the model-visible message stream.

## Sources

Local Claude Code sourcemap:

- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/AgentTool.tsx:196`
  defines `AgentTool`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/AgentTool.tsx:239`
  accepts the model-supplied `prompt`, `subagent_type`, `description`, and
  background settings.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/AgentTool.tsx:318`
  selects the effective agent type or routes to the fork path.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/AgentTool.tsx:483`
  documents the normal-path vs fork-path split for system prompts and prompt
  messages.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/AgentTool.tsx:538`
  places the parent-supplied `prompt` into a user message for normal subagents.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/AgentTool.tsx:603`
  passes subagent parameters into `runAgent`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/runAgent.ts:368`
  constructs `initialMessages` from optional fork context plus prompt messages.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/runAgent.ts:508`
  resolves the subagent system prompt.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/runAgent.ts:748`
  runs the child agent through the same `query()` loop.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/runAgent.ts:792`
  records child messages into the sidechain transcript.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/agentToolUtils.ts:276`
  finalizes a subagent by extracting text from the last assistant message.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/AgentTool.tsx:1298`
  maps the subagent output into a parent-visible `tool_result`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tasks/LocalAgentTask/LocalAgentTask.tsx:197`
  formats asynchronous subagent completion as a `<task-notification>` user
  message.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/query.ts:1380`
  executes collected tool-use blocks and normalizes resulting messages.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/query.ts:1535`
  notes that `tool_result` messages must not be interleaved with regular user
  messages.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/query.ts:1715`
  builds the next model turn from prior messages, assistant tool-use messages,
  and tool results.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/messages.ts:1989`
  defines `normalizeMessagesForAPI()`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/messages.ts:2411`
  defines `mergeUserMessages()`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/messages.ts:2466`
  hoists `tool_result` blocks to the front of merged user turns.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/services/tools/toolOrchestration.ts:19`
  executes non-streaming tool batches and partitions concurrency-safe calls.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/services/tools/StreamingToolExecutor.ts:34`
  describes streaming-time parallel tool execution.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/BashTool/BashTool.tsx:555`
  maps Bash output into a model-facing `tool_result`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tasks/LocalShellTask/LocalShellTask.tsx:160`
  formats background shell completion as a `<task-notification>`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/constants/xml.ts:1`
  centralizes XML-like tag names used in model-visible text.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/constants/prompts.ts:131`
  tells the model that tool results and user messages may include
  `<system-reminder>` and other tags.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/Tool.ts:321`
  defines the raw tool result wrapper as `ToolResult<Output>`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/Tool.ts:557`
  requires tools to map raw output into API `tool_result` blocks.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/Tool.ts:566`
  defines the separate human-facing `renderToolResultMessage()` path.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/services/tools/toolExecution.ts:1290`
  maps tool raw output into a model-facing `ToolResultBlockParam`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/services/tools/toolExecution.ts:1403`
  adds the processed tool result block to a user-role message.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/services/tools/toolExecution.ts:1456`
  stores both the model-facing content block and the raw `toolUseResult`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/toolResultStorage.ts:189`
  formats large tool results as `<persisted-output>` text with a preview and
  file path.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/BashTool/BashTool.tsx:545`
  notes that Bash UI rendering does not show model-facing persisted-output or
  background wrappers.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/BashTool/BashTool.tsx:555`
  maps Bash raw output fields into `tool_result` content.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/components/messages/UserToolResultMessage/UserToolSuccessMessage.tsx:52`
  renders UI from `message.toolUseResult`, not from the serialized
  model-facing content string.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/TodoWriteTool/TodoWriteTool.ts:52`
  enables legacy `TodoWrite` only when Todo V2 is disabled.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/tasks.ts:133`
  enables Todo V2 by default in interactive sessions.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/TaskCreateTool/TaskCreateTool.ts:68`
  enables `TaskCreate` when Todo V2 is enabled.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/tasks.ts:221`
  stores Todo V2 task files under the Claude config task directory.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/hooks/useTasksV2.ts:20`
  implements the shared file-watched task store for UI task panels.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/attachments.ts:254`
  defines todo/task reminder thresholds as assistant-turn counts.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/attachments.ts:3319`
  counts turns since the last Todo V2 task-management tool call.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/attachments.ts:3413`
  injects a Todo V2 task reminder when the turn thresholds are met.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/messages.ts:3680`
  renders `task_reminder` attachments into model-visible system-reminders.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/services/api/claude.ts:1154`
  filters deferred tools so only non-deferred tools, `ToolSearchTool`, and
  previously discovered deferred tools are sent to the API.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/toolSearch.ts:525`
  discovers deferred tools from `tool_reference` blocks returned by
  `ToolSearchTool`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/messages.ts:3728`
  renders `skill_listing` attachments as system-reminder messages for the
  `SkillTool`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/SkillTool/SkillTool.ts:331`
  defines `SkillTool` as the explicit skill invocation mechanism.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/claudemd.ts:1`
  documents Claude Code memory file discovery and priority order.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/memdir/findRelevantMemories.ts:39`
  selects relevant memory files through a side query over memory headers.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/attachments.ts:2356`
  starts relevant-memory prefetch outside the main model's tool-call path.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/messages.ts:355`
  creates synthetic assistant messages with `model: "<synthetic>"`.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/messages.ts:435`
  creates synthetic assistant API-error messages.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/services/api/errors.ts:425`
  maps API/runtime errors into assistant API-error messages.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/screens/REPL.tsx:2121`
  preserves partially streamed text as a synthetic assistant message when the
  user interrupts generation.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/conversationRecovery.ts:226`
  inserts a synthetic assistant sentinel during conversation recovery.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/utils/messages/mappers.ts:183`
  maps local command output into an SDK-compatible assistant message.
- `/Users/linguanguo/dev/claude-code-sourcemap/restored-src/src/tools/AgentTool/UI.tsx:378`
  builds a UI-facing synthetic assistant completion message for AgentTool.

Local Codex source tree used for comparison:

- `/Users/linguanguo/dev/codex/codex-rs/core/src/session/mod.rs:2557`
  builds Codex developer-role context sections.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/context_manager/updates.rs:178`
  turns developer sections into `role="developer"` messages.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/session/mod.rs:2703`
  builds contextual user-role sections for user instructions and runtime
  environment context.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/agent/control.rs:99`
  defines what forked subagent history keeps.
- `/Users/linguanguo/dev/codex/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:227`
  defines `fork_turns` behavior for MultiAgentV2.
- `/Users/linguanguo/dev/codex/codex-rs/protocol/src/protocol.rs:813`
  converts inter-agent communication into model input.

## Core Model

Claude Code exposes a subagent as a tool call:

```json
{
  "type": "tool_use",
  "name": "Agent",
  "input": {
    "description": "...",
    "subagent_type": "Explore",
    "prompt": "...",
    "run_in_background": false
  }
}
```

The tool implementation does not execute a normal function. It creates a child
agent context, runs a child `query()` loop, records the child trace separately,
and returns only a compact result to the parent.

## Message Shape

Claude Code's model-facing messages use multi-part content blocks. The useful
mental model is:

```text
conversation = message[]
message = { role, content: block[] }
block = text | tool_use | tool_result | image | document | ...
```

This matters because "merge user messages" does not mean flattening everything
into one string. It means appending content blocks into a legal user turn while
preserving block semantics.

An assistant tool call is a structured block:

```json
{
  "role": "assistant",
  "content": [
    { "type": "tool_use", "id": "toolu_1", "name": "Read", "input": {} }
  ]
}
```

The runtime response is a user-role message containing a structured
`tool_result` block:

```json
{
  "role": "user",
  "content": [
    { "type": "tool_result", "tool_use_id": "toolu_1", "content": "..." }
  ]
}
```

The `user` role here is not necessarily a human. It is also the channel by which
the runtime returns observations from tools, tasks, hooks, and attachments.

## Codex Role Separation

Codex has a more explicit role split than Claude Code because the OpenAI
message protocol supports a `developer` role. This changes how runtime guidance
is injected.

Claude Code often puts runtime/system-ish context into user-role messages with
XML-like wrappers. This is partly a protocol consequence: Anthropic messages
mostly give the product `system` plus `user`/`assistant`, so many non-human
observations, reminders, task notifications, and tool results still enter
through user-role content blocks.

Codex separates this more finely:

- product and framework rules go into `developer` messages, including
  permissions, developer instructions, memory tool instructions, collaboration
  mode, skills, plugins, and commit-message policy;
- contextual runtime state stays in `user` messages, including user
  instructions, environment context, cwd, and open subagent references;
- inter-agent messages use a separate `InterAgentCommunication` object and are
  converted into assistant-role commentary input, rather than being returned as
  ordinary parent tool results.

So the comparison is not "Claude Code XML user messages vs Codex developer
messages" for everything. The better rule is:

```text
Claude Code:
  many runtime observations -> user role + XML-like text markers

Codex:
  policy/rules              -> developer role
  runtime context           -> user role
  agent-to-agent messages   -> assistant commentary via InterAgentCommunication
```

This is one reason Codex's MultiAgentV2 feels more first-class. Subagent state
is not just a tool call that returns one final string. The child is a live
thread with a task path, mailbox, status, and independently retained context.

## Synthetic Assistant Messages

In the normal successful query loop, assistant messages are model/API-originated
response blocks. Runtime-originated context is not usually injected through
assistant messages; it normally uses user-role tool results, user/meta
attachments, system messages, progress messages, or XML-like text wrappers.

There are a few synthetic assistant-message exceptions. `createAssistantMessage()`
constructs an assistant message with `model: "<synthetic>"`, and
`createAssistantAPIErrorMessage()` marks API/runtime failures with
`isApiErrorMessage: true`. These messages are mostly infrastructure artifacts,
not ordinary model turns.

The main cases are:

- API and runtime error presentation: timeouts, image/resize failures, rate
  limits, prompt-too-long errors, refusals, and query failures can be surfaced as
  assistant API-error messages.
- Interrupted streaming preservation: when the user presses Esc during
  generation, already streamed text is committed as a synthetic assistant
  message so the user can still read the partial output. The text itself is
  model-originated, but the committed message wrapper is runtime-created.
- Conversation recovery: if a recovered transcript would otherwise end on a user
  message, Claude Code inserts a `NO_RESPONSE_REQUESTED` assistant sentinel to
  keep the conversation structurally valid.
- SDK/UI compatibility: local command output can be converted into an assistant
  message because some downstream clients do not understand the original
  local-command system subtype.
- UI/internal placeholders: AgentTool and some tool-permission or MCP paths
  create empty or status-style assistant objects as UI/internal call context.

So the precise rule is: normal assistant messages are model/API responses; the
runtime creates a small number of synthetic assistant messages for error
handling, interruption recovery, transcript validity, SDK/UI compatibility, or
internal placeholders. These exceptions should not be treated as the general
context-injection mechanism.

## Message Normalization

Before sending messages to the API, Claude Code runs
`normalizeMessagesForAPI()`. This pass:

1. removes progress and display-only virtual messages;
2. normalizes user content from strings into content blocks;
3. merges consecutive user messages;
4. normalizes assistant tool-use inputs;
5. merges assistant streaming fragments with the same assistant message id.

For adjacent user messages, `mergeUserMessages()` keeps block order by default.
If the boundary is `text` followed by `text`, it adds a newline to the first
text block so the two pieces do not concatenate accidentally:

```text
[{ type: "text", text: "A" }]
+
[{ type: "text", text: "B" }]
=
[{ type: "text", text: "A\n" }, { type: "text", text: "B" }]
```

If merged user content includes `tool_result`, Claude Code hoists
`tool_result` blocks to the front of the user turn. The reason is protocol
safety: a `tool_result` must correspond to a preceding assistant `tool_use`, and
Claude Code avoids interleaving ordinary user messages between tool results.

There is one extra optimization around text siblings after a `tool_result`: in
some cases Claude Code "smooshes" later text or attachment blocks into the
preceding `tool_result.content` rather than keeping them as sibling blocks. The
code comment says this avoids server-rendered `Human:` boundaries after tool
results, which can create poor stop behavior.

## Ordinary Tool Result Flow

Ordinary tools follow the same model-visible protocol as the `Agent` tool:

```text
assistant: tool_use(id=...)
user:      tool_result(tool_use_id=...)
assistant: continue reasoning or answer
```

`runToolUse()` validates the tool name and input, handles permission decisions
and hooks, calls the tool, then maps the result through the tool's
`mapToolResultToToolResultBlockParam()`. The mapped `tool_result` is wrapped in
`createUserMessage()`, so the API-facing role is still `user`.

The result is not always raw tool output. Tool-specific mappers may:

- convert output into plain text;
- return image or document blocks;
- add background task ids and output-file paths;
- replace large output with a `<persisted-output>` reference;
- return structured error text such as `<tool_use_error>...</tool_use_error>`.

For Bash, stdout, stderr, interruption state, large-output persistence, and
background task metadata are combined into the final model-facing
`tool_result.content`.

## Tool Result Projections

Tool output is split into different projections. This is not an LLM formatting
task; it is programmatic runtime formatting.

At the tool interface level, a tool returns a typed runtime object:

```ts
ToolResult<Output> = {
  data: Output
  newMessages?: ...
  contextModifier?: ...
  mcpMeta?: ...
}
```

Each tool then exposes two separate projections:

```ts
mapToolResultToToolResultBlockParam(output, toolUseID): ToolResultBlockParam
renderToolResultMessage?(output, progressMessages, options): React.ReactNode
```

The first projection is model-facing. It produces an API content block such as:

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_xxx",
  "content": "...",
  "is_error": false
}
```

The second projection is human-facing UI. Claude Code renders it from
`message.toolUseResult`, not by blindly displaying the serialized
`tool_result.content`. This lets the UI show progress, concise summaries,
expanded output, hook status, approval state, and other UI-only details without
making those details part of the model context.

There is also a runtime-facing layer: full raw output, sidechain traces,
background task ids, persisted files, metrics, permission decisions, and hook
results can remain in runtime state or logs even when the model receives only a
compact projection.

So the shape is:

```text
raw tool output object
    -> model-facing JSON content block
    -> human-facing React/terminal rendering
    -> runtime persistence, logs, ids, metrics
```

The model-facing JSON block may contain plain text, an array of content blocks,
or programmatically generated XML-like text markers. For Bash:

- the runtime raw object contains fields such as `stdout`, `stderr`,
  `interrupted`, `isImage`, `backgroundTaskId`, `persistedOutputPath`, and
  `persistedOutputSize`;
- if stdout is an image data URI, the model-facing `tool_result.content` becomes
  an image content block;
- if output is large, the runtime writes the full output to disk and replaces
  the model-facing content with a `<persisted-output>` text block containing a
  preview and file path;
- if the command runs in the background, the model-facing content includes a
  task id and output path;
- the Bash UI separately renders stdout/stderr and progress from the raw output
  object, and intentionally does not show the model-facing
  `<persisted-output>` wrapper.

Thus the layers are:

```text
runtime raw result = TypeScript object
API envelope       = JSON content block
content payload    = generated text or content-block array
XML-like tags      = model-readable text conventions inside the payload
```

These XML-like tags are not produced by the model and are not the outer
protocol. They are runtime-generated text markers inserted inside the structured
tool-result content so the model can reliably distinguish tool/system metadata
from ordinary text.

## Todo And Task State

Todo/task state is another example of model-visible state and human-visible UI
being separate projections of runtime state.

Claude Code has two generations of this mechanism:

| Mechanism | Enabled when | State location | Model operation style |
| --- | --- | --- | --- |
| `TodoWrite` | Todo V2 is disabled | `AppState.todos[sessionId or agentId]` | replace the whole todo list |
| Task tools | Todo V2 is enabled | task JSON files under the Claude config task directory | create/update/list/get individual tasks |

In this source tree, Todo V2 is enabled by default for interactive sessions, and
`TodoWrite` remains available for non-interactive/SDK-style paths unless Todo
V2 is explicitly enabled there.

The old `TodoWrite` model is simple:

```text
LLM calls TodoWrite({ todos: [...] })
    -> runtime updates AppState.todos[key]
    -> UI renders the todo panel from AppState
    -> model receives only a compact success tool_result
```

The model supplies the full todo list as tool input. The runtime stores it under
the current session id or agent id. If every item is marked completed, the
runtime clears the stored list. The returned model-visible tool result is only a
short success message, plus an optional verification nudge. The full list is
not echoed back to the model in the normal tool result.

Todo V2 turns this into a small task-state system:

```text
TaskCreate -> create a task file
TaskUpdate -> update status, owner, dependencies, metadata
TaskList   -> read a compact list for the model
TaskGet    -> read one task in detail for the model
Task panel -> render task files for the human
```

Tasks are stored as files, keyed by a task-list id. The task-list id can come
from an explicit environment variable, a teammate/team context, or the session
id. The UI uses a shared `useTasksV2()` store that watches the task directory,
listens for in-process update signals, and polls as a fallback. The human task
panel is therefore not the model context; it is a UI projection of file-backed
runtime state.

The model re-enters this state through tools. `TaskCreate` and `TaskUpdate`
return compact acknowledgements such as "Task #1 created successfully" or
"Updated task #1 status". `TaskList` and `TaskGet` are the read paths that
return task summaries to the model.

Todo/task reminders are also programmatic. They are not timers and they are not
Stop hooks. Claude Code counts assistant turns:

- old `TodoWrite`: turns since the last `TodoWrite` tool use and turns since the
  last `todo_reminder`;
- Todo V2: turns since the last `TaskCreate` or `TaskUpdate` tool use and turns
  since the last `task_reminder`.

The current threshold is 10 assistant turns since the last write/update and 10
assistant turns since the last reminder. Thinking messages are skipped. When
the thresholds are met, the runtime creates a `todo_reminder` or
`task_reminder` attachment. That attachment is rendered as a model-visible
`<system-reminder>` user/meta message containing a short instruction and a
compact snapshot of the existing todos or tasks.

This reminder is an attention-refresh mechanism for externalized state. It does
not intercept completion. `StopHook` runs when the model appears ready to stop;
todo/task reminder injection happens while building context for a future model
turn.

## Parallel Tool Calls

The model can emit multiple `tool_use` blocks in one assistant turn. Claude Code
then executes those tool calls as one batch.

In the non-streaming path, `runTools()` partitions tool calls into batches:

- consecutive concurrency-safe tools can run concurrently;
- non-concurrency-safe tools run serially;
- the default maximum tool-use concurrency is 10.

In the streaming path, `StreamingToolExecutor` can begin executing a tool as
soon as its `tool_use` block streams in. Its design notes say:

- concurrency-safe tools may run in parallel with each other;
- non-concurrent tools require exclusive access;
- results are buffered and emitted in received order where needed.

So "parallel tool use" is not a different API message type. It is multiple
structured `tool_use` blocks in one assistant message, plus runtime scheduling
rules that decide which calls may execute concurrently.

## XML-Like Injection Forms

Claude Code uses two different layers of structure:

1. API-native content blocks: `tool_use`, `tool_result`, `text`, `image`,
   `document`, and similar typed blocks.
2. XML-like text markers inside `text` or `tool_result.content`.

The angle-bracket tags are mostly the second layer. They are model-visible
conventions, not separate API roles. Important examples:

| Tag | Purpose | Typical channel |
| --- | --- | --- |
| `<system-reminder>` | Runtime reminders, attachments, IDE state, memory, plan-mode state | `user` text or `tool_result.content` |
| `<tool_use_error>` | Tool-name, validation, permission, or execution errors | `tool_result.content` |
| `<task-notification>` | Background task or async worker completion | user-role meta message |
| `<persisted-output>` | Large output saved to disk with a preview and path | `tool_result.content` |
| `<command-name>`, `<command-message>`, `<command-args>` | Slash-command or skill invocation metadata | command/meta user messages |
| `<bash-stdout>`, `<bash-stderr>`, `<local-command-stdout>` | Terminal or local command output that should not be mistaken for a human prompt | user/meta messages |

The system prompt explicitly tells the model that tool results and user messages
may include these tags and that the tags contain system-provided information.

## Context Injection Surfaces

Claude Code has three related but distinct context-expansion surfaces: tools,
skills, and memory. They all solve the same pressure problem: the model needs to
know what exists without paying the token cost of seeing every full definition
or document every turn.

Tool definitions are not always fully injected. When dynamic tool loading is
enabled, Claude Code keeps deferred tools out of the API tool list until the
model discovers them through `ToolSearchTool`. `ToolSearchTool` returns
`tool_reference` blocks, and later API requests include only the discovered
tools. In this design, the tool registry is exposed through a search interface
instead of a full upfront schema dump.

Skills use a different two-step interface. Claude Code injects a lightweight
`skill_listing` attachment that tells the model which skills are available. The
full skill content is loaded only when the model calls `SkillTool`. So skills
are not just prompt text and not just tools; they are an indexed capability
surface with an explicit expansion tool.

Memory is different again. Claude Code memory is primarily file-system managed,
not exposed as a dedicated `MemoryTool`. Some memory is injected
automatically: `CLAUDE.md`, `.claude/rules/*.md`, and selected relevant memory
files can enter the context as system-reminder attachments. For relevant
memory, the runtime starts an async prefetch, scans memory headers, and uses a
side query to select up to a small number of relevant files. If the injected
memory is partial or the model needs more detail, it can use general file tools
such as `Read`, `Grep`, or `Glob` to inspect the underlying memory files.

The important distinction is who performs the expansion:

| Surface | Lightweight context | Full expansion path |
| --- | --- | --- |
| Tools | Tool schema subset plus `ToolSearchTool` | Model calls `ToolSearchTool`; runtime includes discovered tool schemas later |
| Skills | `skill_listing` system-reminder | Model calls `SkillTool`; runtime expands the skill instructions/messages |
| Memory | File-based instructions and relevant-memory attachments | Runtime prefetches a few memories; model can continue via file-system tools |

This means memory is not "through Tool" in the same sense as skills. It is
stored and deduplicated as files, opportunistically surfaced by the runtime, and
then extended through the existing file-tool surface when needed.

## Normal Subagent Context

For the normal path, `subagent_type` selects an agent definition. If no type is
provided and the fork experiment is not active, Claude Code defaults to
`general-purpose`.

The selected agent definition provides the subagent system prompt through
`getSystemPrompt()`. Built-in examples:

- `general-purpose`: general research and multi-step task execution.
- `Explore`: read-only codebase search and analysis.
- `Plan`: read-only implementation planning.

Claude Code then enhances that system prompt with environment details such as
absolute-path guidance and shell/cwd notes.

The parent-supplied `prompt` is not merged into the system prompt. It becomes
the first user message for the child:

```ts
promptMessages = [createUserMessage({ content: prompt })]
```

So the normal subagent shape is:

```text
system: selected agent prompt + environment details
user:   parent Agent-tool prompt
```

It does not receive the full parent conversation by default.

## Fork Subagent Path

`forkContextMessages` belongs to the experimental fork path. The feature gate is
`FORK_SUBAGENT`. When active, omitting `subagent_type` triggers an implicit fork.

The fork path differs from the normal path:

- it reuses the parent system prompt instead of the selected agent prompt;
- it reuses the parent's exact tool set for prompt-cache consistency;
- it passes the parent message history as `forkContextMessages`;
- it builds a special final user message that closes the parent assistant's
  tool calls with placeholder `tool_result` blocks and appends a child
  directive.

This is not simply "copy all context and replace the system prompt." The system
prompt stays the parent's. The new child task is appended as a user-level
directive after the fork prefix.

## Sidechain Trace

Subagent execution is recorded separately from the parent conversation. The
child run uses the same `query()` loop, but its messages are written to a
sidechain transcript keyed by `agentId`.

This gives the parent a compressed interface: the parent sees launch/result
messages, while the full child tool trace remains available only if the runtime
or parent explicitly reads the sidechain.

## Synchronous Return

For a synchronous subagent, Claude Code waits for the child loop to complete and
then calls `finalizeAgentTool()`.

The finalizer:

1. finds the last assistant message in the child trace;
2. extracts only `type === "text"` blocks;
3. if the last assistant message has no text, falls back to the most recent
   assistant message with text;
4. adds metadata such as token count, tool-use count, duration, and agent id.

The tool result mapper then returns that extracted text to the parent as the
original `Agent` tool call's `tool_result`.

For one-shot built-in agents such as `Explore` and `Plan`, Claude Code omits the
continuation trailer and returns just the child text content. Other agents may
append an `agentId` and usage block so the parent can continue the worker.

## Asynchronous Return

For a background subagent, the first return to the parent is only a launch
result. It includes the `agentId` and an output-file path. This is mapped into a
normal `tool_result` saying the async agent was launched.

When the background child finishes, Claude Code again finalizes the child text
with `finalizeAgentTool()`. It then enqueues a user-role task notification:

```xml
<task-notification>
<task-id>...</task-id>
<tool-use-id>...</tool-use-id>
<output-file>...</output-file>
<status>completed</status>
<summary>Agent "..." completed</summary>
<result>child final text</result>
<usage>...</usage>
</task-notification>
```

This message looks like a user message, but it is runtime-generated. The
coordinator prompt explicitly tells the parent agent to treat
`<task-notification>` as worker results, not as ordinary user input.

Background Bash follows the same two-stage pattern:

1. the original `Bash` tool call returns a normal `tool_result` with a
   background task id and output-file path;
2. when the command completes, `LocalShellTask` enqueues a
   `<task-notification>` user/meta message with status and summary.

This is not a tool hook in the `PreToolUse` / `PostToolUse` sense. It is a
runtime notification queue. The query loop drains queued notifications and
turns them into attachment/user-role messages on a later model turn.

## Interpretation

Claude Code's subagent contract is mostly text-mediated:

- the parent delegates with a structured `Agent` tool call;
- the child receives a type-specific system prompt plus a user task prompt;
- the child runs an independent tool loop;
- the parent receives final text, not the full child trace;
- async workers re-enter the parent through task-notification XML.
- ordinary tools, subagents, and background tasks all converge on the same
  message-level pattern: assistant `tool_use`, user-role runtime observation,
  then assistant continuation.

The important design move is context isolation. Subagents are used to spend
tokens and tool calls outside the parent attention space, then return a compact
text result. The result is structured enough for routing and UI state, but the
semantic payload remains an assistant text report rather than a typed result
schema.

## Simplified Agent Model

This source reading makes the core agent loop look smaller than it first
appears:

```text
context -> model -> answer | tool_use
tool_use -> runtime -> observation -> context -> model
answer -> completion gate -> done | continue
```

The classic ReAct termination rule is still the base model: tool calls continue
the loop, and no tool call means the model is proposing a final answer. Claude
Code follows this pattern. Its sophistication is mostly the runtime layer around
that small loop:

- tool surface control, including deferred tools and `ToolSearchTool`;
- tool result shaping before observations return to the model;
- context injection through memory, skills, IDE state, reminders, and
  XML-like tags;
- completion interception through Stop hooks and related gates;
- separation between model-visible messages and human UI output;
- subagent sidechains that isolate work and return compact text summaries.

So the practical lesson is not that an agent needs a complicated core
algorithm. The core is simple. Production quality comes from the surrounding
contracts: what enters context, what triggers another turn, what can safely
stop, what must be verified, and what the human UI should show separately from
what the model sees.
