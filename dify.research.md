# Dify Agent Architecture Research

Last Updated: 2026-04-12

> **Research Methodology**: Source-level analysis of langgenius/dify at `/Users/linguanguo/dev/llm-agent-research/dify` (shallow clone). The central question: is Dify's backend workflow engine structurally equivalent to Mastra's Workflow DAG / Google ADK's agent orchestration, or is it fundamentally different?

## Sources

**Workflow Engine Entry**
- `api/core/workflow/workflow_entry.py:1-230` — `WorkflowEntry`: assembles `GraphEngine`, layers
- `api/core/workflow/node_factory.py:1-525` — `DifyNodeFactory`, node registry, node type resolution
- `api/core/workflow/variable_pool_initializer.py:1-16` — shared `VariablePool` API

**Agent Node (Workflow)**
- `api/core/workflow/nodes/agent/agent_node.py:1-195` — `AgentNode._run()`, strategy dispatch
- `api/core/workflow/nodes/agent/entities.py:1-48` — `AgentNodeData`, `AgentInput` type union
- `api/core/workflow/nodes/agent/strategy_protocols.py:1-40` — `AgentStrategyResolver` Protocol
- `api/core/workflow/nodes/agent/runtime_support.py:1-160` — parameter resolution from `VariablePool`
- `api/core/workflow/nodes/agent/plugin_strategy_adapter.py:1-41` — bridge to plugin daemon

**Standalone Agent (`agent_chat`)**
- `api/core/agent/fc_agent_runner.py:1-382` — `FunctionCallAgentRunner.run()`: FC loop
- `api/core/agent/cot_agent_runner.py:1-438` — `CotAgentRunner.run()`: ReAct text-parse loop
- `api/core/agent/entities.py:1-94` — `AgentEntity.Strategy`, `AgentScratchpadUnit`
- `api/core/agent/prompt/template.py:1-107` — ReAct prompt template (Thought/Action/Observation)
- `api/core/agent/output_parser/cot_output_parser.py:1-80` — streaming ReAct output parser
- `api/core/agent/strategy/base.py:1-46` — `BaseAgentStrategy` ABC
- `api/core/agent/strategy/plugin.py:1-65` — `PluginAgentStrategy`

**App Runners**
- `api/core/app/apps/agent_chat/app_runner.py:1-243` — `AgentChatAppRunner`: strategy selection
- `api/core/app/apps/workflow/app_runner.py:1-177` — `WorkflowAppRunner`: Redis channel + `WorkflowEntry`
- `api/core/app/apps/advanced_chat/app_runner.py:1-80` — chat + workflow fusion
- `api/core/app/apps/workflow_app_runner.py:1-165` — shared graph init
- `api/core/app/apps/base_app_queue_manager.py:1-60` — `AppQueueManager`: in-process `queue.Queue`

**Graphon (External Package)**
- `graphon>=0.1.2` (github.com/langgenius/graphon) — **extracted graph engine library**
- `GraphEngine`: queue-based worker pool + command channels + extensible layers
- `VariablePool`: two-level dict keyed by `[node_id, var_name]`

---

## 1. Workflow Execution Engine (`core/workflow/`)

### 1.1 Core Classes — **Graphon is the real engine**

The workflow engine in Dify is **not implemented in `api/core/workflow/` directly**. As of Dify 2.x, the execution engine has been extracted into a separate Python package called **graphon** (`graphon>=0.1.2`), maintained at `github.com/langgenius/graphon`. The `api/core/workflow/` directory is now an **orchestration layer** that configures and drives graphon.

The principal classes:

- **`graphon.graph.Graph`** — parsed graph structure from JSON config (`nodes` + `edges`)
- **`graphon.graph_engine.GraphEngine`** — the execution runtime, created in `WorkflowEntry.__init__` (`workflow_entry.py:181-193`)
- **`graphon.runtime.VariablePool`** — shared state dictionary
- **`graphon.runtime.GraphRuntimeState`** — wraps `VariablePool` + `start_at` timestamp + `execution_context`
- **`WorkflowEntry`** (`workflow_entry.py:135`) — Dify's façade: creates `GraphEngine`, attaches layers (`ExecutionLimitsLayer`, `LLMQuotaLayer`, `ObservabilityLayer`, `WorkflowPersistenceLayer`)

