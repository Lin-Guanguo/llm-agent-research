# LangGraph Agent Architecture Research

Last Updated: 2026-04-12

> **Research Methodology**: Source-level analysis of LangGraph monorepo shallow-cloned at `/Users/linguanguo/dev/llm-agent-research/langgraph` (github.com/langchain-ai/langgraph). Primary focus: Pregel execution engine, `create_react_agent` graph construction, and the existence/absence of Plan-and-Execute patterns. Note: in LangGraph 1.0, `create_react_agent` is deprecated in favor of `langchain.agents.create_agent`; the implementation analyzed here is the in-repo transition version.

## Sources

**Pregel Execution Engine**
- `libs/langgraph/langgraph/pregel/main.py:343-410` — `Pregel` class docstring: BSP model, actors, channels
- `libs/langgraph/langgraph/pregel/main.py:2751-2776` — Main BSP loop
- `libs/langgraph/langgraph/pregel/_loop.py:148-583` — `PregelLoop`: `tick()`, `after_tick()`
- `libs/langgraph/langgraph/pregel/_algo.py:218-491` — `apply_writes()`, `prepare_next_tasks()`
- `libs/langgraph/langgraph/pregel/_runner.py:1-80` — `PregelRunner` parallel task dispatch

**StateGraph API**
- `libs/langgraph/langgraph/graph/state.py:115-890` — `StateGraph[StateT, ContextT, InputT, OutputT]` generic class
- `libs/langgraph/langgraph/types.py:574-794` — `Send`, `Command`, `interrupt()`

**Prebuilt create_react_agent**
- `libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py:278-980` — Full `create_react_agent` implementation
- `libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py:831-859` — `should_continue` conditional edge

**Plan-and-Execute Evidence**
- `examples/plan-and-execute/plan-and-execute.ipynb` — Redirect stub only (historical content moved out)
- `docs/redirects.json:190` — P&E tutorial now redirects to `langchain/middleware/built-in#to-do-list`
- `docs/redirects.json:290` — ReWOO tutorial redirects to `langgraph/overview`
- `examples/rewoo/rewoo.ipynb`, `examples/llm-compiler/LLMCompiler.ipynb` — Both redirect stubs

---

## 1. Pregel — The Underlying Execution Model

### What It Is

LangGraph's runtime is a class called `Pregel` (`libs/langgraph/langgraph/pregel/main.py:343`). The name and model are directly borrowed from Google's Pregel graph computation paper. The docstring at `main.py:347-371` states explicitly:

> Pregel organizes the execution of the application into multiple steps, following the **Pregel Algorithm / Bulk Synchronous Parallel** model.
>
> Each step consists of three phases:
> - **Plan**: Determine which actors to execute in this step.
> - **Execution**: Execute all selected actors in parallel, until all complete, or one fails, or a timeout is reached. During this phase, **channel updates are invisible to actors until the next step**.
> - **Update**: Update the channels with the values written by the actors in this step.

This is BSP (Bulk Synchronous Parallel). Key property: **channel writes from step N are only visible in step N+1**. Within a single step, all eligible actors run in parallel and cannot observe each other's writes.

### The Core BSP Loop

`main.py:2751-2776` contains the literal comment "Similarly to Bulk Synchronous Parallel / Pregel model":

```python
while loop.tick():
    for task in loop.match_cached_writes():
        loop.output_writes(task.id, task.writes, cached=True)
    for _ in runner.tick(
        [t for t in loop.tasks.values() if not t.writes],
        timeout=self.step_timeout,
        get_waiter=get_waiter,
        schedule_task=loop.accept_push,
    ):
        yield from _output(...)
    loop.after_tick()
```

`loop.tick()` (`_loop.py:506-583`):
1. Check recursion limit
2. Call `prepare_next_tasks()` — determine which nodes fire this step
3. Check `interrupt_before`
4. Return `True` if tasks exist, `False` (done) if none fired

`loop.after_tick()` (`_loop.py:585-618`):
1. Call `apply_writes()` — commit all task writes to channels, bump versions
2. Save checkpoint
3. Check `interrupt_after`

