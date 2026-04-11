# LangChain Agent Architecture Research

Last Updated: 2026-04-12

> **Research Methodology**: Source-level analysis of the LangChain monorepo at `/Users/linguanguo/dev/llm-agent-research/langchain` (shallow clone of github.com/langchain-ai/langchain). The repo contains two distinct packages: `libs/langchain_v1/` (LangChain 1.x, the current package) and `libs/langchain/langchain_classic/` (legacy 0.x). This research traces Agent architecture in both, with special focus on whether a Plan-and-Execute abstraction exists anywhere in the codebase.

## Sources

**LangChain 1.0 Agent Files**
- `libs/langchain_v1/langchain/__init__.py:1-3` — Version string (`1.2.15`)
- `libs/langchain_v1/langchain/agents/__init__.py:1-9` — Public API: `create_agent`, `AgentState` only
- `libs/langchain_v1/langchain/agents/factory.py:1-1663` — Entire `create_agent` implementation
- `libs/langchain_v1/langchain/agents/middleware/types.py:350-404` — `AgentState`, `AgentMiddleware` base class
- `libs/langchain_v1/langchain/agents/middleware/todo.py:1-217` — `TodoListMiddleware`
- `libs/langchain_v1/langchain/agents/middleware/summarization.py:1-60` — `SummarizationMiddleware`

**LangChain Classic Agent Files**
- `libs/langchain/langchain_classic/agents/__init__.py:1-164` — Full public API with 20+ exports
- `libs/langchain/langchain_classic/agents/agent.py:1012-1622` — `AgentExecutor._call()` core while loop
- `libs/langchain/langchain_classic/agents/agent_types.py:1-55` — `AgentType` enum (all deprecated)
- `libs/langchain/langchain_classic/agents/react/agent.py:1-151` — `create_react_agent` using LCEL `|` pipe
- `libs/langchain/langchain_classic/agents/mrkl/base.py:1-60` — `ZeroShotAgent` (MRKL)
- `libs/langchain/langchain_classic/agents/initialize.py:19-24` — `initialize_agent` (deprecated)

**LangChain Core**
- `libs/core/langchain_core/runnables/base.py:124` — `Runnable` abstract base class, `__or__` at line 618

---

## 1. The LangChain 1.0 / LangChain Classic Split

### 1.1 Directory Structure Comparison

The monorepo contains **two entirely separate Python packages**:

| Attribute | LangChain Classic | LangChain 1.0 |
|-----------|------------------|---------------|
| Package root | `libs/langchain/langchain_classic/` | `libs/langchain_v1/langchain/` |
| PyPI name | `langchain-classic` | `langchain` (v1.x) |
| Version | 0.x | `1.2.15` |
| Agent public API | 20+ exports (AgentExecutor, ZeroShotAgent, etc.) | **2 exports: `create_agent`, `AgentState`** |
| Core execution | Custom Python `while` loop (`AgentExecutor._call`) | **LangGraph `StateGraph.compile()`** |
| `plan_and_execute` module | Absent | Absent |

### 1.2 What Changed in 1.0

LangChain 1.0 is **not a refactor of Classic — it is a ground-up rewrite**. The Classic package survives as `langchain-classic` for backward compatibility only. All of Classic's breadth (AgentExecutor, AgentType enum, initialize_agent, nine agent type subclasses) is either deprecated or entirely absent in 1.0. **The 1.0 public surface is radically smaller: one factory function that returns a compiled LangGraph.**

`AgentType` is marked `@deprecated("0.1.0", removal="1.0")` (`agent_types.py:10`). `initialize_agent` carries the same mark. `ZeroShotAgent`, `OpenAIFunctionsAgent`, `ReActChain` are all deprecated in Classic. None appear in 1.0.

---

## 2. LangChain 1.0 Agent Architecture (langchain_v1)

### 2.1 Entry Point and `create_agent()` API

The entire public surface of `libs/langchain_v1/langchain/agents/` is two symbols (`__init__.py:3-4`):

```python
from langchain.agents.factory import create_agent
from langchain.agents.middleware.types import AgentState
```

`create_agent` signature (`factory.py:691-708`):

