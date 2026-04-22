# DeerFlow Agent Architecture Research

Last Updated: 2026-04-22

> **Research Methodology**: Source-level analysis of `bytedance/deer-flow` 2.0 at `/Users/linguanguo/dev/llm-agent-research/deer-flow` (submoduled from github.com/bytedance/deer-flow). DeerFlow 2.0 is a **ground-up rewrite** released 2026-02-28 that shares no code with v1 (`README.md:16-17`). The central question: what exactly is a "super agent harness" at code level, and where does DeerFlow's architecture sit relative to (a) LangGraph, (b) Dify's workflow platform, and (c) Claude Code's agent harness shape?

## Sources

**Agent Construction**
- `backend/langgraph.json:1-14` — LangGraph graph registration (`lead_agent` entry, async checkpointer)
- `backend/packages/harness/deerflow/agents/lead_agent/agent.py:1-358` — full lead agent factory
- `backend/packages/harness/deerflow/agents/__init__.py:1-22` — package exports
- `backend/packages/harness/deerflow/agents/thread_state.py:1-55` — `ThreadState` extending `langchain.agents.AgentState`

**Middleware System (the 18-layer stack)**
- `backend/packages/harness/deerflow/agents/lead_agent/agent.py:215-277` — `_build_middlewares()` composition
- `backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py:68-143` — shared base builder (`_build_runtime_middlewares`) reused by lead and subagents
- `backend/packages/harness/deerflow/agents/middlewares/*.py` — 17 middleware files, ~2,980 lines total

**Subagent Execution Engine**
- `backend/packages/harness/deerflow/subagents/executor.py:1-612` — full executor with three thread pools, isolated event loops, cooperative cancellation
- `backend/packages/harness/deerflow/subagents/executor.py:14,179-185` — each subagent is a second `create_agent` instance
- `backend/packages/harness/deerflow/subagents/executor.py:73-80` — `_scheduler_pool`, `_execution_pool`, `_isolated_loop_pool`

**Sandbox and Skills**
- `backend/packages/harness/deerflow/sandbox/sandbox.py:1-93` — abstract `Sandbox` interface
- `backend/packages/harness/deerflow/sandbox/middleware.py:21-83` — `SandboxMiddleware` lifecycle
- `backend/packages/harness/deerflow/skills/loader.py:25-103` — `load_skills()` scanning `skills/{public,custom}/`

**Runtime Modes**
- `backend/packages/harness/deerflow/client.py:1-120` — `DeerFlowClient` embedded runtime
- `backend/packages/harness/deerflow/client.py:35` — **reuses** `_build_middlewares` from lead agent
- `backend/app/channels/manager.py:1-30` — IM bridge over `langgraph-sdk` HTTP

**Hard Boundaries**
- `backend/tests/test_harness_boundary.py:1-46` — CI-enforced import firewall (`deerflow.*` must not import `app.*`)

**Documentation (quoted verbatim where cited)**
- `backend/CLAUDE.md:7-18` — product architecture description
- `backend/CLAUDE.md:113-137` — harness/app split rules
- `backend/CLAUDE.md:159-179` — canonical 18-middleware ordering

---

## 1. Positioning and the Diagnostic Finding

### 1.1 Marketing Claim

The README opens with (`README.md:12`):

> DeerFlow (Deep Exploration and Efficient Research Flow) is an open-source **super agent harness** that orchestrates sub-agents, memory, and sandboxes to do almost anything — powered by extensible skills.

"Super agent harness" is the positioning. DeerFlow 2.0 is explicitly a ground-up rewrite (`README.md:16-17`): v1 was a Deep Research framework, v2 is something else. Understanding what that "something else" actually is, at code level, is the research question.

### 1.2 Code-Level Finding — The Thinnest Possible Wrapper

At its core, DeerFlow's lead agent is **two calls to `langchain.agents.create_agent`** (`agent.py:341` and `agent.py:350`):

```python
return create_agent(
    model=create_chat_model(name=model_name, thinking_enabled=thinking_enabled),
    tools=get_available_tools(model_name=model_name, ...),
    middleware=_build_middlewares(config, model_name=model_name, agent_name=agent_name),
    system_prompt=apply_prompt_template(...),
    state_schema=ThreadState,
)
```

That is the whole agent. There is no custom `StateGraph`, no custom `Pregel` node wiring, no hand-rolled ReAct loop. Everything custom in DeerFlow is delivered through **four parameters** of a standard LangChain 1.0 factory: `tools`, `middleware`, `system_prompt`, `state_schema`.

