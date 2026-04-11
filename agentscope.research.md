# AgentScope Agent Architecture Research

Last Updated: 2026-04-12

> **Research Methodology**: Source-level analysis of agentscope-ai/agentscope at `/Users/linguanguo/dev/llm-agent-research/agentscope`. Previous README-level analysis placed this firmly in ④a (ReAct school). This source-level analysis specifically investigated whether `src/agentscope/plan/` changes that conclusion. **It does — AgentScope is the only framework found so far that has genuine Plan/SubTask Pydantic data models, even if the execution model is still ReAct.**

## Sources

**Core Agent Files**
- `src/agentscope/agent/__init__.py` — Agent class exports
- `src/agentscope/agent/_agent_base.py:30-80` — AgentBase class definition
- `src/agentscope/agent/_react_agent_base.py:1-117` — ReActAgentBase with reasoning/acting split
- `src/agentscope/agent/_react_agent.py:98-797` — ReActAgent full implementation

**The plan/ Module (Core Subject)**
- `src/agentscope/plan/__init__.py` — Module exports: SubTask, Plan, PlanNotebook, DefaultPlanToHint, PlanStorageBase, InMemoryPlanStorage
- `src/agentscope/plan/_plan_model.py:1-201` — SubTask and Plan Pydantic models
- `src/agentscope/plan/_plan_notebook.py:1-905` — PlanNotebook: tool provider + hint generator
- `src/agentscope/plan/_storage_base.py` — PlanStorageBase abstract class
- `src/agentscope/plan/_in_memory_storage.py` — InMemoryPlanStorage default implementation

**Pipeline Module**
- `src/agentscope/pipeline/_functional.py:1-193` — sequential_pipeline, fanout_pipeline
- `src/agentscope/pipeline/_class.py:1-80` — SequentialPipeline, FanoutPipeline classes
- `src/agentscope/pipeline/_msghub.py:1-157` — MsgHub broadcast context manager

**Examples**
- `examples/agent/react_agent/main.py` — Canonical hello-agent example
- `examples/agent/meta_planner_agent/main.py` — Plan-enabled meta planner
- `examples/functionality/plan/main_agent_managed_plan.py` — Agent self-manages plan
- `examples/functionality/plan/main_manual_plan.py` — Programmer pre-creates plan
- `docs/tutorial/en/src/task_plan.py` — Official plan tutorial

---

## 1. Agent Definition (API Layer)

AgentScope's primary agent class is `ReActAgent`, defined in `src/agentscope/agent/_react_agent.py:98`. The inheritance hierarchy:

```
AgentBase  (StateModule, _agent_base.py:30)
  └── ReActAgentBase  (_react_agent_base.py:12)
        └── ReActAgent  (_react_agent.py:98)
```

`AgentBase` provides hooks: `pre_reply`, `post_reply`, `pre_print`, `post_print`, `pre_observe`, `post_observe`. `ReActAgentBase` adds `pre_reasoning`, `post_reasoning`, `pre_acting`, `post_acting` and declares abstract `_reasoning()` / `_acting()`.

`ReActAgent.__init__` (`_react_agent.py:177-348`) requires `name`, `sys_prompt`, `model`, `formatter`; optional `toolkit`, `memory`, `long_term_memory`, `plan_notebook: PlanNotebook | None`, `max_iters: int = 10`.

Exported agent types (`agent/__init__.py`): `AgentBase`, `ReActAgentBase`, `ReActAgent`, `UserAgent`, `A2AAgent`, `RealtimeAgent`. **There is no standalone `PlannerAgent` or `PlanExecutorAgent` class** — Plan capability is delivered exclusively as a parameter to `ReActAgent`.

---

## 2. Execution Model: ReActAgent (Source-Level Confirmation)

`ReActAgent.reply()` (`_react_agent.py:376-537`) implements a bounded loop:

```python
for _ in range(self.max_iters):                          # _react_agent.py:432
    await self._compress_memory_if_needed()

    msg_reasoning = await self._reasoning(tool_choice)   # LLM call

    futures = [
        self._acting(tool_call)
        for tool_call in msg_reasoning.get_content_blocks("tool_use")
    ]
    if self.parallel_tool_calls:
        structured_outputs = await asyncio.gather(*futures)
    else:
        structured_outputs = [await _ for _ in futures]

    # Exit: no tool calls means task done
    elif not msg_reasoning.has_content_blocks("tool_use"):
        reply_msg = msg_reasoning
        break
```

