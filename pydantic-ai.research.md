# Pydantic AI Agent Architecture Research

Last Updated: 2026-04-12

> **Research Methodology**: Source-level analysis of pydantic/pydantic-ai monorepo at `/Users/linguanguo/dev/llm-agent-research/pydantic-ai`. Primary files: `pydantic_graph/pydantic_graph/nodes.py`, `pydantic_graph/pydantic_graph/graph.py`, `pydantic_ai_slim/pydantic_ai/_agent_graph.py`, `pydantic_ai_slim/pydantic_ai/agent/__init__.py`. Every architectural claim cites file:line.

## Sources

**Core Agent Files**
- `pydantic_ai_slim/pydantic_ai/agent/__init__.py:141-350` — `Agent` class and `__init__` API
- `pydantic_ai_slim/pydantic_ai/_agent_graph.py:73-155` — `GraphAgentState`, `GraphAgentDeps`, `AgentNode` base
- `pydantic_ai_slim/pydantic_ai/_agent_graph.py:172-290` — `UserPromptNode`
- `pydantic_ai_slim/pydantic_ai/_agent_graph.py:515-948` — `ModelRequestNode`
- `pydantic_ai_slim/pydantic_ai/_agent_graph.py:951-1232` — `CallToolsNode`
- `pydantic_ai_slim/pydantic_ai/_agent_graph.py:1824-1853` — `build_agent_graph()` factory

**pydantic_graph Library**
- `pydantic_graph/pydantic_graph/nodes.py:1-205` — `BaseNode`, `End`, `Edge`, `GraphRunContext`, `NodeDef`
- `pydantic_graph/pydantic_graph/graph.py:1-800` — `Graph`, `GraphRun`, `GraphRunResult`
- `pydantic_graph/pydantic_graph/beta/graph_builder.py:64-160` — `GraphBuilder` (beta fluent API)
- `pydantic_graph/pydantic_graph/beta/step.py:25-100` — `StepContext`, `StepFunction` protocols
- `pydantic_graph/pyproject.toml` — standalone package declaration

**Tool Schema**
- `pydantic_ai_slim/pydantic_ai/_function_schema.py:90-200` — `function_schema()` introspection engine

**Documentation**
- `docs/multi-agent-applications.md` — official multi-agent patterns
- `docs/graph.md` — pydantic_graph documentation

---

## 1. Agent Definition (API Layer)

### 1.1 Agent class and type parameters

`Agent` at `agent/__init__.py:141`:

```python
@dataclasses.dataclass(init=False)
class Agent(AbstractAgent[AgentDepsT, OutputDataT]):
    def __init__(
        self,
        model: models.Model | models.KnownModelName | str | None = None,
        *,
        output_type: OutputSpec[OutputDataT] = str,
        deps_type: type[AgentDepsT] = NoneType,
        ...
    ):
```

`Agent` is generic over `AgentDepsT` (dependencies) and `OutputDataT` (result type). The `output_type` parameter accepts `str`, a Pydantic `BaseModel`, a union, or an output function. `deps_type` propagates `AgentDepsT` through `RunContext[AgentDepsT]` and tool signatures.

### 1.2 Tool definition (automatic schema from signatures)

Tools are registered via `@agent.tool` decorator. The engine at `_function_schema.py:90` uses `inspect.signature()` and `get_type_hints()` to auto-extract field names, types, defaults, and docstring descriptions. **No explicit schema declaration needed:**

```python
@agent.tool
async def get_weather(ctx: RunContext[MyDeps], city: str, units: Literal['C', 'F']) -> str:
    """Get current weather for a city."""
    ...
```

Compared to Mastra's `createTool({ inputSchema: z.object({...}) })`, Pydantic AI infers the complete schema from the Python function signature. The docstring becomes the tool description. This is tighter language integration than any other framework.

### 1.3 Output type parameterization

When `output_type` is a Pydantic model, the agent generates a synthetic `final_result` tool with the model's JSON schema and forces the LLM to call it. The return is validated by `SchemaValidator`. The `Agent[DepsT, OutputT]` declaration propagates `OutputT` through to `AgentRunResult[OutputT].output`.

---

## 2. Execution Model

### 2.1 The main loop

When `agent.run(user_prompt)` is called, at `agent/__init__.py:1116`:

```python
graph = _agent_graph.build_agent_graph(self.name, self._deps_type, output_type_)
```

The resulting graph has a **fixed topology of four node types** that always runs in this pattern:

```
UserPromptNode → ModelRequestNode → CallToolsNode → [ModelRequestNode | End]
                                                          ↑________________|
                                       (loop back if tool calls returned)
```