This is the diagnostic finding. DeerFlow is **the thinnest imaginable LangChain-1.0 wrapper that can still deliver a full product**: its entire technical identity lives in (a) its middleware stack, (b) its tool registry, and (c) its sandbox/skills infrastructure. The graph topology is LangChain's, not DeerFlow's.

### 1.3 Why This Is Notable

Cross-referencing `langgraph.research.md` in this repo: LangGraph 1.0 deliberately deprecated `create_react_agent` and relocated the opinionated agent factory to `langchain.agents.create_agent`, with Plan-and-Execute moved into a `langchain.agents.middleware` "to-do list" pattern. At the time of that research, no major open-source product had been identified that **built a real product on that new middleware world**. DeerFlow 2.0 is that product — the first large ByteDance-backed system publicly betting on LangChain 1.0 middleware as the primary extension surface.

This changes the story for the "④b is empty" argument in this repo's blog: ④b (P&E as graph topology) remains empty, but **middleware-as-extension-surface is now a shipped production pattern**, not just a documented redirect.

---

## 2. The Base Abstraction — `create_agent` + `AgentMiddleware`

### 2.1 Graph Registration

`backend/langgraph.json:1-14` registers exactly one graph:

```json
{
  "python_version": "3.12",
  "graphs": { "lead_agent": "deerflow.agents:make_lead_agent" },
  "checkpointer": { "path": "./packages/harness/deerflow/agents/checkpointer/async_provider.py:make_checkpointer" }
}
```

The LangGraph Server loads `make_lead_agent` as a RunnableConfig-driven graph factory. From LangGraph's perspective, DeerFlow is a single graph — the agent identity, the subagents, the skills, the sandbox, none of these are graph nodes. They are all composed inside the one agent that `create_agent` returns.

### 2.2 Middleware Hook Surface

A grep over the 17 middleware files (`agents/middlewares/*.py`) enumerates every hook used in DeerFlow:

```
before_agent, after_agent         — per-invocation lifecycle
before_model, after_model         — per-LLM-call hooks
wrap_model_call                   — decorator around the LLM invocation
wrap_tool_call, awrap_tool_call   — decorator around tool execution (sync/async)
```

These are the hooks exposed by `langchain.agents.middleware.AgentMiddleware`. The harness package lives entirely within this hook surface — it never reaches into LangGraph's Pregel layer directly. Every "feature" of DeerFlow is expressible as one or more of these six hooks plus some state schema additions.

### 2.3 State Schema

`ThreadState` (`agents/thread_state.py:48-55`) extends LangChain's `AgentState` with:

```python
class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]
    thread_data: NotRequired[ThreadDataState | None]
    title: NotRequired[str | None]
    artifacts: Annotated[list[str], merge_artifacts]
    todos: NotRequired[list | None]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

Two custom reducers at `thread_state.py:21-45` — `merge_artifacts` (order-preserving dedup) and `merge_viewed_images` (dict merge, empty-dict means clear). The reducers are LangGraph's channel composition primitive, used minimally. This is the only point where DeerFlow touches LangGraph-layer concepts directly.

---

## 3. The Middleware Chain Is the Architecture

### 3.1 The 18-Layer Composition

`_build_middlewares()` at `agent.py:215-277` assembles the chain in strict insertion order. The shared base is `build_lead_runtime_middlewares(lazy_init=True)` at `tool_error_handling_middleware.py:128-134`, which expands through `_build_runtime_middlewares` at `tool_error_handling_middleware.py:68-125`.

The canonical ordering (with optional flags):

| # | Middleware | Hook(s) used | When added |
|---|---|---|---|
| 1 | `ThreadDataMiddleware` | `before_agent` | always |
| 2 | `UploadsMiddleware` | `before_agent` | always (lead only; `include_uploads=True`) |
| 3 | `SandboxMiddleware` | `before_agent` / `after_agent` | always |
| 4 | `DanglingToolCallMiddleware` | `before_model` | always (repair state mid-flight) |
| 5 | `LLMErrorHandlingMiddleware` | `wrap_model_call` | always |
| 6 | `GuardrailMiddleware` | `wrap_tool_call` | optional (`guardrails.enabled`) |
| 7 | `SandboxAuditMiddleware` | `wrap_tool_call` | always |
| 8 | `ToolErrorHandlingMiddleware` | `wrap_tool_call` / `awrap_tool_call` | always |
| 9 | `DeerFlowSummarizationMiddleware` | `before_model` | optional (`summarization.enabled`) |
| 10 | `TodoMiddleware` | prompt + tool injection | optional (`is_plan_mode`) |
| 11 | `TokenUsageMiddleware` | `after_model` | optional (`token_usage.enabled`) |
| 12 | `TitleMiddleware` | `after_model` | always |
| 13 | `MemoryMiddleware` | `after_agent` | always |
| 14 | `ViewImageMiddleware` | `before_model` | conditional (model `supports_vision`) |
| 15 | `DeferredToolFilterMiddleware` | `before_model` | optional (`tool_search.enabled`) |
| 16 | `SubagentLimitMiddleware` | `after_model` | optional (`subagent_enabled`) |
| 17 | `LoopDetectionMiddleware` | `after_model` | always |
| 18 | `ClarificationMiddleware` | terminator — must be last | always |

The 17 middleware files sum to ~2,980 LoC (measured by `wc -l` over `agents/middlewares/`). **This is DeerFlow's actual code mass on the agent-execution axis.** It is not a thin layer; it is the product.

### 3.2 What This Looks Like Architecturally

In LangGraph's `create_react_agent` classical model, the cycle is: `agent → tools → agent → ...`, terminated when the LLM produces no tool calls. In DeerFlow's model, the same cycle runs inside `create_agent`, but **each arc is wrapped by the middleware chain**:

- Before each LLM call: `DanglingToolCall` repairs tool-call pairing, `Summarization` compresses context if over threshold, `ViewImage` injects base64, `DeferredToolFilter` hides long-tail tools until search is used.
- Around each LLM call: `LLMErrorHandling` catches provider failures and normalizes them into recoverable errors.
- After each LLM call: `TokenUsage` records metrics, `Title` auto-generates thread title, `SubagentLimit` truncates excess `task` calls to 3, `LoopDetection` hard-stops repeat tool loops.
- Around each tool call: `Guardrail` pre-authorizes, `SandboxAudit` logs, `ToolErrorHandling` catches exceptions into `ToolMessage`s.
- On every invocation: `ThreadData`, `Uploads`, `Sandbox` set up per-thread isolated workspace, then tear down.
- At terminal boundary: `Clarification` intercepts `ask_clarification` tool calls and raises `Command(goto=END)` to pause for user input.

Any imperative "runner" style framework (CrewAI, AgentScope, Google ADK) would encode each of these as a method inside some Agent class. DeerFlow externalizes every one into an independent middleware object with a narrow hook contract. **This is the single largest architectural choice that distinguishes DeerFlow from the other frameworks in this repo.**

### 3.3 Guardrail as Reflection-Loaded Plugin

`tool_error_handling_middleware.py:97-119` shows that `GuardrailMiddleware` is loaded via `deerflow.reflection.resolve_variable(config.provider.use)` — exactly the same pattern Dify uses for its plugin node strategies. Three provider options documented: built-in `AllowlistProvider`, third-party `aport-agent-guardrails`, or user-supplied. This makes DeerFlow's guardrail architecture more similar to Dify's plugin Protocol than to LangGraph's opinionated prebuilts.

---

## 4. Harness / App Split with CI-Enforced Firewall

### 4.1 The Rule

`backend/CLAUDE.md:115-120` states:

> - **Harness** (`packages/harness/deerflow/`): Publishable agent framework package (`deerflow-harness`). Import prefix: `deerflow.*`.
> - **App** (`app/`): Unpublished application code. Import prefix: `app.*`.
>
> **Dependency rule**: App imports deerflow, but deerflow never imports app.

### 4.2 Enforcement

`tests/test_harness_boundary.py:37-46` walks all `.py` files under `packages/harness/deerflow/`, parses each with `ast.parse`, and fails the CI if any `Import` or `ImportFrom` node has a module starting with `app.`:

```python
for py_file in sorted(HARNESS_ROOT.rglob("*.py")):
    for lineno, module in _collect_imports(py_file):
        if any(module == prefix.rstrip(".") or module.startswith(prefix) for prefix in BANNED_PREFIXES):
            violations.append(f"  {rel}:{lineno}  imports {module}")