Exit conditions:
1. LLM emits no tool calls → loop ends (`_react_agent.py:513-518`)
2. `max_iters` exhausted → `_summarizing()` forced
3. For structured output: `generate_response` tool called and validated → loop ends

**There is no pre-loop planning phase.** The LLM decides whether to use tools on every iteration.

---

## 3. The plan/ Module — Core Question of This Research

### 3.1 Structure

The `src/agentscope/plan/` directory contains five files:

| File | Core Class | Role |
|------|-----------|------|
| `_plan_model.py` | `SubTask`, `Plan` | Pydantic data models with state machine |
| `_plan_notebook.py` | `PlanNotebook`, `DefaultPlanToHint` | Tool provider + context hint generator |
| `_storage_base.py` | `PlanStorageBase` | Abstract CRUD interface for plan history |
| `_in_memory_storage.py` | `InMemoryPlanStorage` | Default in-memory implementation |
| `__init__.py` | — | Re-exports all above |

**`SubTask`** (`_plan_model.py:11-101`): Pydantic `BaseModel` with `name`, `description`, `expected_outcome`, `outcome`, `state: Literal["todo", "in_progress", "done", "abandoned"]`, timestamps. `finish(outcome)` transitions state to `"done"`.

**`Plan`** (`_plan_model.py:104-200`): Pydantic `BaseModel` with `id`, `name`, `description`, `expected_outcome`, `subtasks: list[SubTask]`, `state`, timestamps. `refresh_plan_state()` syncs plan state from subtask states.

**`PlanNotebook`** (`_plan_notebook.py:172-905`): manages `current_plan: Plan | None` and exposes:
- 8 tool functions for LLM use: `create_plan`, `view_subtasks`, `update_subtask_state`, `finish_subtask`, `revise_current_plan`, `finish_plan`, `view_historical_plans`, `recover_historical_plan`
- `get_current_hint()` — generates a `<system-hint>` message based on plan state
- `list_tools()` — returns all 8 tool callables for registration into `Toolkit`
- `register_plan_change_hook()` — callback for UI visualization

### 3.2 Is Plan a First-Class Primitive or a ReAct Tool?

**How Plan is integrated into ReActAgent** (`_react_agent.py:326-348`):

```python
if plan_notebook:
    self.plan_notebook = plan_notebook
    if enable_meta_tool:
        self.toolkit.create_tool_group("plan_related", ...)
        for tool in plan_notebook.list_tools():
            self.toolkit.register_tool_function(tool, group_name="plan_related")
    else:
        for tool in plan_notebook.list_tools():
            self.toolkit.register_tool_function(tool)
```

Plan tools are registered into the agent's `Toolkit` — the exact same container as domain tools like `execute_shell_command`. They are peers, not a separate layer.

**How hints are injected** (`_react_agent.py:546-566`, inside `_reasoning()`):

```python
if self.plan_notebook:
    hint_msg = await self.plan_notebook.get_current_hint()
    await self.memory.add(hint_msg, marks=_MemoryMark.HINT)

prompt = await self.formatter.format(msgs=[
    Msg("system", self.sys_prompt, "system"),
    *await self.memory.get_memory(...),   # includes hint_msg
])
await self.memory.delete_by_mark(mark=_MemoryMark.HINT)  # ephemeral
```

The hint is injected into working memory before the LLM call and deleted immediately after. It is **ephemeral context**, not a durable state machine on the execution path. The framework does not act on the hint — it trusts the LLM to read it and respond accordingly.

**The mechanism in full**:
1. User message → `reply()` enters the ReAct loop
2. Each `_reasoning()` call injects the current plan state as a hint
3. LLM sees the hint and decides: call a plan tool or a domain tool or produce text
4. `_acting()` executes whatever the LLM chose — plan tools and domain tools are handled identically
5. Loop continues until LLM emits no tool calls