### Node Triggering: PULL and PUSH

`prepare_next_tasks()` (`_algo.py:400-491`) identifies two task types:

**PULL tasks**: edge-triggered. A node fires when channels it subscribes to have been updated since it last processed them. Version comparison: `checkpoint["channel_versions"][chan] > seen.get(chan, null_version)`. Standard graph edge model.

**PUSH tasks**: Send-triggered. A node fires because another node returned a `Send(node, state)` packet into the `TASKS` topic channel. These are consumed at the start of each step. PUSH enables dynamic fan-out: a single routing decision can spawn N parallel executions.

### Parallelism

All tasks (both PULL and PUSH) are submitted to a `concurrent.futures.ThreadPoolExecutor`. Nodes in the same superstep run concurrently.

### Comparison with Traditional DAG Engines

| Property | LangGraph Pregel | Airflow / Mastra Workflow |
|---|---|---|
| Execution model | BSP supersteps | DAG topological sort |
| Concurrency | Thread pool per step | DAG-defined branches |
| Cycles | **Native (loop = back-edge)** | Impossible (acyclic) |
| Termination | "No tasks fired" | "No downstream nodes" |
| Dynamic routing | `Send` PUSH mid-step | Conditional branch at compile time |

**The critical difference**: LangGraph **natively supports cycles** because termination is event-driven, not structure-driven. Airflow and Mastra Workflow require an explicit loop construct; in LangGraph, the loop IS the graph topology.

---

## 2. StateGraph API and Type System

`StateGraph` is defined as `class StateGraph(Generic[StateT, ContextT, InputT, OutputT])` at `graph/state.py:115`. Four type parameters:

- `StateT`: shared state schema — TypedDict, Pydantic `BaseModel`, or dataclass
- `ContextT`: immutable runtime context (user_id, db_conn, etc.)
- `InputT`: optional separate input schema
- `OutputT`: optional separate output schema

### State Schema Typing

State keys use `Annotated[type, reducer]` to attach a merge function:

```python
class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]  # reducer accumulates
    plan: list[str]       # plain → LastValue channel (last write wins)
    count: int
```

`_add_schema()` (`state.py:260-290`) translates annotations to channel objects at graph construction:
- Plain key → `LastValue(type)` channel
- `Annotated[type, reducer]` → `BinaryOperatorAggregate(type, reducer)` channel

**Versus Mastra**: Mastra enforces Zod contract validation at each step boundary. LangGraph validates at graph compile time and uses Python type annotations for static checking. LangGraph is more flexible but provides weaker runtime type guarantees.

---

## 3. Conditional Edges and Command Primitives

Two orthogonal routing mechanisms:

### Conditional Edges

`add_conditional_edges(source, path_fn, path_map)` at `state.py:842-890`. After `source` completes, `path_fn` runs and returns a node name or list of node names. Static routing table at graph compile time.

### Send (Dynamic Fan-Out)

`Send(node, arg)` at `types.py:574-647`. Spawns multiple parallel instances with independent state:

```python
# map-reduce pattern
def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]
```

### Command (Imperative State + Routing)

`Command(goto=..., update=..., resume=...)` at `types.py:652-703`:

```python
def my_node(state):
    return Command(
        update={"result": "done"},    # state update
        goto="next_node",             # or list[Send] for fan-out
    )
```

`Command.PARENT` routes to the parent graph, enabling subgraph-to-parent escalation.

### Theoretical P&E Assembly

These primitives can express Plan-and-Execute:

```python
class PEState(TypedDict):
    goal: str
    plan: list[str]
    results: Annotated[list[str], operator.add]

def planner(state):
    plan = llm.invoke(f"Plan for: {state['goal']}")
    return {"plan": parse_plan(plan)}

def executor(state):  # receives individual step via Send
    result = execute_step(state["step"])
    return {"results": [result]}

def route_after_plan(state):
    return [Send("executor", {**state, "step": s}) for s in state["plan"]]

workflow = StateGraph(PEState)
workflow.add_node("planner", planner)
workflow.add_node("executor", executor)
workflow.add_edge(START, "planner")
workflow.add_conditional_edges("planner", route_after_plan)
workflow.add_edge("executor", END)
```