assert not violations, "Harness layer must not import from app layer:\n" + "\n".join(violations)
```

This is not a stylistic guideline; it is an AST-based test that runs on every PR. Among the frameworks analyzed in this repo, only Dify has a comparable hard boundary (its plugin daemon RPC), and Dify's boundary is a process boundary, not a test-enforced package boundary.

### 4.3 Why It Matters

The firewall means `deerflow-harness` is genuinely publishable as a standalone package. A downstream consumer can `pip install deerflow-harness` and build a different application — different channels, different Gateway, different UI — on top. This is structurally closer to the `langchain.agents` package philosophy than to a monolithic open-source product.

---

## 5. Three Runtime Modes, One Middleware Stack

DeerFlow runs the same agent code in three configurations:

| Mode | Process topology | Agent host | Invoked via |
|---|---|---|---|
| **Standard** (`make dev`) | 4 processes: LangGraph Server + Gateway + Frontend + Nginx | LangGraph Server (port 2024) | `langgraph-sdk` HTTP |
| **Gateway-embedded** (`make dev-pro`) | 3 processes: Gateway + Frontend + Nginx | Embedded in Gateway via `RunManager` + `run_agent()` + `StreamBridge` | FastAPI routes |
| **Embedded Python** (`DeerFlowClient`) | 1 process: consumer code | In-process | `client.chat()` / `client.stream()` |

The embedded client at `client.py:35` has a single revealing import:

```python
from deerflow.agents.lead_agent.agent import _build_middlewares
```

**`DeerFlowClient` reuses the exact same `_build_middlewares()` function that the LangGraph Server uses.** All three runtime modes instantiate `create_agent()` with the same middleware list, the same `ThreadState`, the same system prompt template. The difference between the modes is purely transport: how the HTTP/streaming surface is wired, and whether there is a LangGraph checkpointer attached.

This convergence is the mechanical justification for the harness/app split. Without the import firewall, it would be tempting for the embedded client or the Gateway to reach into app-layer routers. The firewall keeps harness code runnable in any of the three hosts without modification.

---

## 6. Subagent Execution — A Second Full Agent

### 6.1 What a Subagent Actually Is

The `task` tool delegates to `SubagentExecutor` (`subagents/executor.py:128-529`). The critical line is at `executor.py:14,179-185`:

```python
from langchain.agents import create_agent
...
def _create_agent(self):
    model = create_chat_model(name=model_name, thinking_enabled=False)
    middlewares = build_subagent_runtime_middlewares(lazy_init=True)  # reuse lead's base
    return create_agent(
        model=model,
        tools=self.tools,
        middleware=middlewares,
        system_prompt=self.config.system_prompt,
        state_schema=ThreadState,
    )