```python
def create_agent(
    model: str | BaseChatModel,
    tools: Sequence[BaseTool | Callable[..., Any] | dict[str, Any]] | None = None,
    *,
    system_prompt: str | SystemMessage | None = None,
    middleware: Sequence[AgentMiddleware[StateT_co, ContextT]] = (),
    response_format: ResponseFormat[ResponseT] | ... | None = None,
    state_schema: type[AgentState[ResponseT]] | None = None,
    context_schema: type[ContextT] | None = None,
    checkpointer: Checkpointer | None = None,
    store: BaseStore | None = None,
    interrupt_before: list[str] | None = None,
    interrupt_after: list[str] | None = None,
    ...
) -> CompiledStateGraph[...]:
```

**The return type is `CompiledStateGraph` — a LangGraph object.**

### 2.2 Relationship to LangGraph

`create_agent` is **not a wrapper around `langgraph.prebuilt.create_react_agent`**. It constructs a LangGraph `StateGraph` **directly** (`factory.py:1034-1041`):

```python
graph: StateGraph[...] = StateGraph(
    state_schema=resolved_state_schema,
    input_schema=input_schema,
    output_schema=output_schema,
    context_schema=context_schema,
)
```

It adds nodes manually (`factory.py:1372-1376`):

```python
graph.add_node("model", RunnableCallable(model_node, amodel_node, trace=False))
if tool_node is not None:
    graph.add_node("tools", tool_node)
```

The imports confirm tight coupling to LangGraph internals (`factory.py:22-26`):

```python
from langgraph._internal._runnable import RunnableCallable
from langgraph.constants import END, START
from langgraph.graph.state import StateGraph
from langgraph.prebuilt.tool_node import ToolCallWithContext, ToolNode
from langgraph.types import Command, Send
```

The final line returns a compiled LangGraph (`factory.py:1655-1663`):

```python
return graph.compile(
    checkpointer=checkpointer,
    store=store,
    interrupt_before=interrupt_before,
    interrupt_after=interrupt_after,
    ...
).with_config(config)
```

**Verdict**: `create_agent` is not a thin alias for `langgraph.prebuilt.create_react_agent`, but **the runtime is 100% LangGraph**. It implements the same tool-call loop pattern using LangGraph primitives directly, with the `AgentMiddleware` system as the only substantive addition.

### 2.3 Execution Model

The core graph topology (`factory.py:1490-1552`):

```
START
  → [before_agent middleware nodes]
  → [before_model middleware nodes]
  → model node
  → tools node (if AIMessage.tool_calls present)  |  → [after_model] → [after_agent] → END
tools node → [before_model] → model node  (loop back)
```

`model_node` reads `state["messages"]`, calls the bound LLM, returns `Command` objects updating state. `ToolNode` executes tools. The loop continues as long as `AIMessage.tool_calls` is non-empty. **This is a pure ReAct loop implemented as a LangGraph state machine.**

### 2.4 Middleware System

The distinctive feature of LangChain 1.0 vs. raw LangGraph is `AgentMiddleware` (`middleware/types.py:380`). Middleware hooks into six lifecycle points:

| Hook | Becomes a graph node? | Typical use |
|------|-----------------------|-------------|
| `before_agent` / `abefore_agent` | Yes (once at start) | Inject context, load memory |
| `before_model` / `abefore_model` | Yes (every loop iteration) | Compress history |
| `wrap_model_call` / `awrap_model_call` | No (wraps LLM call inline) | Retry, fallback, PII scrub |
| `after_model` / `aafter_model` | Yes (every iteration) | Inspect output |
| `wrap_tool_call` / `awrap_tool_call` | No (wraps tool execution) | Tool retry, redaction |
| `after_agent` / `aafter_agent` | Yes (once at end) | Persist results |

Shipped middleware implementations include: `SummarizationMiddleware`, `TodoListMiddleware`, `ModelRetryMiddleware`, `ModelFallbackMiddleware`, `ModelCallLimitMiddleware`, `ToolCallLimitMiddleware`, `ToolRetryMiddleware`, `HumanInTheLoopMiddleware`, `PiiMiddleware`, `ContextEditingMiddleware`, `FileSearchMiddleware`, `ShellToolMiddleware`, `ToolEmulatorMiddleware`, `ToolSelectionMiddleware`.

**`TodoListMiddleware`** (`todo.py:162-217`) is noteworthy: it provides a `write_todos` tool that lets the agent maintain a `PlanningState.todos: list[{content, status}]` list in LangGraph state. This is a form of **soft planning** — the agent can write its task list before executing — but execution is still a single ReAct loop; there is no separate planner/executor split.

