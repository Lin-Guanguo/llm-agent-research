# Google ADK Agent Architecture Research

Last Updated: 2026-04-12

> **Research Methodology**: Source-level analysis of google/adk-python at `/Users/linguanguo/dev/llm-agent-research/google-adk` (local clone of github.com/google/adk-python). This replaces the earlier README-level version. Every architectural claim cites a specific file and line number from the source.

## Sources

**Core Agent Files**
- `src/google/adk/agents/base_agent.py:86-701` — `BaseAgent`: `run_async()`, callback system
- `src/google/adk/agents/llm_agent.py:195-1015` — `LlmAgent`: fields, `_run_async_impl`, `_llm_flow` property
- `src/google/adk/agents/loop_agent.py:52-166` — `LoopAgent._run_async_impl`
- `src/google/adk/agents/sequential_agent.py:48-159` — `SequentialAgent._run_async_impl`
- `src/google/adk/agents/parallel_agent.py:150-216` — `ParallelAgent._run_async_impl`
- `src/google/adk/agents/langgraph_agent.py:52-143` — `LangGraphAgent` (concept wrapper)

**Execution Loop**
- `src/google/adk/flows/llm_flows/base_llm_flow.py:797-1230` — `BaseLlmFlow.run_async()`, `_run_one_step_async`, tool dispatch
- `src/google/adk/flows/llm_flows/single_flow.py:78-88` — `SingleFlow`: assembles request/response processor list
- `src/google/adk/flows/llm_flows/auto_flow.py:23-44` — `AutoFlow`: adds agent transfer processor
- `src/google/adk/flows/llm_flows/functions.py:355-433` — `handle_function_calls_async`: parallel tool execution

**Planner Files**
- `src/google/adk/planners/base_planner.py:29-68` — `BasePlanner` ABC
- `src/google/adk/planners/plan_re_act_planner.py:35-210` — `PlanReActPlanner`
- `src/google/adk/planners/built_in_planner.py:32-86` — `BuiltInPlanner` (thinking config)
- `src/google/adk/flows/llm_flows/_nl_planning.py:39-133` — Planning processors in flow

**Multi-Agent**
- `src/google/adk/flows/llm_flows/agent_transfer.py:37-175` — `transfer_to_agent` tool injection
- `src/google/adk/tools/transfer_to_agent_tool.py:26-89` — `TransferToAgentTool`
- `src/google/adk/tools/agent_tool.py:94-304` — `AgentTool`: wrap agent as tool

**Events and State**
- `src/google/adk/events/event.py:31-130` — `Event` class
- `src/google/adk/events/event_actions.py:51-115` — `EventActions`
- `src/google/adk/runners.py:503-633` — `Runner.run_async()`: entry point

**Examples**
- `contributing/samples/workflow_agent_seq/agent.py` — `SequentialAgent` pipeline
- `contributing/samples/workflow_triage/agent.py` + `execution_agent.py` — LLM triage pattern

---

## 1. Agent Definition (API Layer)

`LlmAgent` (aliased as `Agent`) is a Pydantic `BaseModel` at `src/google/adk/agents/llm_agent.py:195`. It inherits from `BaseAgent` (`base_agent.py:86`), which provides `run_async()`, `run_live()`, and the callback framework.

Key fields on `LlmAgent` (lines 204-464):

```python
model: Union[str, BaseLlm] = ''        # Default: gemini-2.5-flash
instruction: Union[str, InstructionProvider] = ''
tools: list[ToolUnion] = []
input_schema: Optional[type[BaseModel]] = None   # For AgentTool wrapping
output_schema: Optional[SchemaType] = None       # Constrains final response
output_key: Optional[str] = None        # Save final text to session.state[key]
planner: Optional[BasePlanner] = None   # Optional planning prompt modifier
disallow_transfer_to_parent: bool = False
disallow_transfer_to_peers: bool = False
```