```

**A subagent is a second `create_agent` instance**, with its own middleware stack (via `build_subagent_runtime_middlewares` at `tool_error_handling_middleware.py:137-143`, which reuses the same `_build_runtime_middlewares` base but with `include_uploads=False`). This is *not* like Claude Code's subagents, which are lightweight new LLM calls with a different system prompt. In DeerFlow, a subagent reconstructs the full middleware-wrapped graph — including Sandbox, ThreadData, ToolErrorHandling, Guardrails, SandboxAudit.

### 6.2 Three Thread Pools

`executor.py:73-80`:

```python
_scheduler_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-scheduler-")
_execution_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-exec-")
_isolated_loop_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-isolated-")
```

| Pool | Role |
|---|---|
| `_scheduler_pool` | accepts `execute_async()` requests, flips status to RUNNING, then submits to the execution pool with a timeout future |
| `_execution_pool` | actually runs `self.execute()` (which wraps `_aexecute` in `asyncio.run`); timeout enforced here |
| `_isolated_loop_pool` | used *only* when `execute()` is called from within an already-running event loop. Spawns a new thread, sets a fresh event loop, runs `_aexecute` inside it |

The `_isolated_loop_pool` exists for a specific reason documented at `executor.py:380-416`: `langchain.agents.create_agent` creates `httpx` clients bound to a specific asyncio loop. If the parent agent is async and calls `task()` synchronously, nesting `asyncio.run` in the same loop breaks. The fix is to detach into a brand-new thread with a fresh loop. This is a subtle but real cost of reusing `create_agent` for subagents.

### 6.3 Cooperative Cancellation

`executor.py:260-272`: cancellation is checked at `astream()` chunk boundaries via `result.cancel_event.is_set()`. A long-running tool call within a single iteration cannot be interrupted — cancellation resolves on the next yielded chunk. `executor.py:532` hard-codes `MAX_CONCURRENT_SUBAGENTS = 3` and `SubagentLimitMiddleware` truncates excess `task` tool calls from the AIMessage at `after_model`.

### 6.4 Cost Profile vs Claude Code

| Dimension | DeerFlow subagent | Claude Code subagent |
|---|---|---|
| Instantiation | Full `create_agent` + middleware chain | New LLM call with substituted system prompt |
| Tool filter | `_filter_tools` (allow/disallow lists) on full tool registry | Declared per-agent in frontmatter |
| Sandbox | Shared sandbox state passed through | Same main sandbox |
| Concurrency | 3 hard cap; threading + isolated loop complexity | Implementation-dependent |
| Loop detection | Own `LoopDetectionMiddleware` | Main agent's state/retry logic |

The cost and complexity of DeerFlow's subagent model is non-trivial compared to Claude Code's prompt-substitution approach. The tradeoff: DeerFlow subagents inherit the *full* guardrail/audit/error-handling stack, which matters in a multi-tenant production setting.

---

## 7. Sandbox Abstraction and Virtual Paths

### 7.1 Interface

`sandbox/sandbox.py:6-93` — abstract `Sandbox` class with seven methods: `execute_command`, `read_file`, `write_file`, `update_file`, `list_dir`, `glob`, `grep`. This is close to the set of operations Claude Code's bash/str-replace/read/write tools expose to its model. The provider pattern (`SandboxProvider.acquire / get / release`) decouples the identity of the sandbox (referenced in `ThreadState.sandbox.sandbox_id`) from its physical implementation.

### 7.2 Three Implementations

Per `backend/CLAUDE.md:228-234`:

- `LocalSandboxProvider` — singleton local filesystem with path mappings
- `AioSandboxProvider` (in `community/`) — Docker-based isolated sandbox
- A Kubernetes provisioner mode (provisioner service on port 8002, started only in k8s mode)

### 7.3 Virtual Path System

`backend/CLAUDE.md:236-240`:

> **Virtual Path System**:
> - Agent sees: `/mnt/user-data/{workspace,uploads,outputs}`, `/mnt/skills`
> - Physical: `backend/.deer-flow/threads/{thread_id}/user-data/...`, `deer-flow/skills/`
> - Translation: `replace_virtual_path()` / `replace_virtual_paths_in_command()`

The LLM always sees the same virtual paths regardless of sandbox backend. In local mode, `bash` commands are rewritten by `replace_virtual_paths_in_command` before execution. In docker mode, the virtual paths are volume mounts. The agent's prompt and tools stay identical across modes — **only the physical binding differs**. This is the same architectural idea Claude Code uses for its sandbox virtualization.

### 7.4 Lifecycle Middleware

`sandbox/middleware.py:21-83` — `SandboxMiddleware(lazy_init=True)` acquires the sandbox on the first tool call (not on `before_agent`), reuses it across turns within the same thread, and releases on `after_agent`. Lazy init saves sandbox creation cost for conversation turns that don't require tools.

---

## 8. Skills System — Claude Code Format on LangGraph

### 8.1 Discovery and Format

`skills/loader.py:25-103`:

```python
for category in ["public", "custom"]:
    category_path = skills_path / category
    ...
    for current_root, dir_names, file_names in os.walk(category_path, followlinks=True):
        if "SKILL.md" not in file_names:
            continue
        skill_file = Path(current_root) / "SKILL.md"
        skill = parse_skill_file(skill_file, category=category, relative_path=relative_path)