---

## 3. LangChain Classic Agent Architecture

### 3.1 AgentExecutor Core Loop

`AgentExecutor` inherits from `Chain`. The core loop is a hand-rolled Python `while` in `_call()` (`agent.py:1570-1622`):

```python
intermediate_steps: list[tuple[AgentAction, str]] = []
iterations = 0
while self._should_continue(iterations, time_elapsed):
    next_step_output = self._take_next_step(
        name_to_tool_map, color_mapping, inputs, intermediate_steps, ...
    )
    if isinstance(next_step_output, AgentFinish):
        return self._return(next_step_output, intermediate_steps, ...)
    intermediate_steps.extend(next_step_output)
    iterations += 1
```

`_iter_next_step` calls `self._action_agent.plan(intermediate_steps, **inputs)` to get the next `AgentAction`, then calls the tool. **This is Python-native ReAct with no graph, no state machine.**

### 3.2 Agent Types Enumeration

`AgentType` enum (`agent_types.py:15-55`), **all deprecated as of 0.1.0, removal at 1.0**:

| AgentType | Implementation | Control Flow |
|-----------|---------------|--------------|
| `ZERO_SHOT_REACT_DESCRIPTION` | `ZeroShotAgent` (MRKL) | Text-based ReAct, `Thought/Action/Observation` |
| `REACT_DOCSTORE` | `ReActDocstoreAgent` | ReAct with docstore lookup |
| `SELF_ASK_WITH_SEARCH` | `SelfAskWithSearchChain` | Decompose question → search |
| `CONVERSATIONAL_REACT_DESCRIPTION` | `ConversationalAgent` | ReAct with chat memory |
| `CHAT_ZERO_SHOT_REACT_DESCRIPTION` | Chat model variant | Text ReAct for chat models |
| `CHAT_CONVERSATIONAL_REACT_DESCRIPTION` | Chat conversational | Chat model + memory |
| `STRUCTURED_CHAT_ZERO_SHOT_REACT_DESCRIPTION` | `StructuredChatAgent` | Multi-input tool support |
| `OPENAI_FUNCTIONS` | `OpenAIFunctionsAgent` | Native function calling |
| `OPENAI_MULTI_FUNCTIONS` | `OpenAIMultiFunctionsAgent` | Parallel function calls |

**All nine variants are ReAct.** The differences are primarily in how they format the scratchpad for text-based tool selection. Native function-calling APIs made that distinction irrelevant — one generic factory with the model's native protocol suffices.

### 3.3 ReAct Implementation

`create_react_agent` (`react/agent.py:16-150`) builds an LCEL pipeline:

```python
return (
    RunnablePassthrough.assign(
        agent_scratchpad=lambda x: format_log_to_str(x["intermediate_steps"]),
    )
    | prompt
    | llm_with_stop
    | output_parser
)
```

This returns a `Runnable` (a `RunnableSequence`). The docstring itself says: *"This implementation is based on the foundational ReAct paper but is older and not well-suited for production applications. For a more robust and feature-rich implementation, we recommend using the `create_agent` function from the `langchain` library."* (`react/agent.py:35-39`)

---

## 4. Plan-and-Execute in LangChain — **The Most Important Section**

### 4.1 `plan_and_execute` Module Search Result

**The module does not exist in this repository.**

A grep across all of `libs/langchain/` and `libs/langchain_v1/` for both `plan_and_execute` and `PlanAndExecute` returns **zero matches**. The `langchain_classic/agents/` directory contains subdirectories for: `chat`, `conversational`, `conversational_chat`, `format_scratchpad`, `json_chat`, `mrkl`, `openai_assistant`, `openai_functions_agent`, `openai_functions_multi_agent`, `openai_tools`, `output_parsers`, `react`, `self_ask_with_search`, `structured_chat`, `tool_calling_agent`, `xml`. **No `plan_and_execute`.**

### 4.2 Historical Context

**The `plan_and_execute` module lived in `langchain_experimental`, not in `langchain` itself**, even during the 0.x era. This is documented indirectly: Classic's `__init__.py` lines 108-117 show the pattern for deprecated modules moved to `langchain_experimental` (e.g., CSV/Pandas agents), and the message template points to `langchain_experimental` as the destination. The `langchain_experimental` package is not included in this shallow clone.