Node roles:
1. `UserPromptNode.run()` — assembles system prompts + user message → returns `ModelRequestNode`
2. `ModelRequestNode.run()` — calls the LLM, receives `ModelResponse` → returns `CallToolsNode`
3. `CallToolsNode.run()` — examines response: if tool calls present, execute them and return `ModelRequestNode` (loop); if final output, return `End(FinalResult)`

`SetFinalResult` handles the streaming path. `build_agent_graph()` at `_agent_graph.py:1824-1853` always creates these same four nodes regardless of agent configuration.

### 2.2 Is it ReAct?

**Yes — standard ReAct.** The LLM decides each turn whether to call tools or produce final output. The 4-node graph is fixed infrastructure that does not change based on input. There is no planning step, no plan artifact, no pre-execution topology generation.

**Key evidence**: `build_agent_graph()` at line 1824 creates the same four nodes regardless of agent configuration, output type, or tool set. Topology is not input-dependent. Only loop termination depends on LLM-emitted `finish_reason` + presence of tool calls.

---

## 3. The pydantic_graph Library

### 3.1 Standalone library or pydantic-ai internal?

**`pydantic-graph` is a separately published standalone library.** Its `pyproject.toml:14` declares `name = "pydantic-graph"`. `pydantic-ai-slim` depends on it: `"pydantic-graph=={{ version }}"` (`pydantic_ai_slim/pyproject.toml:60`).

`docs/graph.md:17-20` explicitly states: *"This library is developed as part of Pydantic AI; it has no dependency on `pydantic-ai` and can be considered as a pure graph-based state machine library."*

### 3.2 Core abstractions

`BaseNode[StateT, DepsT, NodeRunEndT]` (`nodes.py:37`):

```python
class BaseNode(ABC, Generic[StateT, DepsT, NodeRunEndT]):
    @abstractmethod
    async def run(
        self, ctx: GraphRunContext[StateT, DepsT]
    ) -> BaseNode[StateT, DepsT, Any] | End[NodeRunEndT]:
        ...
```

**The return type annotation is machine-read at graph construction time** to determine legal node transitions. `get_node_def()` at `nodes.py:104-136` calls `get_type_hints(cls.run)` and `_utils.get_union_args()` to extract every concrete node type in the return union. These become the `next_node_edges` of the `NodeDef`. **This is the architecturally distinctive feature — no other framework studied extracts edges from return type annotations.**

`GraphRunContext[StateT, DepsT]` carries `state: StateT` and `deps: DepsT`.

`End[RunEndT]` signals completion with `data: RunEndT`.

`Graph[StateT, DepsT, RunEndT]` (`graph.py:25`) validates at `__init__` time that all node references are registered and types agree across nodes.

### 3.3 Type parameterization

Three generic parameters of `BaseNode`:

- **`StateT`** — shared mutable state threaded through all nodes (must match across all nodes)
- **`DepsT`** — read-only dependency injection (contravariant)
- **`NodeRunEndT`** — covariant, applies when node returns `End()`; types the terminal output

`Graph[StateT, DepsT, RunEndT]` infers these types from the node list at `graph.py:462-493` by reading `typing_extensions.get_original_bases(node_def.node)`. Mismatches are caught at construction time.

### 3.4 Execution model

`Graph.run()` → `Graph.iter()` → `GraphRun`. `GraphRun.__anext__()` at `graph.py:757-766`:

```python
async def __anext__(self):
    if isinstance(self._next_node, End):
        raise StopAsyncIteration
    return await self.next(self._next_node)
```

`next()` calls `node.run(ctx)` and stores the returned next node in `self._next_node`. State persisted via `BaseStatePersistence` after each node. Loop until `End`.

---

## 4. The _agent_graph.py Module

### 4.1 How agent is built on top of pydantic_graph

All three agent node classes extend `AgentNode` (`_agent_graph.py:148`):

```python
class AgentNode(
    BaseNode[
        GraphAgentState,
        GraphAgentDeps[DepsT, Any],
        result.FinalResult[NodeRunEndT]
    ]
):
    ...
```

This parameterization fixes state type as `GraphAgentState` and deps type as `GraphAgentDeps[DepsT, OutputT]` for all agent nodes. `OutputT` threads through all three nodes and into `End[result.FinalResult[OutputT]]`.

`build_agent_graph()` at line 1824 uses the beta `GraphBuilder`:

```python
def build_agent_graph(name, deps_type, output_type) -> Graph[...]:
    g = GraphBuilder(
        state_type=GraphAgentState,
        deps_type=GraphAgentDeps[DepsT, OutputT],
        input_type=UserPromptNode[DepsT, OutputT],
        output_type=result.FinalResult[OutputT],
        auto_instrument=False,
    )
    g.add(
        g.edge_from(g.start_node).to(UserPromptNode[DepsT, OutputT]),
        g.node(UserPromptNode[DepsT, OutputT]),
        g.node(ModelRequestNode[DepsT, OutputT]),
        g.node(CallToolsNode[DepsT, OutputT]),
        g.node(SetFinalResult[DepsT, OutputT]),
    )
    return g.build(validate_graph_structure=False)
```

Rebuilt on each `agent._run()` call, not cached.

### 4.2 Flow-level typing? (Critical Question)

`OutputT` is threaded through the entire execution — from `Agent[DepsT, OutputT]` declaration through `GraphAgentDeps[DepsT, OutputT]` through `CallToolsNode[DepsT, OutputT]` to `End[result.FinalResult[OutputT]]`.

**However**, the data channel between nodes is **not typed at the step-output level**. Nodes communicate via shared mutable `GraphAgentState.message_history: list[ModelMessage]` — a bag-typed list. `UserPromptNode.run()` returns a `ModelRequestNode` instance (not data); `ModelRequestNode.run()` returns a `CallToolsNode` instance. The actual conversation content flows through `GraphAgentState`.

**The critical distinction**: **node transition destinations are type-checked; inter-node payload types are not.** The graph knows `UserPromptNode` transitions to `ModelRequestNode`, but the content `ModelRequestNode` finds in `state.message_history` is not checked by the type system.

This is a **typed state machine**, not **typed dataflow**. StateT is a shared mutable bag, analogous to ADK's `session.state` but at least with a named type.

---

## 5. Multi-Agent Support

Pydantic AI documents five multi-agent levels (`docs/multi-agent-applications.md:3-9`). Two primary patterns:

**Pattern A: Agent delegation (tool-based, default)**

```python
@joke_selection_agent.tool
async def joke_factory(ctx: RunContext[None], count: int) -> list[str]:
    r = await joke_generation_agent.run(f'Please generate {count} jokes.', usage=ctx.usage)
    return r.output  # only output is returned; sub-agent message history is discarded
```

Only `r.output` propagates back. Sub-agent message history and intermediate reasoning lost. **Same tool-based handoff failure mode as Mastra and Google ADK `AgentTool`.** However, `usage=ctx.usage` aggregates token usage across agents — a small mitigation.

**Pattern B: Graph-based multi-agent (developer-built pydantic_graph code)**

```python
@dataclass
class ResearchNode(BaseNode[PipelineState]):
    async def run(self, ctx: GraphRunContext[PipelineState]) -> SummarizeNode:
        result = await researcher_agent.run(ctx.state.topic)
        ctx.state.research = result.output
        return SummarizeNode()
```

Developer writes their own `pydantic_graph` graph where nodes call agents. Full trace can be preserved **if the developer stores it in state**. Node transitions between agents are type-checked. This is the only pattern where trace preservation is possible — but it requires significant developer effort and is not framework-provided.

---

## 6. Typing and Validation

### 6.1 Compile-time type tracking

Pydantic AI achieves compile-time type safety at four levels:

1. **Agent return type**: `agent.run_sync()` returns `AgentRunResult[OutputDataT]`; `result.output` has type `OutputDataT`
2. **Tool `RunContext` parameterization**: `@agent.tool` functions taking `RunContext[AgentDepsT]` are type-checked against agent's `AgentDepsT`
3. **pydantic_graph node transitions**: return type of `BaseNode.run()` defines legal transitions; mismatches caught at graph construction + by type checker
4. **Output validators**: `@agent.output_validator` receives and returns `OutputDataT`, type-checked

### 6.2 Runtime validation

Pydantic v2 `SchemaValidator` at every boundary:
- Tool call arguments (validated against function type hints)
- Final output (validated against `output_type` schema)
- Output validators (user-defined)

**Automatic retries** on validation failure: agent re-prompts the LLM with a `RetryPromptPart` containing the validation error, up to `max_result_retries` times. This is the **strongest differentiator vs. Mastra** — Mastra requires a separate `StructuredOutputProcessor` agent for this behavior.

### 6.3 Comparison with user's typed dataflow (⑤)