This is valid LangGraph code. **The question is whether LangGraph officially promotes this. Spoiler: it used to, and no longer does.**

---

## 4. Prebuilt create_react_agent (④a Usage)

`create_react_agent` at `libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py:278` builds a `StateGraph` with this topology:

**Nodes**:
- Optional `pre_model_hook` (message trimming, summarization)
- `agent` — `call_model()`: applies prompt, invokes LLM, appends `AIMessage`
- Optional `post_model_hook` (human-in-loop, guardrails)
- `tools` — `ToolNode`: executes tool calls, appends `ToolMessage`s
- Optional `generate_structured_response`

**Graph topology**:
```
START → [pre_model_hook →] agent
agent --[conditional: should_continue]--> tools | END
tools → [pre_model_hook →] agent    ← the loop back-edge
```

The `should_continue` function (`chat_agent_executor.py:831-859`):

```python
def should_continue(state) -> str | list[Send]:
    last_message = messages[-1]
    if not isinstance(last_message, AIMessage) or not last_message.tool_calls:
        return END                     # no tool calls → terminate
    else:
        return [Send("tools", ToolCallWithContext(tool_call=call, state=state))
                for call in last_message.tool_calls]   # fan-out per tool call
```

**This is the ReAct loop expressed entirely as graph topology.** There is no `while` loop, no special ReAct code — the cycle exists because the `tools` node eventually routes back to `agent`. The loop terminates when `agent` produces no tool calls.

**Important**: `create_react_agent` carries a `@deprecated` decorator at `chat_agent_executor.py:274-277`. In LangGraph 1.0 it is replaced by `langchain.agents.create_agent`. LangGraph 1.0 is moving the opinionated agent factory **out of LangGraph into the `langchain` package**, leaving LangGraph as pure graph infrastructure.

---

## 5. Plan-and-Execute in LangGraph (Existence Check) — **The Most Important Section**

### Search Results

Full-repo grep for `plan_and_execute`, `plan-and-execute`, `PlanAndExecute` returns exactly two files:

1. **`examples/plan-and-execute/plan-and-execute.ipynb`** — A stub notebook with a single markdown cell: "This file has been moved." No code. No tutorial content.

2. **`docs/redirects.json:190`** — The old tutorial URL `/tutorials/plan-and-execute/plan-and-execute` now redirects to:
   `https://docs.langchain.com/oss/python/langchain/middleware/built-in#to-do-list`

**The redirect destination is the langchain middleware documentation page**, not a P&E tutorial. In LangGraph 1.0, P&E was reconceived as a `langchain` middleware ("to-do list" pattern), not a LangGraph graph topology.

Similarly redirected (stubs only):
- `examples/rewoo/rewoo.ipynb` — ReWOO (P&E variant with parallel execution)
- `examples/llm-compiler/LLMCompiler.ipynb` — LLM Compiler (parallel P&E with dependency resolution)

### Interpretation

The P&E tutorial was a **first-class LangGraph tutorial in v0.x**. In LangGraph 1.0, it was **deliberately removed from the LangGraph repository and redirected to langchain's middleware system**. This is not abandonment of the concept — it is reclassification of where P&E lives in the stack.

**LangGraph's current official position: Plan-and-Execute is not a LangGraph graph pattern; it is a langchain agent middleware pattern.**

There is **no official P&E graph topology, no P&E prebuilt node, and no P&E documentation** in the current LangGraph codebase.

---

## 6. Multi-Agent Patterns (Supervisor, Swarm)

### Subgraph Composition

The primary multi-agent mechanism is **subgraph composition**: a compiled `CompiledStateGraph` can be used as a node inside another `StateGraph`. The subgraph receives a slice of the parent state, executes its own Pregel loop in a nested namespace, and returns writes that propagate back.

### Handoff Mechanism