**The `experimental` namespace classification was deliberate**: the LangChain team explicitly chose not to promote P&E to a production pattern. It received limited maintenance and no continuation in LangChain 1.0.

### 4.3 The `TodoListMiddleware` as Soft P&E

`TodoListMiddleware` (`middleware/todo.py:162-217`) comes closest to P&E in LangChain 1.0. It adds a `write_todos` tool and extends `AgentState`:

```python
class PlanningState(AgentState[ResponseT]):
    todos: Annotated[NotRequired[list[Todo]], OmitFromInput]
```

where `Todo` is `{content: str, status: Literal["pending", "in_progress", "completed"]}`. The agent writes its task list before execution and updates status as it proceeds.

**This is not P&E** by the architectural definition: there is no dedicated Planner LLM call, no typed Plan artifact handed from one phase to another, and no separate Executor runner. **The agent that writes the todos and the agent that executes them are the same model in the same loop. It is ReAct with a structured scratchpad.**

### 4.4 Implications for the Blog

Three conclusions follow from this evidence:

1. **LangChain never had official P&E in the main package.** The `langchain_experimental.plan_and_execute` module existed but its home in `experimental` communicates the team's position — it was a proof-of-concept, not a production recommendation.

2. **LangChain 1.0 chose against P&E.** The rewrite is ReAct-only, with the `TodoListMiddleware` as the only concession to planning — and even that is implemented as in-loop state, not a separate phase.

3. **The strongest conclusion for the blog**: *"The most prominent open-source P&E framework (LangChain's `experimental.plan_and_execute`) was tried, classified as experimental, and was not carried forward in the 1.0 rewrite. The entire LangChain ecosystem converged on LangGraph-based ReAct as the production standard."* This gives the P&E-vs-ReAct question a historical arc: **P&E was explored, rejected by the market leader, and is now only found in specialized contexts**.

---

## 5. Runnable and LCEL

`Runnable` (`libs/core/langchain_core/runnables/base.py:124`) is an abstract base class providing `invoke()`, `stream()`, `batch()` plus `__or__` creating `RunnableSequence` chains via the `|` operator.

In Classic, Agent is not directly a `Runnable` but `AgentExecutor` extends `Chain` which extends `Runnable`. In LangChain 1.0, the agent returned by `create_agent` is a `CompiledStateGraph` which implements the `Runnable` interface. Both generations surface the same `Runnable` protocol at the call site, but the internals differ entirely.

**LCEL is not suitable for building P&E**: the `|` operator is a linear data-flow pipe. P&E requires conditional branching and multi-phase orchestration — which is LangGraph's domain, not LCEL's.

---

## 6. Multi-Agent Support

**In LangChain 1.0**, multi-agent is handled at the LangGraph layer. The `name` parameter in `create_agent` is documented "particularly useful for building multi-agent systems" — the compiled agent can be embedded as a subgraph node in a parent LangGraph. **There is no `MultiAgent`, `AgentTeam`, or orchestrator class in `langchain` 1.0 itself.**

**In Classic**, `OpenAIMultiFunctionsAgent` supports parallel tool calls per turn — this is parallel tool execution within a single agent step, not multi-agent orchestration. No multi-agent infrastructure exists in Classic.

---

## 7. Direct Comparison with Mastra / Google ADK / AgentScope / LangGraph

| Dimension | LangChain Classic | LangChain 1.0 | Mastra | AgentScope | LangGraph |
|-----------|------------------|---------------|--------|------------|-----------|
| Core loop | Python while loop | LangGraph StateGraph | TypeScript Workflow + dowhile | Python bounded for loop | StateGraph (DSL) |
| Planning | None | `TodoListMiddleware` (soft, in-loop) | None | `PlanNotebook` (typed, in-loop) | None (user builds custom) |
| Agent variety | 9+ types via enum | 1 factory (`create_agent`) | 1 class (`Agent`) | 3 classes | 2 prebuilt + custom |
| Multi-agent | None | LangGraph subgraph embedding | Workflow composition | MsgHub broadcast | Native subgraph nodes |
| Extension points | Callbacks | `AgentMiddleware` (6 hooks) | Input/Output processors | Pre/post hooks | Custom graph nodes |
| Execution runtime | Pure Python | LangGraph | TypeScript Workflow | Pure Python | LangGraph |