`WorkflowEntry.run()` at `workflow_entry.py:218-230` is literally:

```python
def run(self) -> Generator[GraphEngineEvent, None, None]:
    graph_engine = self.graph_engine
    generator = graph_engine.run()
    yield from generator
```

Everything downstream is graphon.

### 1.2 Main Execution Loop

`GraphEngine.run()` inside graphon:

1. Initialize layers (debug logging, execution limits, quota, observability)
2. Emit `GraphRunStartedEvent`
3. Call `_start_execution()`:
   - Start a **worker pool** (size configurable via `GRAPH_ENGINE_MIN_WORKERS` / `GRAPH_ENGINE_MAX_WORKERS`)
   - Enqueue the root node into a **ready queue**
   - Start a **dispatcher** that reads from an event queue
4. Workers pull node IDs from the ready queue, execute `node.run()` which yields events
5. Events flow to the dispatcher → `EventHandler` registry → `EdgeProcessor` determines next eligible nodes → enqueue successors
6. External control via a **command channel** (`InMemoryChannel` for single-step, `RedisChannel` for workflow app runs)
7. Yield `GraphEngineEvent` objects out to the caller

### 1.3 Execution Paradigm: **Queue-Based Worker Pool**

This is **not** topological sort, **not** Pregel/BSP, and **not** a state machine. It is a **queue-based, event-driven, concurrent worker pool**:

- A ready queue holds node IDs whose predecessors have all completed
- A dynamic worker pool (thread pool, configurable min/max) pulls from the ready queue
- When a node finishes, the dispatcher processes edge conditions and enqueues successor nodes
- Control signals (abort, pause, variable update) are delivered through a command channel, which can be Redis-backed for distributed operation