| Dimension | Pydantic AI | User's ⑤ (P&E + typed dataflow) |
|-----------|------------|----------------------------------|
| Agent input type | Validated (Pydantic) | Validated |
| Agent output type | `OutputDataT` propagated via generics | Parameterized |
| **Inter-step type contract** | **Nodes return node instances; data via shared `list[ModelMessage]` bag** | **Step A output type = Step B input type, enforced** |
| Plan artifact | None | Yes (explicit Plan object) |
| Pre-execution cost | None | Estimated in plan |
| LLM decision per step | Yes (ReAct) | No (execution follows plan) |

**The `pydantic_graph` `BaseNode[StateT, DepsT, NodeRunEndT]` is the closest thing to flow-level typing in any public framework found so far** — but `StateT` is a shared mutable object, not a per-step typed output pipe. **The Plan artifact and typed step-to-step contracts of the user's ⑤ system have no equivalent.**

---

## 7. Representative Examples

### 7.1 Simple Agent with structured output

```python
from pydantic import BaseModel
from pydantic_ai import Agent, RunContext

class WeatherResult(BaseModel):
    temperature: float
    condition: str

agent: Agent[None, WeatherResult] = Agent(
    'openai:gpt-4o',
    output_type=WeatherResult,
    system_prompt='You are a weather assistant.',
)

@agent.tool
async def get_current_weather(ctx: RunContext[None], city: str) -> dict:
    return {'temperature': 22.5, 'condition': 'sunny'}

result = agent.run_sync('What is the weather in Paris?')
print(result.output)  # type: WeatherResult, validated by Pydantic
```

`output_type=WeatherResult` generates a `final_result` tool with WeatherResult's JSON schema. The LLM is forced to call this tool with valid WeatherResult JSON. `result.output` is a validated `WeatherResult` instance.

### 7.2 Graph-based multi-agent pipeline

```python
from pydantic_graph import BaseNode, End, Graph, GraphRunContext
from dataclasses import dataclass

@dataclass
class PipelineState:
    topic: str
    research: str = ''
    summary: str = ''

@dataclass
class ResearchNode(BaseNode[PipelineState]):
    async def run(self, ctx: GraphRunContext[PipelineState]) -> SummarizeNode:
        result = await researcher_agent.run(ctx.state.topic)
        ctx.state.research = result.output
        return SummarizeNode()

@dataclass
class SummarizeNode(BaseNode[PipelineState, None, str]):
    async def run(self, ctx: GraphRunContext[PipelineState]) -> End[str]:
        result = await summarizer_agent.run(ctx.state.research)
        return End(result.output)

pipeline = Graph(nodes=[ResearchNode, SummarizeNode])
result = pipeline.run_sync(ResearchNode(), state=PipelineState(topic='climate'))
```

Developer-written infrastructure. `PipelineState` is typed; node transitions checked at construction. Both agents' message histories still discarded on `.run()` return.

---

## 8. Direct Comparison with Mastra / Google ADK / AgentScope

| Dimension | Mastra | Google ADK | AgentScope | Pydantic AI |
|-----------|--------|------------|------------|-------------|
| **Execution model** | ReAct loop | ReAct loop | ReAct loop | ReAct loop |
| **Plan artifact** | None | None | **`Plan`/`SubTask` Pydantic models (real)** | None |
| **Plan enforcement** | N/A | N/A | Prompt-based hints | N/A |
| **Framework typing** | Zod at step/tool boundaries | Pydantic for framework objects; session.state untyped | Pydantic for Plan models | **Python generics at agent + node level (strongest)** |
| **Flow-level typing** | None | None | None | None (typed state machine, not typed dataflow) |
| **Multi-agent handoff** | Tool-based (trace lost) | Tool-based or callbacks | Tool-based | Tool-based (trace lost) or programmatic |
| **Graph execution** | Workflow DAG outer, ReAct inner | No | Pipeline classes | **pydantic_graph as execution substrate** |
| **Output validation** | Zod + external processor | Pydantic if model returns BaseModel | Not centralized | **Pydantic v2 with auto-retry** |
| **Tool schema** | Explicit `z.object({...})` | Python type hints | Pydantic models | **Auto from function signature** |

**Key distinction from Mastra**: Pydantic AI's generics propagate `OutputDataT` from `Agent[Deps, Output]` to `AgentRunResult[Output].output` natively in Python. Mastra's `createStep()` has `outputSchema` but requires explicit annotation at each step; types do not chain automatically.

**Key distinction from AgentScope**: AgentScope has genuine `Plan`/`SubTask` Pydantic models representing a pre-execution plan. Pydantic AI has no equivalent. **AgentScope is closer to ④b than Pydantic AI despite both using Pydantic.**

**Key distinction from Google ADK**: ADK uses `session.state: dict[str, Any]` completely untyped. Pydantic AI's `StateT` is a typed parameter — meaningful improvement even though both are mutable shared objects.