The `_llm_flow` property at `llm_agent.py:717-725` decides whether agent transfer is enabled:

```python
@property
def _llm_flow(self) -> BaseLlmFlow:
    if (self.disallow_transfer_to_parent
            and self.disallow_transfer_to_peers
            and not self.sub_agents):
        return SingleFlow()   # No transfer capability
    else:
        return AutoFlow()     # Adds transfer_to_agent tool to LLM
```

`_run_async_impl` at `llm_agent.py:467-504` delegates to `self._llm_flow.run_async(ctx)` and handles the resumption logic for multi-turn sessions.

---

## 2. Execution Model: Pure ReAct Loop

**The source confirms: Google ADK is a pure ReAct loop.** The main loop lives in `src/google/adk/flows/llm_flows/base_llm_flow.py:797-810`:

```python
async def run_async(self, invocation_context):
    """Runs the flow."""
    while True:
        last_event = None
        async with Aclosing(self._run_one_step_async(invocation_context)) as agen:
            async for event in agen:
                last_event = event
                yield event
        if not last_event or last_event.is_final_response() or last_event.partial:
            break
```

This is the canonical ReAct `while True` loop. One complete iteration equals one LLM call. The loop breaks only when `last_event.is_final_response()` — which is true when the LLM returns a response with no pending function calls.

`_run_one_step_async` at `base_llm_flow.py:812-895` is one ReAct step:

1. Build `LlmRequest` via `_preprocess_async` (instructions, tools, planner prompts)
2. Call the LLM via `_call_llm_async`
3. For each `LlmResponse`, run `_postprocess_async`: yield the response event; if function calls are present, execute them via `_postprocess_handle_function_calls_async`

**Tool execution** at `functions.py:394-410` runs all function calls in one turn **in parallel** via `asyncio.create_task` + `asyncio.gather`.

**Loop termination** relies entirely on the LLM returning a response without function calls (`is_final_response()`). There is no `maxSteps` parameter on `LlmAgent`. The loop is theoretically unbounded. Only `LoopAgent` has explicit `max_iterations`.

**There is no Plan artifact, no Plan class, no plan executor.** Searching the entire codebase finds zero classes named `Plan`, `PlanStep`, `ExecutionPlan`, or `PlanExecutor`. The only "plan" word appears in `PlanReActPlanner` — analyzed in section 5 below.

---

## 3. Workflow Concept

ADK's workflow primitives are pure Python control-flow constructs with no LLM involvement in orchestration decisions.

### SequentialAgent (`sequential_agent.py:54-92`)

```python
for i in range(start_index, len(self.sub_agents)):
    sub_agent = self.sub_agents[i]
    async with Aclosing(sub_agent.run_async(ctx)) as agen:
        async for event in agen:
            yield event
```

A Python for-loop. Execution order is fixed at construction. Sub-agents communicate via `session.state` — agent A sets `output_key="foo"`, agent B reads `{foo}` via instruction template injection.

### LoopAgent (`loop_agent.py:82-115`)

```python
while (
    not self.max_iterations or times_looped < self.max_iterations
) and not (should_exit or pause_invocation):
    for i in range(start_index, len(self.sub_agents)):
        async for event in sub_agent.run_async(ctx):
            yield event
            if event.actions.escalate:
                should_exit = True
```

Outer Python while-loop, bounded by `max_iterations` or `event.actions.escalate = True`. The escalation signal must be emitted by one of the sub-agents by setting `tool_context.actions.escalate = True` in a tool call. No LLM is consulted for the loop control decision. Each sub-agent inside runs its own independent ReAct loop.

### ParallelAgent (`parallel_agent.py:164-205`)

Runs all sub-agents concurrently via `asyncio.TaskGroup` (Python 3.11+) or manual task creation. Each sub-agent gets an isolated `InvocationContext.branch` to prevent cross-contamination of event history.