```

Skills live at `deer-flow/skills/{public,custom}/<name>/SKILL.md`. Each `SKILL.md` has YAML frontmatter (name, description, license, allowed-tools per `CLAUDE.md:293`). `public/` is committed; `custom/` is `.gitignored`. This is the same layout and file format Claude Code uses for its skills.

### 8.2 Enable State

Not in the file, but in `extensions_config.json` under the `skills` key (map of skill name → `{enabled: true/false}`). `skills/loader.py:87-92` reads the config and sets `skill.enabled` accordingly. Gateway API `PUT /api/skills/{name}` updates the config. This separates *availability* (file on disk) from *activation* (config flag).

### 8.3 Injection

Per `backend/CLAUDE.md:294`: enabled skills are listed in the agent system prompt with container paths. Agents see skill content inside the sandbox at `/mnt/skills/{category}/{name}/`. So activation is a two-part mechanism: the skill name enters the system prompt (so the LLM knows it exists), and the skill directory is mounted into the sandbox (so tool calls can read it). This is identical in shape to Claude Code's skill mechanism — the only difference is where the sandbox lives.

### 8.4 Installation

`POST /api/skills/install` (`backend/CLAUDE.md:219`) accepts a `.skill` ZIP archive and extracts it to `custom/`. This is the installation path that makes skills "extensible" at runtime without restart.

---

## 9. IM Channel Layer — LangGraph Server as Protocol

### 9.1 Architectural Role

`app/channels/manager.py:1-30` imports:

```python
import httpx
from langgraph_sdk.errors import ConflictError
from app.channels.commands import KNOWN_CHANNEL_COMMANDS
from app.channels.message_bus import InboundMessage, ...
```

Note: **no `deerflow.*` imports**. The channel layer does not reach into the harness. It talks to the agent via `langgraph-sdk` HTTP only (`DEFAULT_LANGGRAPH_URL = "http://localhost:2024"`, `DEFAULT_ASSISTANT_ID = "lead_agent"` at `manager.py:22-25`).

### 9.2 Why This Matters

From the harness's perspective, there is no special IM support code. A Telegram message, a Feishu event, and a browser frontend request all hit the same LangGraph Server endpoint with the same `thread_id` scheme. The only difference between a chat from Feishu and a chat from the web UI is which `Channel` subclass is on the other side and what `assistant_id` / `context` the run was started with.

The harness/app split described in §4 is what makes this clean. The channels layer in `app/` treats the harness as a pure HTTP service. If you replaced `app/channels/` with a different set of integrations (e.g., Discord, Mattermost), the harness would not need to know.

### 9.3 Supported Platforms

Per `README.md:366-372`: Telegram (Bot API long-polling), Slack (Socket Mode), Feishu/Lark (WebSocket), WeChat (Tencent iLink), WeCom (WebSocket). All five use long-poll or socket transports, so no public IP is required — this is deployment-friendly for enterprise on-prem.

---

## 10. Memory System — Brief

Covered in detail in this repo's sister `llm-memory-research/` and cross-referenced here for completeness.

`backend/CLAUDE.md:343-370` describes the memory pipeline:

1. `MemoryMiddleware` (entry #13 in the middleware chain) filters messages to (user input + final AI response) and enqueues for update.
2. Queue at `memory/queue.py` debounces (default 30s), deduplicates per-thread, batches.
3. Background thread invokes an LLM to extract fact updates.
4. Atomic write to `backend/.deer-flow/memory.json` (temp file + rename), with cache invalidation.
5. Next interaction injects top 15 facts + context into `<memory>` tags in the system prompt.

Data model: `workContext`, `personalContext`, `topOfMind` (narrative 1-3 sentence summaries) plus discrete facts with `id`, `content`, `category`, `confidence`, `createdAt`, `source`. This hybrid narrative + atomic-fact schema is worth flagging alongside the memory-architecture discussion in `llm-memory-research` — it is closer to Claude Code's memory file pattern than to LangGraph's check-pointed state.

---

## 11. Direct Comparison with LangGraph / Dify / Claude Code

### 11.1 Control-Flow Substrate

| Framework | Substrate | Customization surface |
|---|---|---|
| **LangGraph** (raw) | `StateGraph` + `Pregel` BSP engine | Graph nodes + edges; `create_react_agent` is a two-node graph |
| **Dify** | `graphon.GraphEngine` (external package), queue-based worker pool | JSON DAG + plugin-daemon for agent strategies |
| **Claude Code** (closed) | Proprietary agent runner with middleware-style extensions | Skills, hooks, subagents; single-agent loop |
| **DeerFlow 2.0** | `langchain.agents.create_agent` (one graph) + 18 middlewares | AgentMiddleware hooks + tool registry |

### 11.2 Architectural Shape

| Axis | DeerFlow | LangGraph prebuilts | Dify | Claude Code |
|---|---|---|---|---|
| Is there a Plan artifact? | Optional (`TodoMiddleware` when `is_plan_mode`) | Optional (langchain `create_agent` to-do middleware) | Optional (TodoList-style plugin) | Optional (TodoWrite tool) |
| Multi-agent composition | Flat: lead + subagents via `task()` | Subgraph composition + `Send` | Workflow DAG + AgentNode | Lead + Task() subagents |
| Sandbox abstraction | First-class, 3 backends | None | Node-level via tool | First-class, one backend |
| Skills | Claude Code format, runtime install | None | Plugin daemon | Native |
| IM channels | First-class, 5 platforms | None (user-built) | Limited plugins | None (CLI only) |
| Extension surface | AgentMiddleware + reflection config | StateGraph nodes | Plugin daemon RPC | Hook scripts |

### 11.3 The "Claude Code on LangGraph" Framing

DeerFlow's product surface — lead agent + `task()` subagents + skills + sandbox + todos + clarification — is a near bit-for-bit replica of Claude Code's architectural shape. The distinguishing fact is the *mechanism*: Claude Code is a proprietary agent runner with purpose-built internals. DeerFlow realizes the same shape on top of LangChain 1.0's `create_agent` + `AgentMiddleware`, with no custom graph code of its own.

This is the most interesting architectural observation in this research: **"Claude Code's product shape is expressible as ~20 LangChain middlewares."** That is a substantive claim. It suggests the Claude Code harness shape is not tied to a proprietary runtime — it is a portable pattern that can live on any framework that exposes enough middleware hooks.

### 11.4 The DeerFlow / Dify Asymmetry

Both are ByteDance/Chinese-ecosystem-adjacent open-source AI platforms. Superficially they occupy similar space. At code level they are radically different:

- **Dify**: visual DAG editor, multi-tenant queue engine (`graphon`), plugin daemon for agent strategies, Redis command channel. The agent is a *node* in a workflow.
- **DeerFlow**: no visual editor, single agent with middleware stack, no plugin daemon (but reflection-loaded guardrails), LangGraph checkpointer for state. The workflow *is* the agent.

Dify is Direction ② (Workflow Platform) in this repo's taxonomy. DeerFlow is Direction ③/④ (Code-First Agent Framework / Harness). They are not competitors in the same segment — they are on different axes of the Agent Map.

---

## 12. Position on the Agent Architecture Spectrum

### 12.1 By Mechanism

DeerFlow is a **single-agent harness** (④a by this repo's taxonomy) built on `langchain.agents.create_agent`. The ReAct cycle is standard. There is **no Plan-and-Execute graph topology** — `TodoMiddleware` is a prompting and tool-injection mechanism, not a structural plan-execute split. There is no hierarchical agent composition in the LangGraph sense — subagents are invoked via `task()` as tool calls, not as subgraphs.

### 12.2 By Product Form

DeerFlow is a **deployable product** (Docker, k8s, local), not a developer library. It ships a full stack: frontend (Next.js), reverse proxy (Nginx), API gateway (FastAPI), agent runtime (LangGraph Server), optional sandbox provisioner, and five IM integrations. This makes it structurally comparable to Dify at the product layer, even though its architecture is in a different family.

### 12.3 Contribution to This Repo's Taxonomy

Before DeerFlow, the map was:

- Direction ① Experimental / Research → AutoGPT, BabyAGI (2023)
- Direction ② Workflow Platform → Dify, Coze, Bailian, Mastra-with-editor
- Direction ③ Code-First Framework → LangGraph, AgentScope, Google ADK, CrewAI, Pydantic-AI, Mastra
- Direction ④a Single-Agent Harness (prebuilt) → `create_react_agent`, various ReAct agent libraries
- Direction ④b P&E Graph Topology → empty (relocated to middleware per `langgraph.research.md`)
- Direction ⑤ P&E + Typed Dataflow (user's production agent) → personal

DeerFlow is a specimen that extends the map in a new way: **Direction ④a realized as a full product with a Claude Code-shaped feature surface, implemented via middleware rather than custom runtime**. This strengthens the ④b-is-silent-but-middleware-is-alive story from the LangGraph research: LangChain 1.0's middleware API is not just a documentation redirect, it has at least one major open-source production adopter.

---

## Key Architectural Insights

**1. `create_agent` + `AgentMiddleware` is a viable production abstraction.** DeerFlow is the first large open-source product to build a full harness on top of LangChain 1.0's new middleware API rather than rolling custom graph code. This is architectural evidence that the LangGraph → LangChain middleware migration (documented in `langgraph.research.md`) is landing with real adopters.

**2. A Claude Code-shaped harness is expressible in ~20 LangChain middlewares.** Lead agent, subagents via `task()`, skills, sandbox, todos, clarification, memory, guardrails — every element of the Claude Code product shape is realized as a middleware class or a tool registration. The Claude Code pattern is not tied to a proprietary runner; it is portable.

**3. The harness/app split with CI-enforced import firewall is unusual and load-bearing.** `test_harness_boundary.py` makes `deerflow-harness` genuinely publishable as a standalone library. The firewall is what enables three runtime modes (LangGraph Server, Gateway-embedded, `DeerFlowClient`) to share the same middleware stack without modification. This pattern is uncommon in open-source agent frameworks — most either lack the separation or enforce it only by convention.

**4. Subagents are *full* agents, not lightweight prompts.** A DeerFlow subagent is a second `create_agent` instance with its own middleware chain. This is heavier than Claude Code's prompt-substitution subagents, but gives subagents the full guardrail/audit/error-handling stack. Three thread pools (scheduler + execution + isolated-loop) and cooperative cancellation at `astream()` boundaries are the concrete cost. `MAX_CONCURRENT_SUBAGENTS = 3` is hardcoded in `executor.py:532`.

**5. Marketing claim "super agent harness" is precise, not hyperbolic.** Harness, in the Claude Code sense, means the outer framework that manages the loop, tools, and lifecycle while remaining open to extensions. DeerFlow's middleware stack *is* exactly that structure — with an unusually disciplined claim that 100% of the custom behavior is expressible via LangChain 1.0's formal hook surface.

**6. The true novelty is not in the features, but in the layering.** Every individual feature in DeerFlow exists in some form elsewhere (skills → Claude Code; sandbox → Code Interpreter / e2b; todos → Claude Code; subagents → CrewAI-style; guardrails → aport / promptarmor; IM channels → many vendors). The novelty is realizing all of them as middlewares on a shared `create_agent` substrate with a hard harness/app split. This is a specific architectural bet that pays off if LangChain 1.0's middleware surface is stable; it is a significant risk if that surface changes.

---

## Conclusion

DeerFlow 2.0 is a **Claude Code-shaped single-agent harness realized as a stack of ~20 LangChain 1.0 middlewares on top of a single `create_agent` call**, with three runtime hosts (LangGraph Server, Gateway-embedded, embedded Python client) sharing the same middleware construction, separated from the application layer (FastAPI Gateway + five IM channels) by a CI-enforced import firewall.

For this repo's taxonomy, DeerFlow is:
- **Direction ④a** on the control-flow axis (single-agent ReAct, not P&E, not graph-as-plan)
- **Direction "product"** on the packaging axis (Docker-deploy, not library)
- A **proof point** for LangChain 1.0's middleware-as-extension-surface direction — the first major open-source bet on that pattern

For the blog's central argument about ④b being empty: DeerFlow does not fill ④b (there is no Plan-and-Execute graph topology here). What it does do is demonstrate that **the LangGraph-to-LangChain-middleware migration from `langgraph.research.md` is not vapor** — real production software is being built on `create_agent` + `AgentMiddleware`, and the result is structurally close to Claude Code's product shape but built on an open foundation.

The key takeaway for readers of this repo: if you want to understand what a "super agent harness" looks like in 2026 when built on open-source LangChain 1.0 infrastructure, read DeerFlow's middleware directory and `_build_middlewares()`. It is 2,980 lines of middleware code that reproduce Claude Code's shape without writing any custom graph execution code. The choice between "rewrite the runner" (Mastra, AgentScope, ADK, Claude Code) and "stack middlewares on a standard `create_agent`" (DeerFlow) is now a real architectural decision with a shipped exemplar on each side.

**Essential Files for Understanding DeerFlow's Agent Architecture:**

- `backend/langgraph.json` — graph registration (`make_lead_agent`, async checkpointer)
- `backend/packages/harness/deerflow/agents/lead_agent/agent.py` — two `create_agent` calls + `_build_middlewares`
- `backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py` — shared middleware base (`_build_runtime_middlewares`)
- `backend/packages/harness/deerflow/agents/thread_state.py` — state schema extending `AgentState`
- `backend/packages/harness/deerflow/subagents/executor.py` — subagents as second `create_agent`, three thread pools, isolated loops
- `backend/packages/harness/deerflow/sandbox/{sandbox.py,middleware.py}` — sandbox interface and lifecycle
- `backend/packages/harness/deerflow/skills/loader.py` — Claude Code-format skill discovery
- `backend/packages/harness/deerflow/client.py` — embedded runtime reusing `_build_middlewares`
- `backend/app/channels/manager.py` — IM bridge via `langgraph-sdk` only (no harness imports)
- `backend/tests/test_harness_boundary.py` — CI-enforced harness/app firewall

**Cross-References:**
- `langgraph.research.md` — for the LangChain 1.0 middleware migration context
- `dify.research.md` — for the contrast between workflow-platform and agent-harness product categories
- `/Users/linguanguo/dev/llm-memory-research/` — for the memory-management axis of DeerFlow, and for Claude Code / Codex / Cursor harness comparisons