---

## 9. Position on the Agent Architecture Spectrum

**Verdict: Pydantic AI is ④a (Code Framework, ReAct school), with the strongest typing infrastructure of any public framework in that category — but it is not ④b and not a new ④c.**

Reasoning:

1. **No Plan artifact**: `build_agent_graph()` always creates the same fixed 4-node graph. No planning step, no pre-execution structured plan, no cost prediction. Conclusively ④a.

2. **ReAct loop confirmed**: `UserPromptNode → ModelRequestNode → CallToolsNode → loop`. The LLM decides each turn. Standard ReAct, identical pattern to Mastra and ADK.

3. **Typed state machine, not typed dataflow**: `pydantic_graph` provides typed node transitions (encoded in return type annotations, validated at construction time). This is genuinely novel. But typed state machine ≠ typed dataflow: the inter-node channel is `GraphAgentState.message_history: list[ModelMessage]` — a shared bag, not a typed pipe.

4. **The user-facing pydantic_graph API is the closest thing to ④c infrastructure in any public framework**: When a developer writes their own graph directly, `BaseNode[StateT, DepsT, NodeRunEndT]` gives flow-level state typing via `StateT`. All nodes must agree on `StateT`. Edge extraction from return type annotations is unique. **But this is a user-extensible graph library, not the Agent execution model itself.**

5. **The pydantic-ai `Agent` does not expose this typing to users**: The agent's internal pydantic_graph usage is an implementation detail with `GraphAgentState` as an opaque bag. Users of `Agent` do not interact with graph topology at all.

**The user's ⑤ claim remains distinctive**: Pydantic AI implements neither a Plan artifact nor typed step-to-step data contracts. `pydantic_graph`'s `StateT` threading is the closest analogue found in any public framework, but `StateT` is a shared mutable object, not a typed pipe. The Plan artifact + "step A's output type is step B's input type" contract of the user's ⑤ has no equivalent.

---

## Key Architectural Insights

1. **`pydantic_graph` is architecturally the most interesting public graph library in the AI ecosystem**: Reading node transitions from Python return type annotations at construction time, with runtime validation, has no equivalent in Mastra, ADK, or AgentScope. Genuine design innovation.

2. **Agent uses pydantic_graph as execution substrate, not workflow definition**: The 4-node graph is fixed infrastructure invisible to most users. The graph feature is primarily useful when developers drop down to pydantic_graph directly to build multi-agent topologies.

3. **Tool schema from function signatures is a genuine ergonomic advantage**: The `_function_schema.py` introspection engine extracts JSON schema from Python type hints automatically. Tighter language integration than any competitor.

4. **Beta `GraphBuilder` API suggests future direction**: `pydantic_graph/pydantic_graph/beta/` adds fluent `.step().then().fork().join()` API, suggesting movement toward DAG-based workflow builder similar to Mastra Workflow but with stronger Python generics. ④c-like capabilities could emerge here, but beta and unstable.

5. **Auto-retry with validation error feedback is the strongest differentiator vs. Mastra**: When `output_type` is a Pydantic model, the agent automatically retries with structured error prompt if LLM output fails validation. Mastra requires a separate processor.

6. **Multi-agent handoff remains tool-based by default**: Same trace-loss limitation as Mastra and ADK. Only the programmatic hand-off pattern (using `result.all_messages()`) preserves trace, and it requires developer discipline.

---

## Conclusion

Pydantic AI is firmly **④a (Code Framework, ReAct school)** — the most type-sophisticated framework in that category, but still ReAct in execution model.

Unique contributions:
- Standalone typed state machine library (`pydantic_graph`) encoding node transitions as Python type annotations
- Automatic tool schema extraction from function signatures
- Tightly parameterized `Agent[DepsT, OutputT]` propagating types through entire call chain via Python generics
- Automatic structured output retry with Pydantic validation error feedback

What it lacks relative to ④b or ⑤:
- No Plan artifact or planning phase
- No pre-execution cost or step estimation
- **No flow-level typing between agent steps** (inter-node channel is `list[ModelMessage]`, not typed step outputs)
- Graph infrastructure under `Agent` is fixed and opaque; typed graph workflows require writing `pydantic_graph` directly

**For the blog**: Pydantic AI should be positioned in ④a alongside Mastra and ADK, with a footnote that it has the most sophisticated type infrastructure in that tier. The "typed dataflow ④c" or "④b P&E" claims do not apply. **The user's ⑤ uniqueness claim survives**: the Plan artifact combined with typed step-to-step contracts is not present in any public framework, including Pydantic AI.