**These three are flow-control primitives, not a Plan executor.** They are the Python equivalent of sequential/parallel composition. Compared to Mastra's Workflow DAG, ADK's three fixed types are less general (no `.branch()`, no arbitrary `.then()` chains of heterogeneous steps) but simpler to understand.

---

## 4. Multi-Agent Composition and Handoff

ADK has two distinct multi-agent patterns with very different observability properties.

### Pattern A: `transfer_to_agent` (Context-Preserving)

When `AutoFlow` is active, `_AgentTransferLlmRequestProcessor` (`agent_transfer.py:37-72`) injects a `TransferToAgentTool` into the LLM request. A system prompt instruction (`agent_transfer.py:109-126`) tells the LLM when to transfer:

> "If another agent is better for answering the question according to its description, call `transfer_to_agent` function to transfer the question to that agent."

The LLM calls the tool with `agent_name`. The tool implementation at `transfer_to_agent_tool.py:26-40`:

```python
def transfer_to_agent(agent_name: str, tool_context: ToolContext) -> None:
    tool_context.actions.transfer_to_agent = agent_name
```

After tool execution, `base_llm_flow.py:1140-1147` detects the transfer:

```python
transfer_to_agent = function_response_event.actions.transfer_to_agent
if transfer_to_agent:
    agent_to_run = self._get_agent_to_run(invocation_context, transfer_to_agent)
    async with Aclosing(agent_to_run.run_async(invocation_context)) as agen:
        async for event in agen:
            yield event
```

The child agent receives the **same `invocation_context`** — it sees the parent's full conversation history. All child events are yielded to the parent's caller. The next user turn routes back to `LlmAgent._get_subagent_to_resume` (`llm_agent.py:727-770`), which finds the last transfer target by scanning the event history.

**Trace fidelity**: full. The child's intermediate reasoning and tool calls are visible in the session events.

### Pattern B: `AgentTool` (Isolated, Final-Text Only)

`src/google/adk/tools/agent_tool.py:192-286` wraps any `BaseAgent` as a callable tool:

```python
async def run_async(self, *, args, tool_context):
    runner = Runner(
        agent=self.agent,
        session_service=InMemorySessionService(),   # Brand-new isolated session
        ...
    )
    last_content = None
    async for event in runner.run_async(...):
        if event.actions.state_delta:
            tool_context.state.update(event.actions.state_delta)
        if event.content:
            last_content = event.content    # Only last content kept
    return merged_text   # Only final text returned; intermediate steps discarded
```

The sub-agent runs in a completely isolated session. All intermediate tool calls, reasoning steps, and partial responses from the sub-agent are discarded. Only the final text (and `state_delta`) is returned to the parent.

This is the exact failure mode described in Cognition's "Don't Build Multi-Agents": the parent loses the child's reasoning trace. ADK ships this pattern as `AgentTool`.

**ADK offers both patterns.** The documentation recommends `transfer_to_agent` for routing to specialist agents, and `AgentTool` for tool-like sub-tasks where only the result matters.

---

## 5. Control and Predictability Mechanisms

### The Planner System: Looks Like P&E, Is Not

`LlmAgent` has a `planner: Optional[BasePlanner]` field (`llm_agent.py:355-361`). Two concrete implementations exist.

**`PlanReActPlanner`** (`planners/plan_re_act_planner.py:35-210`) is the most interesting. Its `_build_nl_planner_instruction` method constructs a system prompt (lines 153-210):

> "Follow this process when answering the question:
> (1) first come up with a plan in natural language text format;
> (2) Then use tools to execute the plan...
>
> The planning part should be under `/*PLANNING*/`.
> The tool code snippets should be under `/*ACTION*/`.
> The final answer part should be under `/*FINAL_ANSWER*/`."

This is a **prompt engineering convention**. The `_NlPlanningRequestProcessor` at `_nl_planning.py:39-65` appends this instruction to the `LlmRequest` before each LLM call. The `_NlPlanningResponse` processor at `_nl_planning.py:69-103` parses the response, marks `/*PLANNING*/` and `/*REASONING*/` parts as `thought=True` (hidden), and keeps the `/*FINAL_ANSWER*/` part visible.