There is **no first-class `handoff()` primitive** in `libs/prebuilt/`. The documented supervisor and swarm patterns use standard graph primitives:

- **Supervisor pattern**: Supervisor node calls LLM to decide which worker subgraph to invoke. The "handoff" is a `Send(worker_name, task_state)` returned by the supervisor's routing function.
- **Swarm pattern**: Agents share state channels; writes trigger subscriptions via BSP channel update mechanism.

Unlike ADK's `transfer_to_agent()` (tool call causing agent substitution), LangGraph handoffs are graph topology decisions.

### State Sharing

All subgraphs in the same execution share the parent `StateGraph`'s state by default. This is structurally different from Mastra's `AgentTool` (isolated) or ADK's `transfer_to_agent` (context hand-over).

---

## 7. Checkpointer and Human-in-the-Loop

### interrupt() Mechanism

`interrupt(value)` at `types.py:705-794` is called from within a node. On first invocation: raises `GraphInterrupt`, saves current writes, delivers `value` to the caller. On resume via `Command(resume=value)`: the node re-executes from the start, `interrupt()` returns the provided value.

```python
# Pattern: plan review before execution
def planner(state):
    plan = llm.invoke(f"Plan for: {state['goal']}")
    approved_plan = interrupt(plan)    # pause here, surface plan to human
    return {"plan": approved_plan}    # resume with human-approved plan
```

### P&E Plan Review Applicability

The `interrupt()` primitive technically supports "generate plan → pause for review → execute approved plan" workflows. **But there is no prebuilt `PlannerWithReview` node.** Related helper classes (`HumanInterruptConfig`, `ActionRequest`, `HumanInterrupt`) are deprecated in LangGraph 1.0 and moved to `langchain.agents.interrupt`.

---

## 8. Direct Comparison with Mastra / Google ADK / AgentScope

### "Agent is a Graph" vs "Loop Inside Agent"

| Framework | Loop Location | Architecture |
|---|---|---|
| Mastra | `.dowhile()` method | Loop inside Agent class, Workflow outside |
| Google ADK | `while True: _run_one_step_async` | Loop inside BaseLlmFlow, orchestration outside |
| AgentScope | `for _ in range(max_iters)` | Loop inside Agent, Plan as parameter |
| **LangGraph** | **No loop function — loop IS the `agent → tools` back-edge** | **Graph topology is the loop** |

In Mastra, ADK, and AgentScope, the ReAct loop is an imperative `while`/`for` inside a method. In LangGraph, **the loop is the graph's conditional back-edge. The graph IS the agent.**

### Cycle Support

| Framework | Cycles |
|---|---|
| Mastra Workflow | No (DAG only; loops hidden inside agent) |
| ADK SequentialAgent/ParallelAgent | No (DAG only) |
| **LangGraph Pregel** | **Yes, first-class (BSP terminates on event absence)** |

---

## 9. Position on the Agent Architecture Spectrum

### 9.1 Dominant Usage Pattern (What Most Users Do)

```python
from langchain.agents import create_agent

agent = create_agent(model, tools=[search, calculator])
result = agent.invoke({"messages": [{"role": "user", "content": "..."}]})
```

This is **④a**: a prebuilt ReAct agent backed by a `StateGraph` two-node cycle. The vast majority of LangGraph tutorials and community examples use this pattern.

### 9.2 Theoretical Capability (What the Primitives Allow)

The raw `StateGraph` + `Send` + `Command` + `interrupt()` primitives form a **general-purpose stateful graph execution engine** capable of expressing:
- ReAct loops (④a) — demonstrated by `create_react_agent`
- **Plan-and-Execute (④b) — demonstrated historically by `plan-and-execute.ipynb` (now archived)**
- Map-reduce workflows (parallel P&E) — demonstrated by `rewoo.ipynb` (now archived)
- Hierarchical multi-agent orchestration
- Human-in-the-loop with arbitrary pause/resume
- Linear DAG workflows equivalent to Mastra Workflow

### 9.3 Final Classification