**The single code-enforced constraint**: `PlanNotebook.update_subtask_state()` (`_plan_notebook.py:492-513`) rejects marking a subtask `in_progress` if any previous subtask is not `done` or `abandoned`. This is **the only place in the entire codebase where plan execution order is enforced by code rather than prompt**.

### 3.3 Relationship with ReActAgent

The official tutorial (`docs/tutorial/en/src/task_plan.py:22-34`) states:

> "the plan module works by providing tool functions for plan management and inserting hint messages to guide the ReAct agent to complete the plan"

Architecturally:

```
ReActAgent.reply()
  └── for _ in range(max_iters):                       # ReAct loop unchanged
        └── _reasoning()
              ├── inject hint_msg if plan_notebook exists
              ├── LLM call (sees hint + plan tools in tool schema)
              └── LLM decides what tool to call (or none)
        └── _acting()
              └── execute tool calls (plan tools handled same as domain tools)
```

There is no "planning phase" in this flow. There is no typed handoff between a planner and an executor. The `Plan` artifact exists in memory but is invisible to the framework's control flow.

### 3.4 Verdict: ④a or ④b?

**④a, but with a credible ④b sidecar that is unique among frameworks studied.**

The test for ④b: does the framework generate a Plan artifact first, then execute each step deterministically? No. `PlanNotebook.create_plan()` is a tool the LLM may or may not call. There is no framework-enforced separation of planning and execution phases. The loop structure is identical whether `plan_notebook` is provided or not.

But the sidecar is sophisticated in ways that change the picture:
- **Typed `Plan`/`SubTask` Pydantic models** — genuine data artifacts with state machines
- **Persistent storage with history** (`PlanStorageBase`, `InMemoryPlanStorage`)
- **Sequential-order enforcement at the tool level** (`update_subtask_state` refuses out-of-order transitions)
- **State-adaptive hint generation** (different prompts for "no plan" / "subtask in-progress" / "all done")
- **UI change hooks** (`register_plan_change_hook`)

**Among all ④a frameworks studied, AgentScope is the one that has built the most complete ④b-like infrastructure and chosen not to make it mandatory.** This is a qualitatively different position than Mastra/Google ADK where plan is absent entirely.

---

## 4. The pipeline/ Module

`src/agentscope/pipeline/` exports:

| Export | Description |
|--------|-------------|
| `sequential_pipeline(agents, msg)` | Chain agents: output of agent[n] becomes input to agent[n+1] |
| `fanout_pipeline(agents, msg, enable_gather)` | Same input to all agents; parallel or sequential |
| `SequentialPipeline` | Reusable class wrapper |
| `FanoutPipeline` | Reusable class wrapper |
| `MsgHub` | `async with` context: auto-broadcasts replies to all participants |
| `stream_printing_messages` | Captures streaming output via asyncio Queue |

`sequential_pipeline` (`_functional.py:10-44`) is 3 lines: `for agent in agents: msg = await agent(msg); return msg`. No type validation, no DAG, no branching.

`MsgHub` (`_msghub.py:14-157`) is the most architecturally distinctive primitive: it installs subscriber hooks so any `reply()` call auto-calls `observe()` on all other participants. Enables debate/simulation patterns without manual routing code.