**The key structural point**: this runs inside the existing `while True` ReAct loop. There is no separate "plan phase" before the loop. The planner does not produce a `Plan` object that the executor then processes step by step. The LLM produces a plan-formatted text and then proceeds with tool calls in subsequent iterations of the same loop. Re-planning (via `/*REPLANNING*/`) is handled by the LLM naturally as part of a new loop iteration.

**`BuiltInPlanner`** (`planners/built_in_planner.py:57-70`) sets `llm_request.config.thinking_config` — it enables Gemini's built-in chain-of-thought. The thinking tokens are hidden. This is not planning in the P&E sense; it is model-internal reasoning that happens within a single LLM call.

### Callbacks as Interceptors

`before_model_callback` / `after_model_callback` / `before_tool_callback` / `after_tool_callback` (`llm_agent.py:374-463`) intercept the ReAct loop at specific points. If `before_model_callback` returns a non-None `LlmResponse`, the LLM call is skipped and that response is used directly. This is ADK's primary human-in-the-loop mechanism — the callback can prompt for human approval before proceeding.

### Session State as Event Bus

State changes flow through `Event.actions.state_delta` — a `dict[str, object]` attached to any event. The `Runner` applies `state_delta` to `session.state` as events are processed. Partial (streaming) events do **not** trigger state updates; only complete events do. This atomicity prevents partial tool results from polluting state.

---

## 6. Typing and Validation

ADK uses Pydantic `BaseModel` for all framework scaffolding. Validation exists at these boundaries:

1. **Agent construction**: `name` validated as Python identifier (`base_agent.py:556-570`); `generate_content_config` validated to not include `tools` or `system_instruction` (those must go through dedicated fields, `llm_agent.py:865-882`)
2. **Agent I/O when used as `AgentTool`**: `input_schema: type[BaseModel]` defines the tool parameters; `output_schema` is validated via `validate_schema()` on the final text (`llm_agent.py:858`)
3. **Tool inputs**: Python function type hints converted to JSON Schema for the LLM's function declaration
4. **Event structure**: all `Event`, `EventActions`, `LlmRequest`, `LlmResponse` are Pydantic models

**What is not validated**:
- Data flowing through `session.state` between agents — untyped `dict[str, object]`
- `output_key` / `{variable_name}` template binding — pure string interpolation, no schema check
- Sequential pipeline step-to-step contracts — agent A's `output_key="foo"` is read by agent B as raw string; no schema enforces the structure

**Comparison to Mastra's Zod**: Mastra validates `outputSchema` / `inputSchema` on every Workflow step. ADK validates at agent-level I/O only when wrapped as `AgentTool`. For `SequentialAgent` pipelines, there is no equivalent of Mastra's step boundary validation — just untyped string passing through state.

---

## 7. Direct Comparison with Mastra

| Dimension | Google ADK | Mastra |
|---|---|---|
| **Core loop** | `while True: _run_one_step_async` (`base_llm_flow.py:797`) | `.dowhile(agenticExecutionWorkflow)` (`agentic-loop/index.ts:72`) |
| **Loop termination** | `last_event.is_final_response()` — LLM must stop itself | `stepResult.isContinued = false` OR `stopWhen(stepCountIs(5))` default |
| **No-plan-artifact proof** | No `Plan`, `PlanStep`, `ExecutionPlan` class anywhere in codebase | No "Plan" message type, no separate planning phase |
| **Tool execution** | `asyncio.gather` (parallel) per LLM turn (`functions.py:394`) | `.foreach(toolCallStep, { concurrency })` per iteration |
| **Workflow orchestration** | 3 fixed primitives: Sequential / Parallel / Loop | General DAG: `.then()`, `.parallel()`, `.foreach()`, `.dowhile()`, `.branch()` |
| **Step-to-step typing** | Untyped `session.state` dict + string template injection | Zod `inputSchema` / `outputSchema` on each Step |
| **Multi-agent handoff** | Two modes: `transfer_to_agent` (full context) + `AgentTool` (text only) | One mode: sub-agent via tool call (final result only) |
| **Max iterations guard** | Only on `LoopAgent`; `LlmAgent` has no default limit | `stopWhen: stepCountIs(5)` default on every agent |
| **Human-in-the-loop** | `before_model_callback` + `before_tool_callback` | `.suspend()` / `.resume()` on workflow steps |
| **Language** | Python | TypeScript |