**LangGraph is ④a by dominant usage, but is architecturally a general-purpose graph execution substrate that has deliberately stepped back from being ④b.**

Specific claims:

1. **The prebuilt layer is ④a.** `create_react_agent` / `create_agent` is a ReAct factory. This is what 90%+ of practitioners use.

2. **The primitive layer is meta-level.** `StateGraph`, `Pregel`, `Send`, `Command` can express any agent pattern. They are not opinionated.

3. **P&E was explored and abandoned as a first-class LangGraph pattern.** The tutorial existed, was taught as part of LangGraph's official documentation, and was removed in LangGraph 1.0. The redirect to `langchain/middleware/built-in#to-do-list` makes the reclassification explicit: P&E now belongs to the `langchain` agent middleware layer, not the LangGraph graph layer.

4. **LangGraph 1.0's direction is infrastructure-first.** Moving `create_react_agent` to `langchain.agents`, deprecating `HumanInterruptConfig` to `langchain.agents.interrupt` — these moves signal that LangGraph is thinning its own pattern layer.

**For the blog "④b is empty" argument**: The more precise statement is **"④b was attempted by LangGraph and deliberately relocated to a higher layer."** ④b is not conceptually empty — the primitives can do it, the tutorials existed. But no major framework currently maintains P&E as a first-class graph pattern. **LangGraph's 1.0 pivot makes ④b "silent rather than empty": possible, explored, and quietly moved to middleware.**

---

## Key Architectural Insights

**1. BSP is not a DAG engine.** Termination on "no tasks fired this step" makes cycles structurally trivial. This is LangGraph's deepest architectural difference from every other framework analyzed. It explains why LangGraph can express agent loops as graph topology while Mastra/ADK need explicit loop code inside agents.

**2. `create_react_agent` is pure graph assembly.** The ReAct agent is `StateGraph` + two `add_node()` + one `add_conditional_edges()`. There is no special ReAct execution code. The loop emerges from topology. Any developer who understands the primitives can assemble P&E the exact same way — the framework does not distinguish.

**3. LangGraph 1.0 is moving "up" and thinning itself.** Deprecated symbols moved to `langchain`: `create_react_agent` → `langchain.agents`, `HumanInterruptConfig` → `langchain.agents.interrupt`, P&E tutorial → langchain middleware docs. LangGraph is positioning itself as pure infrastructure, not an agent framework.

**4. The P&E redirect is the most diagnostic evidence.** The redirect to `langchain/middleware/built-in#to-do-list` signals that LangGraph team reconceives P&E as a middleware concern — a layer above graph topology. This is the clearest official statement about where LangGraph does and does not want to be.

**5. Channel typing is weaker than Mastra's Zod.** LangGraph's TypedDict annotations provide static analysis support but not runtime contract enforcement at node boundaries. For production typed-dataflow architectures requiring guaranteed schema validation at every interface, Mastra's Zod approach is stronger.

---

## Conclusion

LangGraph is the **execution substrate of the modern LangChain ecosystem**, not an opinionated Agent framework itself. By dominant usage it is ④a (via `create_react_agent` / `create_agent`); by theoretical capability it is meta-level (primitives express any agent pattern including P&E).

**The most important finding for the blog**: LangGraph historically maintained first-class P&E tutorials (`plan-and-execute`, `rewoo`, `llm-compiler`), and in the 1.0 release **deliberately relocated them to langchain middleware**. The redirect to `langchain/middleware/built-in#to-do-list` is the cleanest public statement that the LangChain/LangGraph team has evaluated P&E and chosen to reclassify it: not a graph pattern, but a middleware pattern. This makes ④b "silent rather than empty" — possible in primitives, explored in tutorials, but quietly removed from the official recommendation surface.

**For the blog's central argument**: The user's ⑤ (P&E + typed dataflow) is not alone because "no one thought of P&E." It is alone because the frameworks that thought of P&E (LangChain experimental, LangGraph tutorials) deliberately moved it to middleware or dropped it entirely. **The user is not in a direction no one explored; the user is in a direction the industry explored and walked away from.** This is a more interesting story.