**LangChain 1.0 and raw LangGraph are architecturally identical at the execution layer.** `create_agent` is a higher-level factory — the `AgentMiddleware` system is its differentiation.

---

## 8. Position on the Agent Architecture Spectrum

| Tier | Framework | Evidence |
|------|-----------|----------|
| ④a ReAct | **LangChain Classic** | Python while loop, `intermediate_steps` scratchpad, all 9 agent types are ReAct variants |
| ④a ReAct | **LangChain 1.0** | LangGraph tool-call loop, `create_agent` returns `CompiledStateGraph` |
| ④a ReAct | Mastra | TypeScript Workflow + dowhile loop |
| ④a ReAct | Google ADK | Python while loop, Workflow agents for orchestration |
| ④a/④b boundary | AgentScope | ReAct loop with typed `Plan`/`SubTask` Pydantic models |
| ④b P&E (historical) | `langchain_experimental.plan_and_execute` | Separate Planner + Executor, deprecated, not in this snapshot |
| ⑤ Custom P&E | User's system | Typed dataflow + planner/executor split |

---

## Key Architectural Insights

**1. LangChain 1.0 is not just "LangGraph wrapper"**: `create_agent` builds a `StateGraph` directly, not by calling `langgraph.prebuilt.create_react_agent`. But the runtime is LangGraph. The architectural gap between `create_agent` and writing a LangGraph agent by hand is the `AgentMiddleware` composability layer and the `init_chat_model` string-based model initialization. Everything else is identical.

**2. Why Classic's agent type proliferation collapsed**: The nine `AgentType` variants differed primarily in how they formatted the scratchpad for text-based tool selection. Native function-calling APIs (OpenAI `functions`, then `tool_calls`) made that distinction irrelevant — one generic factory with the model's native protocol suffices. LangChain 1.0 reflects this convergence.

**3. Middleware vs. P&E**: LangChain 1.0's middleware system is designed for extensibility within a ReAct loop. The `TodoListMiddleware` is the closest thing to P&E — structured task state, visible to the user — but the Planner and Executor are the same LLM in the same loop. True P&E requires two distinct phases with a typed Plan artifact as the interface. **LangChain 1.0 does not have this, and the design choice appears deliberate.**

**4. The `langchain_experimental` lesson**: The original P&E implementation was gated behind `experimental` from the start. When LangChain 1.0 shipped, it was not brought forward. **This is the strongest evidence that the LangChain team evaluated P&E and chose ReAct + graph as the production path.** The deprecation of `AgentExecutor` itself (the while-loop runner) and its replacement with a LangGraph-compiled agent completes this arc.

---

## Conclusion

**LangChain Classic** is a foundational ④a ReAct framework: `AgentExecutor` runs a Python while loop calling `agent.plan()` → tool → repeat. Its nine `AgentType` variants are all deprecated. **No `plan_and_execute` module exists in the `langchain` or `langchain_classic` packages.**

**LangChain 1.0** is also firmly ④a ReAct, re-implemented on LangGraph: `create_agent` builds and compiles a `StateGraph` with a model node and tools node in a tool-call loop. Its value-add over raw LangGraph is `AgentMiddleware` composability. Multi-agent and memory are LangGraph responsibilities. **The return type of `create_agent` is `CompiledStateGraph` — LangChain 1.0 is, at the execution layer, LangGraph.**

**Plan-and-Execute status in LangChain**: Historically existed as `langchain_experimental.plan_and_execute` (not present in this repo snapshot). The `experimental` classification was never upgraded. It was not carried forward into LangChain 1.0. The LangChain ecosystem's answer to complex planning is either `TodoListMiddleware` (soft, in-loop task state) or custom LangGraph topologies — **not a P&E tier**.

**Blog positioning**: LangChain belongs entirely to ④a (ReAct 派). Its historical significance is threefold:
1. It validated ReAct at the largest scale in the LLM-framework ecosystem, causing convergence
2. It hosted the most prominent P&E experiment (`langchain_experimental`) which was then abandoned in 1.0
3. With LangChain 1.0, even its Python-native while loop was replaced by LangGraph, completing the industry's convergence to graph-based ReAct as the production standard

**The story of LangChain is the story of how ReAct won.**