**ADK's `transfer_to_agent` is better than Mastra's model**: it preserves the full trace and shares context. Mastra's only option (sub-agent via tool) discards intermediate reasoning. ADK's `AgentTool` does the same as Mastra, but ADK at least offers the better alternative.

**The loops are structurally equivalent**: `while True: _run_one_step_async` vs. `.dowhile(agenticExecutionWorkflow)`. Different syntax, same semantic. The primary behavioral difference is that Mastra enforces a default 5-step limit while ADK relies on the LLM's own stop signal.

---

## 8. Representative Examples

### Example 1: Sequential Code Pipeline

From `contributing/samples/workflow_agent_seq/agent.py` — three agents in a pipeline:

```python
code_writer_agent = LlmAgent(
    name="CodeWriterAgent",
    instruction="Write Python code...",
    output_key="generated_code",        # saved to session.state
)
code_reviewer_agent = LlmAgent(
    name="CodeReviewerAgent",
    instruction="Review:\n```python\n{generated_code}\n```",  # read from state
    output_key="review_comments",
)
code_refactorer_agent = LlmAgent(
    name="CodeRefactorerAgent",
    instruction="Refactor {generated_code} based on {review_comments}",
)
root_agent = SequentialAgent(
    name="CodePipelineAgent",
    sub_agents=[code_writer_agent, code_reviewer_agent, code_refactorer_agent],
)
```

Data flow is `session.state` via `output_key` and `{variable_name}` template injection in instructions. The connection is string interpolation, not typed.

### Example 2: LLM Triage + Parallel Execution

From `contributing/samples/workflow_triage/` — the LLM decides which workers to run, Python orchestration executes them:

```python
# LLM decides which workers are needed
root_agent = Agent(
    instruction="Analyze input. Call update_execution_plan with relevant workers.",
    tools=[update_execution_plan],   # stores ["code_agent", "math_agent"] to state
    sub_agents=[plan_execution_agent],
)

# Python orchestration executes in parallel, then summarizes
plan_execution_agent = SequentialAgent(sub_agents=[
    ParallelAgent(sub_agents=[code_agent, math_agent]),  # parallel execution
    execution_summary_agent,                              # reads outputs from state
])
```

Workers use `before_agent_callback` to skip if not in `execution_agents` state list. This pattern separates the "what to do" (LLM triage) from the "how to execute" (Python orchestration) — but the "plan" here is just a string list in state, not a structured typed plan artifact.

---

## 9. Position on the Agent Architecture Spectrum

**Google ADK is ④a. It is not ④b. Four source-code proofs:**

**Proof 1 — Main loop is ReAct** (`base_llm_flow.py:797-810`): `while True: LLM call → tool calls → repeat`. There is no pre-execution planning phase before this loop, no plan generation step before iteration begins.

**Proof 2 — `PlanReActPlanner` is prompt engineering** (`plan_re_act_planner.py:153-210`): the planner injects a system-prompt instruction asking the LLM to format its output with `/*PLANNING*/` tags. It runs as a `BaseLlmRequestProcessor` inside `SingleFlow`'s processor list (`_nl_planning.py:39-65`) — the same processor list that handles instructions and tool declarations. It modifies the LLM's response format, not the execution architecture.