**Relationship to plan/**: These are orthogonal. `pipeline/` handles multi-agent composition. `plan/` handles intra-agent task decomposition. Neither imports the other.

**Comparison with Mastra Workflow**: Mastra's Workflow is a typed DAG with per-step `inputSchema`/`outputSchema` and suspend/resume. AgentScope's pipeline is a thin async utility. The `MsgHub` pattern has no Mastra equivalent.

---

## 5. Multi-Agent Composition and Handoff

Three built-in patterns + one emergent:

**Pattern 1: Pipeline** — `sequential_pipeline` or `fanout_pipeline`. Linear or broadcast routing with no typed handoff.

**Pattern 2: MsgHub** — `async with MsgHub(participants=[a1, a2, a3])`: every reply is auto-broadcast. Used for debates, simulations, round-table discussions.

**Pattern 3: A2AAgent** — dedicated `src/agentscope/agent/_a2a_agent.py` and `src/agentscope/a2a/` module. Agent-to-Agent protocol, suggesting standardized cross-service agent communication. This is explicitly more than message passing.

**Emergent Pattern 4 (Meta Planner)** — `examples/agent/meta_planner_agent/main.py`: a `ReActAgent` with `PlanNotebook` that also has a `create_worker` tool. The planner decomposes tasks using plan tools, then delegates execution to dynamically created worker agents. Orchestration is LLM-driven.

---

## 6. Typing and Validation

Pydantic is used throughout:
- `SubTask`, `Plan` — `BaseModel` (`_plan_model.py:11, 104`)
- `ReActAgent.CompressionConfig` — `BaseModel`
- Structured agent output: `structured_model: Type[BaseModel]` param to `reply()` (`_react_agent.py:379`)
- Tool arguments: derived from Python function signatures

**Flow-level typing**: None in `pipeline/`. `sequential_pipeline` passes `Msg | list[Msg] | None` between agents with no schema validation at handoffs. This is weaker than Mastra's per-step Zod schemas.

**Comparison with Mastra**: Both use runtime schema validation at agent and tool boundaries. Neither has compile-time flow-level type guarantees. AgentScope's Pydantic usage is broader (plan data models, compression config), but pipeline boundaries are untyped. Equivalent profile at pipeline level, stronger at plan level.

---

## 7. Representative Examples

### 7.1 Hello AgentScope — Pure ReAct

`examples/agent/react_agent/main.py`:

```python
agent = ReActAgent(
    name="Friday",
    sys_prompt="You are a helpful assistant named Friday.",
    model=DashScopeChatModel(model_name="qwen-max", ...),
    formatter=DashScopeChatFormatter(),
    toolkit=toolkit,   # execute_shell_command, execute_python_code, view_text_file
    memory=InMemoryMemory(),
)
```

### 7.2 Plan-Related Example (Agent Self-Manages)

`examples/functionality/plan/main_agent_managed_plan.py`:

```python
agent = ReActAgent(
    name="Friday",
    sys_prompt="Your target is to finish the given task with careful planning.",
    ...
    enable_meta_tool=True,     # plan tools in activation group
    plan_notebook=PlanNotebook(),
)
```

### 7.3 Plan-Related Example (Programmer Pre-Creates Plan)

`examples/functionality/plan/main_manual_plan.py` — closest to ④b-style usage:

```python
plan_notebook = PlanNotebook()
await plan_notebook.create_plan(
    name="Comprehensive Report on AgentScope",
    expected_outcome="A markdown format report...",
    subtasks=[SubTask(name="Clone the repository", ...), ...],
)
agent = ReActAgent(..., plan_notebook=plan_notebook)
msg = Msg("user", "Now start to finish the task by the given plan", "user")
```

The programmer creates the `Plan` artifact; the agent executes it via ReAct guided by hints. The execution is still LLM-driven per iteration — but this is the one usage pattern where a human can pre-commit to a plan and hand it to the agent.

---

## 8. Direct Comparison with Mastra

| Dimension | AgentScope | Mastra |
|-----------|-----------|--------|
| **Core execution** | ReAct loop, `max_iters=10` default | ReAct loop, `.dowhile`, `stepCountIs(5)` default |
| **Plan concept** | `PlanNotebook` — sidecar (tools + hints) | None |
| **Structured task decomp.** | `Plan`/`SubTask` Pydantic models | None |
| **Plan persistence** | `PlanStorageBase` (historical plans, recovery) | None |
| **Plan execution order enforcement** | Yes, at `update_subtask_state` tool level | N/A |
| **Outer composition** | Thin pipeline utility (seq, fanout, MsgHub) | Typed DAG Workflow |
| **Step-level typing** | None at pipeline boundaries | Zod schemas per step |
| **Multi-agent broadcast** | `MsgHub` (auto-broadcast via `observe()`) | Not built-in |
| **A2A protocol** | Dedicated `A2AAgent` + `a2a/` module | Tool-based sub-agent calls |
| **Memory** | Short + long-term + auto-compression | Observational only, opt-in |
| **Language** | Python, async-first | TypeScript |

**Core architectural difference**: Mastra's outer composition is a typed DAG (Workflow); agents are steps in it. AgentScope's outer composition is thin utility functions; the `PlanNotebook` moves the "what comes next" logic inside the agent as LLM-controlled hint following. Neither has a framework-enforced Plan-then-Execute phase — but AgentScope has the **data model for it**.

---

## 9. Position on the Agent Architecture Spectrum

```
④a (ReAct)                     Hybrid                    ④b (P&E)
     |                             |                          |
[Mastra]  [Google ADK]        [AgentScope]              [Pure P&E]
     |                             |                          |
ReAct loop only           ReAct loop +               Planner phase →
No plan artifact          PlanNotebook                Plan artifact →
                          (typed Plan/SubTask,        Executor phase
                          persistent storage,         deterministic
                          sequential enforcement
                          at tool level,
                          state-adaptive hints)
                          Thin pipeline utilities
```

AgentScope sits **at the rightmost edge of ④a** — furthest from pure ReAct among frameworks studied. Mastra and Google ADK are clean ④a. AgentScope is ④a with ④b primitives that nearly cross the line.

---

## Key Architectural Insights

1. **Plan as Stateful Tool Group**: `plan/` gives the LLM a structured vocabulary for task decomposition and injects context-appropriate hints. Control flow of the agent loop is unchanged. This is prompt engineering + structured data packaged as a reusable module — valuable and more principled than anything in Mastra, but not an execution model change.

2. **Hint Injection Pattern**: `DefaultPlanToHint` (`_plan_notebook.py:16-169`) generates different hint text for each plan state. Added to working memory before LLM call, deleted after (`_react_agent.py:546-566`). A state machine implemented in prompt space.

3. **Single Code-Enforced Plan Constraint**: `PlanNotebook.update_subtask_state()` (`_plan_notebook.py:492-513`) rejects out-of-order transitions. This is the only place in the codebase where plan execution order is enforced by code rather than by prompt — a minimal but meaningful architectural choice.

4. **Two Plan Activation Modes**: `enable_meta_tool=True` groups plan tools into an optional activation group. `enable_meta_tool=False` (default) makes them always active. The former supports the `meta_planner_agent` orchestrator pattern.

5. **Richer Memory Architecture than Mastra**: Short-term (`InMemoryMemory`, `AsyncSQLAlchemyMemory`) + long-term (`Mem0LongTermMemory`, `ReMePersonalLongTermMemory`, `ReMeTaskLongTermMemory`, `ReMeToolLongTermMemory`) + built-in compression (`CompressionConfig` with `SummarySchema`).

6. **A2AAgent as Unexpected Infrastructure**: Dedicated `A2AAgent` class and `a2a/` module suggests enterprise multi-agent deployment targets. Not found in Mastra or Google ADK.

7. **DashScope-First Origin**: All examples default to `DashScopeChatModel` (Alibaba's API). The model abstraction is generic, but the design priorities reflect Alibaba's production needs: long-running tasks (memory compression), multi-agent enterprise systems (A2A), and complex task decomposition (PlanNotebook).

---

## Conclusion

AgentScope is a **④a (ReAct school) framework with the most sophisticated ④b-like primitives of any framework studied**. The `plan/` module does not change the fundamental classification, but it does place AgentScope at the rightmost edge of ④a.

- **Plan as data model** (`Plan`, `SubTask`): genuine ④b-like artifacts, typed and persistent — **the only framework studied that has these at the source-code level**
- **Plan as runtime mechanism** (`PlanNotebook`): sophisticated, but adds prompt guidance to an unchanged ReAct loop — not a separate execution phase
- **Execution architecture**: identical to standard ReAct regardless of whether `PlanNotebook` is used

**For the blog**: the ④b slot is not completely empty — AgentScope provides the strongest case that ④b-style primitives can coexist with ④a execution. A refined blog position: "④a with ④b primitives — the Plan artifact exists, the P&E execution contract does not." This is a more nuanced conclusion than "④b is empty" but sharpens rather than weakens the central argument: **even the framework that builds a full Plan data model chose not to enforce P&E at execution time**. The gap between ④a and ⑤ is real.

**One-sentence placement**: AgentScope is a Python, async-first ReAct framework (④a) with a `PlanNotebook` sidecar providing ④b-style Plan/SubTask data models and LLM-guided execution hints, but without a framework-enforced plan-then-execute phase — sitting at the rightmost edge of ④a, closest to but not crossing into ④b.