This architecture was refactored in Dify 2.0 (GitHub Issue #24067) from a monolithic single-threaded executor to the current queue-based design.

**Configuration** (`workflow_entry.py:186-190`):
```python
GraphEngineConfig(
    min_workers=dify_config.GRAPH_ENGINE_MIN_WORKERS,
    max_workers=dify_config.GRAPH_ENGINE_MAX_WORKERS,
    scale_up_threshold=dify_config.GRAPH_ENGINE_SCALE_UP_THRESHOLD,
    scale_down_idle_time=dify_config.GRAPH_ENGINE_SCALE_DOWN_IDLE_TIME,
)
```

---

## 2. Node Types Inventory (24 types)

Graphon's `BuiltinNodeTypes` enum defines 24 built-in node types:

| Type | Purpose |
|------|---------|
| `start` / `end` / `answer` | Lifecycle |
| `llm` | LLM call node |
| `knowledge_retrieval` | RAG |
| `if_else` | Conditional branching |
| `code` | Sandboxed code execution |
| `template_transform` | Jinja2 rendering |
| `question_classifier` | LLM-based routing |
| `http_request` | HTTP call |
| `tool` | Tool invocation |
| `datasource` | External data source |
| `variable_aggregator` / `legacy_variable_aggregator` | Merge variables |
| `loop` / `loop_start` / `loop_end` | For-each loop |
| `iteration` / `iteration_start` | Iteration control |
| `parameter_extractor` | LLM-based param extraction |
| `variable_assigner` | Variable assignment |
| `document_extractor` | File/PDF parsing |
| `list_operator` | List manipulation |
| **`agent`** | **Agent-as-node** |
| `human_input` | Human-in-the-loop |

---

## 3. Agent Node — **Plugin Dispatch Boundary**

### 3.1 Structure

`AgentNode` (`agent_node.py:29`) inherits from graphon's `Node[AgentNodeData]`. Its configuration is:

```python
class AgentNodeData(BaseNodeData):
    type: NodeType = BuiltinNodeTypes.AGENT
    agent_strategy_provider_name: str   # plugin provider, e.g. "langgenius/agent_strategies"
    agent_strategy_name: str            # strategy name, e.g. "react" or "function_calling"
    memory: MemoryConfig | None = None
    agent_parameters: dict[str, AgentInput]
```

### 3.2 The Agent Node Does NOT Contain a ReAct Loop

`AgentNode._run()` at `agent_node.py:74-170`:

```python
strategy = self._strategy_resolver.resolve(
    tenant_id=..., agent_strategy_provider_name=..., agent_strategy_name=...
)
parameters = self._runtime_support.build_parameters(...)
message_stream = strategy.invoke(params=parameters, ...)
yield from self._message_transformer.transform(messages=message_stream, ...)
```

**The key insight**: the workflow Agent node **delegates entirely to a plugin strategy via `strategy.invoke()`**. The plugin strategy is resolved by `PluginAgentStrategyResolver` → `PluginAgentClient`, which communicates with a **Dify plugin daemon over an RPC channel**. The actual loop (ReAct iterations, tool calls) runs **inside the plugin daemon process**, not in Dify core.

This is a major architectural shift from older Dify: **the Agent node in Dify 2.x is a plugin dispatch boundary, not a self-contained loop.**

---

## 4. Standalone Agent Runner — Two Separate Implementations

### 4.1 Purpose

`api/core/agent/` contains the **legacy agent execution infrastructure**, used **exclusively by the `agent_chat` app type**. It has **nothing to do** with the workflow `AgentNode`.

### 4.2 Two Completely Independent Agent Execution Paths

| Dimension | `agent_chat` app | Workflow `AgentNode` |
|-----------|-----------------|---------------------|
| Location | `core/agent/` | `core/workflow/nodes/agent/` |
| Loop implementation | In-process Python `while` loop | Plugin daemon (external process) |
| Strategy | `AgentEntity.Strategy.CHAIN_OF_THOUGHT` or `FUNCTION_CALLING` | Any plugin strategy |
| Entry point | `AgentChatAppRunner.run()` | `AgentNode._run()` |
| Tool calling | Direct `ToolEngine.agent_invoke()` | Via plugin daemon stream |

### 4.3 ReAct Strategy vs Function Calling Strategy

**These are two genuinely different implementations**, not a flag branch.

**Function Calling (`FunctionCallAgentRunner`, `fc_agent_runner.py:36-320`)**:

```python
while function_call_state and iteration_step <= max_iteration_steps:
    function_call_state = False
    chunks = model_instance.invoke_llm(
        prompt_messages=prompt_messages,
        tools=prompt_messages_tools,   # LLM gets full tool schema
        stream=self.stream_tool_call,
        ...
    )
    for chunk in chunks:
        if self.check_tool_calls(chunk):
            function_call_state = True
            tool_calls.extend(self.extract_tool_calls(chunk))
    for tool_call_id, tool_call_name, tool_call_args in tool_calls:
        tool_invoke_response = ToolEngine.agent_invoke(...)
    iteration_step += 1
```

**Chain of Thought / ReAct (`CotAgentRunner`, `cot_agent_runner.py:32-438`)**:

```python
while function_call_state and iteration_step <= max_iteration_steps:
    chunks = model_instance.invoke_llm(
        prompt_messages=prompt_messages,
        tools=[],          # no tool schema — pure text
        stop=["Observation"],   # stop at Observation token
        stream=True,
    )
    react_chunks = CotAgentOutputParser.handle_react_stream_output(chunks, usage_dict)
    for chunk in react_chunks:
        if isinstance(chunk, AgentScratchpadUnit.Action):
            # extracted Action from text
        ...
    if scratchpad.action.action_name.lower() == "final answer":
        break
    else:
        tool_invoke_response = self._handle_invoke_action(action, ...)
```

The prompt template (`prompt/template.py:1-44`) explicitly encodes the ReAct format:

```
Thought: consider previous and subsequent steps
Action: {"action": $TOOL_NAME, "action_input": $ACTION_INPUT}
Observation: action result
... (repeat N times)
Action: {"action": "Final Answer", "action_input": "Final response"}
```

The stop token `"Observation"` is injected into `model_conf.stop` so the LLM halts before generating the observation.

**`AgentChatAppRunner` selects between them** (`app_runner.py:185-210`):
- If model supports `MULTI_TOOL_CALL` or `TOOL_CALL` features → force `FUNCTION_CALLING`
- Else if strategy is `CHAIN_OF_THOUGHT` → use `CotChatAgentRunner` or `CotCompletionAgentRunner`

---

## 5. App Types

### 5.1 `agent_chat` App

Entry: `AgentChatAppGenerator.generate()` → worker thread → `AgentChatAppRunner.run()` → FC or CoT runner → `runner.run(message, query, inputs)`.

The agent runner publishes events to an in-process `queue.Queue` via `AppQueueManager.publish()`. The main thread reads from this queue and streams to the HTTP client.

**Architecture**: traditional chat app. No graph, no workflow. The agent loop is a Python `while` loop with `iteration_step <= max_iteration_steps`. Max iterations capped at `min(agent.max_iteration, 99) + 1`.

### 5.2 `workflow` App

Entry: `WorkflowAppGenerator.generate()` → worker thread → `WorkflowAppRunner.run()` → creates `VariablePool` + `GraphRuntimeState` + `WorkflowEntry` → `workflow_entry.run()` → yields `GraphEngineEvent` stream.

A **`RedisChannel`** is created per execution (`app_runner.py:136-137`):
```python
channel_key = f"workflow:{task_id}:commands"
command_channel = RedisChannel(redis_client, channel_key)
```

This enables external abort/pause signals to reach the in-flight `GraphEngine`. The workflow graph config is a JSON structure with `nodes` and `edges` arrays loaded from the database.

**Architecture**: graph-based. The graph is a DAG of typed nodes. The engine is a concurrent worker pool. Node outputs land in the shared `VariablePool`.

### 5.3 `advanced_chat` App

Combines both: prepends a message history context to the variable pool, then runs the workflow graph (same `WorkflowEntry` path as the `workflow` app). "Chatbot with workflow" mode — a chat interface where the backend is a full DAG.

---

## 6. State and Data Flow

### 6.1 Inter-Node Data Passing

All inter-node data goes through a single **`VariablePool`** instance, shared across all nodes in one execution run.

`VariablePool` is a two-level nested dict keyed by `[node_id, variable_name]`:
```python
variable_dictionary: defaultdict[str, dict[str, Variable]]
```

A node reads another node's output by accessing `variable_pool.get(["previous_node_id", "output_key"])`. This is a **reference by selector path**, not typed pipe.

### 6.2 Typing and Validation

`VariablePool.add()` enforces that selectors must have exactly 2 elements. Values are auto-cast to the appropriate `Segment` type (string, number, object, file, array).

**There is no Pydantic schema validation between nodes**. A node's output is placed into `VariablePool` as raw values. Type mismatch surfaces at runtime.

**Contrast**:
- Mastra's Workflow steps have `inputSchema` / `outputSchema` (Zod) that are checked at step boundaries
- LangGraph has typed `StateT` dict with channel reducers
- **Dify's `VariablePool` is weakly typed** — closer to LangGraph's `dict[str, Any]` than to Mastra's per-step schemas

---

## 7. Dify 2.0 Queue Scheduling Engine

The queue-based `GraphEngine` architecture (GitHub Issue #24067, released in Dify ~1.9.0) replaced the old monolithic single-threaded graph runner.

**What changed**:
- Old: single-threaded traversal, tight coupling, hard to test
- New: `GraphEngine` with a ready queue + dynamic worker pool + command channel

**Is it Celery?** **No.** The "queue" is an in-process concept: the `ready_queue` holds node IDs ready for execution. Workers are Python threads in the same process. Redis is only for command signaling (`RedisChannel`), which allows external HTTP requests to send abort signals.

**Is it same as Mastra's DAG?** Structurally similar conceptually but mechanically different:
- Mastra's Workflow is a declarative fluent API (`.then().parallel().foreach().dowhile()`) compiled to a step chain
- Graphon's Graph is a JSON config (`{nodes: [...], edges: [...]}`) interpreted at runtime by the engine

**Layer system**: `GraphEngineLayer` hooks (`on_node_run_start`, `on_node_run_end`) provide cross-cutting concerns: persistence, LLM quota enforcement, observability, debug logging.

---

## 8. Direct Comparison with Mastra / Google ADK / LangGraph

### 8.1 Execution Paradigm

| Framework | Paradigm | Loop unit | Concurrency |
|-----------|---------|-----------|------------|
| **Dify (graphon)** | Queue-based worker pool DAG | Node | Thread pool, configurable |
| **Mastra** | Fluent DAG (`.then/.parallel/.foreach/.dowhile`) | Step | Async/await |
| **Google ADK** | Async generator DAG | Agent | `asyncio` |
| **LangGraph** | Pregel BSP (`while loop.tick()`) | Node per superstep | Per-tick parallel dispatch |

All four are graph-based and support DAG parallelism, but the **execution models differ meaningfully**:
- LangGraph's Pregel has strict BSP phases with cross-step write visibility guarantees
- Dify/graphon uses a ready-queue with concurrent workers — writes land in `VariablePool` immediately
- Mastra's Workflow composes a step chain at code time
- Google ADK composes an agent tree at code time

### 8.2 The "Mastra = TypeScript Dify" Claim — Verdict

**At the conceptual/product level: broadly supported.**
Both are workflow-first platforms where agents are typed nodes in a DAG. Both support: start node, LLM node, conditional branching, loop/iteration, code execution, tool invocation, and an agent node that runs a ReAct-style loop. Both target developers who want to build multi-step AI applications.

**At the source-level execution model: partially different, not identical.**

Key differences:

1. **Loop implementation**: Mastra's ReAct loop is a `.dowhile(agenticExecutionWorkflow, condition)` over composable steps — a first-class workflow primitive. Dify's agent loop is either a Python `while` loop in `FunctionCallAgentRunner`/`CotAgentRunner` (for `agent_chat`), or **delegated entirely to a plugin daemon** (for workflow `AgentNode`). Dify has no equivalent of Mastra's `dowhile` as an engine primitive.

2. **Graph execution**: Dify's graphon is a **concurrent queue-based worker pool**. Mastra's Workflow is a sequential/parallel/foreach step chain evaluated linearly (async). Graphon is more infrastructure-heavy — better suited for multi-tenant server deployment. Mastra's Workflow is a simpler in-process step runner.

3. **State typing**: Mastra uses Zod schemas per step boundary. Dify uses a weakly-typed `VariablePool` with selector-based access. Mastra validates at step boundaries at runtime; Dify does not.

4. **Agent architecture (Dify 2.x)**: The workflow `AgentNode` is now a **plugin dispatch boundary**, not a self-contained loop. The ReAct loop lives in a separate plugin process, and different agent strategies (ReAct, function calling, custom) are installable extensions. **Mastra has no equivalent plugin system** — strategies are hardcoded.

**Verdict**: "Mastra = TypeScript Dify" is a **useful analogy but not a precise architectural correspondence**. The product-level logic is the same (build AI workflows as DAGs of typed nodes, with agent nodes that run tool loops). The execution engine details diverge:
- Mastra: TypeScript step-chain runner
- Dify: Python concurrent queue engine with plugin-extensible agent strategies

**A more accurate framing**: **Dify and Mastra are both workflow-first agent platforms in the same product category, but built on different execution philosophies**.

---

## 9. Position on the Agent Architecture Spectrum

**Dify is a Visual Workflow Platform with pluggable agent strategies.** It is a distinct category from pure code frameworks.

Compared to others:
- **vs LangGraph**: LangGraph is a pure code framework (graph-as-code, Pregel engine), no visual editor, no multi-tenant server. Dify provides a visual editor, multi-tenant deployment, plugin ecosystem. LangGraph's BSP model gives stronger consistency guarantees; Dify's queue model scales better.
- **vs Google ADK**: ADK is a pure Python code framework (agent composition in code), no visual editor, no plugin store. Dify has the plugin daemon architecture; ADK has none.
- **vs Mastra**: Mastra is a TypeScript code framework with optional visual editor (`mastra dev`). Closest to Dify in product concept. Mastra is simpler (no plugin daemon, no Redis command channel, no worker pool). Dify is more production-hardened infrastructure.

Dify is the only one in this analysis with:
- A visual workflow editor (no-code/low-code)
- A plugin daemon for extensible agent strategies
- A multi-tenant production server architecture
- Redis-backed command channels for distributed workflow control

---

## Key Architectural Insights

1. **Graphon is the actual engine.** The `api/core/workflow/` directory is a Dify-specific orchestration layer on top of the external `graphon` package. Understanding graphon's queue-based worker pool is essential.

2. **Two completely separate agent implementations coexist.** The `api/core/agent/` path (FC + CoT runners) serves only the `agent_chat` app type, with the loop running as a Python `while` loop. The `api/core/workflow/nodes/agent/` path serves the workflow `AgentNode`, which delegates to a plugin daemon. These share no code.

3. **The Agent node in Dify 2.x is a plugin RPC boundary.** The actual ReAct loop runs in the plugin daemon process. This enables community-contributed agent strategies without touching core code — but it means the agent loop is not inspectable from the main codebase.

4. **`VariablePool` is the shared state bus.** All inter-node data flows through a two-level dictionary. There is no typed pipe between nodes. Compared to Mastra's per-step Zod schemas or LangGraph's typed channels, `VariablePool` is the weakest form of type safety.

5. **Redis is for signals, not task distribution.** The `RedisChannel` command channel enables external abort/pause signals. The node execution itself is threaded within a single process. This is frequently mistaken for Celery-style distributed task queues — it is not.

6. **Layer system is graphon's middleware.** `GraphEngineLayer` hooks compose cross-cutting behavior: DB persistence, LLM quota, OTel tracing, execution limits.

---

## Conclusion

Dify is a **visual, multi-tenant, plugin-extensible workflow platform** where the execution engine is a **queue-based concurrent worker pool** (graphon) over a JSON-defined DAG of typed nodes. It is not a code framework — it is a deployed server product with a UI.

**The "Mastra = TypeScript Dify" analogy is accurate at the product category level** (workflow-first, agent-as-node, typed step graph) **but imprecise at the execution engine level**:
- Mastra: TypeScript step-chain, in-process, `.dowhile()` as engine primitive, Zod-typed boundaries
- Dify: Python concurrent queue engine, plugin daemon for agent strategies, Redis command channel, weakly-typed `VariablePool`

**Better framing**: "Mastra and Dify are both workflow-first agent platforms that treat agents as typed nodes in an executable DAG. Their execution engines differ in language, concurrency model, and extensibility mechanism, but their product philosophy and developer mental model are the same."

On the Agent Map taxonomy:
- Dify sits firmly in **② Workflow Platform** alongside Coze/扣子, Tencent Yuanqi, FastGPT, Baidu AppBuilder, Alibaba Bailian Visual Studio
- It is **not** an ③ Code Framework — the visual editor and plugin daemon make it a different product class
- It is one of the few open-source representatives of ② that also has genuine production-grade infrastructure (queue engine, Redis control channel, multi-tenant layering)

**Essential Files for Understanding Dify's Agent Architecture:**

- `api/core/workflow/workflow_entry.py` — engine assembly point
- `api/core/workflow/node_factory.py` — node registry and dependency injection
- `api/core/workflow/nodes/agent/agent_node.py` — Agent-as-node (plugin dispatch)
- `api/core/workflow/nodes/agent/strategy_protocols.py` — plugin strategy Protocol
- `api/core/agent/fc_agent_runner.py` — FC ReAct loop (agent_chat)
- `api/core/agent/cot_agent_runner.py` — CoT ReAct loop (agent_chat)
- `api/core/agent/entities.py` — Strategy enum (FC vs CoT)
- `api/core/agent/prompt/template.py` — literal ReAct prompt
- `api/core/app/apps/agent_chat/app_runner.py` — FC vs CoT selection
- `api/core/app/apps/workflow/app_runner.py` — WorkflowAppRunner (Redis channel + WorkflowEntry)

**External dependency that is the true execution engine:**
- `github.com/langgenius/graphon` — `GraphEngine`, `Graph`, `VariablePool`, `GraphRuntimeState`, all node base classes