**Proof 3 — Workflow agents are Python control flow** (`sequential_agent.py:67-88`, `loop_agent.py:82-115`): `SequentialAgent` is a Python for-loop, `LoopAgent` is a Python while-loop. No LLM is consulted for orchestration decisions. These are composition primitives, not plan executors.

**Proof 4 — No structured Plan type exists**: a codebase-wide search finds zero classes named `Plan`, `PlanStep`, `ExecutionPlan`, or `PlanExecutor`. The `workflow_triage` example stores a `list[str]` of agent names in `session.state` as its "plan" — an untyped string list is not a structured plan.

**On the LangGraphAgent curiosity**: `langgraph_agent.py:52` describes itself as "Currently a concept implementation." It wraps a compiled LangGraph `CompiledGraph` as a `BaseAgent` and calls `graph.invoke()` synchronously. This is an adapter/integration, not a signal about ADK's own execution model. It is not exported from `agents/__init__.py` and is not part of the standard agent suite.

---

## Key Architectural Insights

1. **Two execution layers**: ADK's architecture is clean at the layer boundary. Inner layer: `BaseLlmFlow.run_async()` — the ReAct loop for one LLM agent. Outer layer: `SequentialAgent` / `LoopAgent` / `ParallelAgent` — Python orchestration composing multiple inner loops. Mastra has the same two layers (agentic-loop inner / Workflow DAG outer), with ADK's outer layer being less general but structurally equivalent.

2. **`transfer_to_agent` vs. `AgentTool` is a significant design choice**: ADK offers both context-preserving handoff (`transfer_to_agent`) and context-discarding delegation (`AgentTool`). Mastra only offers the context-discarding pattern. The existence of `transfer_to_agent` in ADK is architecturally superior for observability but is LLM-driven routing, not structured planning.

3. **`PlanReActPlanner` is the closest ADK gets to ④b and does not cross**: it enforces a planning-first structure within single LLM responses, enables re-planning, and marks planning text as non-visible. But the same `while True` ReAct loop runs before and after enabling this planner. The planner is an add-on, not an architectural foundation.

4. **Safety gap: no default `maxSteps` on `LlmAgent`**: `LlmAgent` has no built-in step limit. Mastra defaults to `stepCountIs(5)`. ADK relies entirely on the LLM emitting a final response. This is a production reliability concern — a misbehaving agent can loop indefinitely unless `LoopAgent.max_iterations` wraps it.

5. **Pydantic validates structure, not dataflow**: ADK's use of Pydantic is thorough for framework objects (`Agent`, `Event`, `EventActions`, `LlmRequest`) but absent at the `session.state` level where runtime data flows. The gap between the well-typed scaffolding and the untyped state bus is the key structural weakness for production reliability.

6. **`LangGraphAgent` signals integration intent**: Google intends ADK to be an integration hub. The existence of a LangGraph adapter suggests ADK sees LangGraph as a peer in the ecosystem, not a competitor's architecture to copy. ADK's own execution model remains independent.

---

## Conclusion

Google ADK is a well-engineered, production-quality ReAct framework. Its execution model is `while True: LLM call → tools → repeat`, with Python orchestration primitives (`SequentialAgent`, `LoopAgent`, `ParallelAgent`) composing multiple agents at the outer level. The `PlanReActPlanner` adds planning-flavored prompt engineering but does not change the execution architecture.

**Google ADK is ④a (Code Framework, ReAct school), not ④b.**

ADK's `transfer_to_agent` mechanism is architecturally better than Mastra's tool-based handoff for multi-agent observability. Its `PlanReActPlanner` is more sophisticated than anything Mastra offers for planning-like behavior. But neither feature crosses the line into a true Plan-and-Execute architecture with a typed plan artifact, structural phase separation, or pre-execution cost estimation.

The ④b category (P&E code framework) remains empty in the public ecosystem. Google ADK confirms, at source level, that the gap between "the best public framework" and "a true P&E + typed dataflow production system" is architectural, not just cosmetic.
