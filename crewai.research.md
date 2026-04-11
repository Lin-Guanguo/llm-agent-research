# CrewAI Agent Architecture Research

Last Updated: 2026-04-12

> **Research Methodology**: Source-level analysis of crewAIInc/crewAI at `/Users/linguanguo/dev/llm-agent-research/crewai/lib/crewai/src/crewai`. The central question: is CrewAI ④a (ReAct), ④b (Plan-and-Execute), or something else? **Major finding: CrewAI has a genuine ④b implementation in `src/crewai/experimental/agent_executor.py` that extends `Flow[AgentExecutorState]` and cites arxiv 2503.09572 (Plan-and-Act).**

## Sources

**Core Abstractions**
- `src/crewai/crew.py:158-419` — Crew class definition
- `src/crewai/crew.py:877-965` — `kickoff()` entry point
- `src/crewai/crew.py:1375-1408` — Sequential and hierarchical process execution
- `src/crewai/crew.py:1419-1486` — `_execute_tasks()`
- `src/crewai/crew.py:1577-1580` — `_get_agent_to_use()` hierarchical routing
- `src/crewai/crew.py:1733-1742` — `_get_context()` inter-task data passing
- `src/crewai/process.py` — Process enum (2 values: sequential, hierarchical)
- `src/crewai/task.py:88-200` — Task class definition
- `src/crewai/agent/core.py:148-358` — Agent class
- `src/crewai/agents/agent_builder/base_agent.py:162-280` — BaseAgent abstract class

**Execution Loop (default)**
- `src/crewai/agents/crew_agent_executor.py:290-456` — `_invoke_loop_react()` and `_invoke_loop_native_tools()`
- `src/crewai/agents/parser.py:1-80` — ReAct output parser
- `src/crewai/agents/constants.py` — FINAL_ANSWER_ACTION, ACTION_REGEX

**Plan Features (experimental ④b)**
- `src/crewai/utilities/planning_handler.py` — CrewPlanner (crew-level planning, prompt augmentation)
- `src/crewai/agent/planning_config.py:78-137` — PlanningConfig: per-agent planning
- `src/crewai/utilities/planning_types.py` — **`PlanStep`, `TodoItem`, `TodoList`, `StepObservation`**
- `src/crewai/experimental/agent_executor.py:118-342` — **`AgentExecutor` (experimental P&E executor)**
- `src/crewai/agents/step_executor.py:60-100` — `StepExecutor`: single-step isolated execution
- `src/crewai/agents/planner_observer.py:39-80` — `PlannerObserver`: post-step observation

**Handoff / Delegation**
- `src/crewai/tools/agent_tools/agent_tools.py` — AgentTools: DelegateWorkTool + AskQuestionTool
- `src/crewai/tools/agent_tools/base_agent_tools.py:48-130` — `_execute()`

**Flow Module**
- `src/crewai/flow/flow.py:175-461` — `@start`, `@listen`, `@router` decorators
- `src/crewai/flow/flow.py:892-990` — Flow class

**Prompts**
- `src/crewai/translations/en.json` — All prompt templates

---

## 1. The Crew / Agent / Task Trinity

### 1.1 Crew class

`Crew` (`crew.py:158`) is a Pydantic `BaseModel`. Essential fields:

```python
tasks: list[Task] = Field(default_factory=list)
agents: list[BaseAgent] = Field(default_factory=list)
process: Process = Field(default=Process.sequential)
manager_agent: BaseAgent | None = Field(default=None)
manager_llm: str | BaseLLM | None = Field(default=None)
planning: bool | None = Field(default=False)
planning_llm: str | BaseLLM | None = Field(default=None)
```

The Crew is a container plus orchestrator. It holds a pre-defined list of `tasks` and `agents` and runs them via `kickoff()`. **The task list is user-defined at construction time** — the Crew does not discover or generate tasks dynamically.

### 1.2 Agent class (and the `agent/` vs `agents/` directory split)

- `src/crewai/agent/` — the **public-facing Agent class**. `agent/core.py:148` contains `class Agent(BaseAgent)`. Also holds `planning_config.py`.
- `src/crewai/agents/` — **internal execution infrastructure**. Contains `crew_agent_executor.py` (ReAct loop), `step_executor.py` (P&E step runner), `planner_observer.py`, `parser.py`.

`BaseAgent` (`agents/agent_builder/base_agent.py:162`) defines: `role`, `goal`, `backstory`, `llm`, `tools`, `max_iter`, `allow_delegation`. The identity trio is injected into every system prompt via `en.json:11`:

```
"You are {role}. {backstory}\nYour personal goal is: {goal}"
```

This is the "CEO / CTO / Analyst" flavor. **`role` is a prompt construct, not a capability boundary.**

### 1.3 Task class

```python
class Task:
    description: str                            # What to do
    expected_output: str                        # Success criteria
    agent: BaseAgent | None                     # Who executes it
    context: list[Task] | None                  # Upstream outputs to inject
    tools: list[BaseTool] | None
```

The `context` field is the inter-task pipe. When unspecified, the Crew automatically feeds the prior task's raw text output. When explicitly set to a list of Tasks, the Crew collects their `.raw` string outputs.

### 1.4 User-defined vs LLM-generated Tasks

**Tasks are always user-defined.** There is no code path in which a Crew generates its own task list. The user writes all tasks before calling `kickoff()`. **This is the key fact driving the classification.**

---

## 2. Process Types

### 2.1 The Process enum

`process.py` is 12 lines:

```python
class Process(str, Enum):
    sequential = "sequential"
    hierarchical = "hierarchical"
    # TODO: consensual = 'consensual'
```

**Two modes.** `consensual` is a TODO comment. There is no `plan`, `goal-directed`, or `autonomous` mode at the Process level.

### 2.2 Sequential process

`crew.py:1375-1377`:
```python
def _run_sequential_process(self) -> CrewOutput:
    return self._execute_tasks(self.tasks)
```

`_execute_tasks()` iterates `tasks` in list order. For each task: `_get_agent_to_use(task)` returns `task.agent` — the agent pre-assigned by the user. Context passed to each task is the concatenated raw text outputs of all prior tasks (`_get_context()`, `crew.py:1733-1742`).

**The sequential process is a static DAG.** The user defines N tasks, each bound to an agent, and they run in order with text piped between them.

### 2.3 Hierarchical process

`crew.py:1379-1382`:
```python
def _run_hierarchical_process(self) -> CrewOutput:
    self._create_manager_agent()
    return self._execute_tasks(self.tasks)
```

`_create_manager_agent()` creates a manager using the prompt from `en.json:2-6`:

```json
"hierarchical_manager_agent": {
  "role": "Crew Manager",
  "goal": "Manage the team to complete the task in the best way possible.",
  "backstory": "You are a seasoned manager with a knack for getting the best out of your team..."
}
```

The manager is given `AgentTools(agents=self.agents).tools()` — two delegation tools: `DelegateWorkTool` and `AskQuestionTool`.

Then `_get_agent_to_use()` (`crew.py:1577-1580`):
```python
def _get_agent_to_use(self, task: Task) -> BaseAgent | None:
    if self.process == Process.hierarchical:
        return self.manager_agent
    return task.agent
```

**Every task in hierarchical mode is routed through the manager.** The user-defined task descriptions serve as prompts to the manager. The manager decides whether to delegate to a worker or answer directly. **The task list is still user-defined and fixed** — what changes is that the manager handles each task and decides whether to sub-delegate.

---

## 3. Agent Execution Model

### 3.1 Default executor: ReAct

The default executor is `CrewAgentExecutor`. `_invoke_loop()` (`crew_agent_executor.py:290-311`) first checks for native function calling, then falls back to text-based ReAct:

```python
if use_native_tools:
    return self._invoke_loop_native_tools()
return self._invoke_loop_react()
```

`_invoke_loop_react()` is the canonical ReAct loop. The output format (`en.json:12`) is classic ReAct text:

```
Thought: you should always think about what to do
Action: [tool_name]
Action Input: [JSON object]
Observation: the result of the action
...
Final Answer: the final answer
```

### 3.2 🎯 The Experimental Plan-and-Execute Executor (Major Finding)

**`AgentExecutor` (`experimental/agent_executor.py:156`) is a second executor activated by setting `executor_class=AgentExecutor` on any `Agent`.** It extends `Flow[AgentExecutorState]` and **implements genuine Plan-and-Execute**:

```python
class AgentExecutorState(BaseModel):
    plan: str | None = None
    plan_ready: bool = False
    todos: TodoList                      # typed list of TodoItem
    replan_count: int = 0
    observations: dict[int, StepObservation]
```

The flow starts at `@start generate_plan()` (`agent_executor.py:283`):

1. If `agent.planning_enabled`, call `AgentReasoning.handle_agent_reasoning()` to **generate a typed plan**
2. Convert plan steps to `TodoItem` objects in `state.todos`
3. `@router check_todos_available` → `@router get_ready_todos_method` → `execute_todo_sequential`
4. `execute_todo_sequential` calls `StepExecutor.execute()` — **one isolated LLM call per step**, with its own message context
5. After each step, `PlannerObserver` analyzes the result and either applies structured refinements or triggers full replanning
6. Loop until all todos complete or max replans exceeded

`PlanningConfig` (`agent/planning_config.py:78-137`) controls reasoning effort:
- `"low"` (observe only)
- `"medium"` (replan on failure)
- `"high"` (full observe-decide-refine-replan pipeline)

**`step_executor.py` explicitly cites `arxiv 2503.09572` (Plan-and-Act) in its module docstring.**

**This is genuine ④b mechanics at the single-agent level.** Typed `Plan`, typed `TodoList`, isolated step execution, observation-driven replanning. It is opt-in, not the default.

---

## 4. Multi-Agent Handoff (Cognition Critique Check)

### 4.1 How Agent A hands off to Agent B

Delegation goes through `DelegateWorkTool`. Schema:

```python
class DelegateWorkToolSchema(BaseModel):
    task: str       # The sub-task description
    context: str    # Context for the sub-task
    coworker: str   # Role name of the target agent
```

`BaseAgentTool._execute()` (`base_agent_tools.py:48-130`):

```python
task_with_assigned_agent = Task(
    description=task,
    agent=selected_agent,
    expected_output=I18N_DEFAULT.slice("manager_request"),
)
return selected_agent.execute_task(task_with_assigned_agent, context)
```

**The return value is a plain `str`** — the worker's final answer text.

### 4.2 Trace preservation: full or text-only?

**Text-only.** The delegating agent receives a plain string back. No tool call history, no intermediate reasoning steps, no structured trace from the worker's execution. The caller sees this in its message history as:

```
Action: Delegate work to coworker
Action Input: {"task": "...", "context": "...", "coworker": "Researcher"}
Observation: [worker's Final Answer text]
```

Between tasks in sequential mode, `_get_context()` calls `aggregate_raw_outputs_from_task_outputs()` — **a string concatenation of `TaskOutput.raw` fields**.

### 4.3 Direct match to Cognition's failure modes

**Cognition's critique applies precisely to default CrewAI:**

- **Sequential handoff**: Agent A's full internal ReAct trace (all Thought/Action/Observation iterations) is discarded. Only `TaskOutput.raw` (the final answer) is passed as context to the next task.
- **Delegation handoff**: `DelegateWorkTool._execute()` returns `selected_agent.execute_task(...)` as a string. The worker's entire reasoning trace is invisible to the caller.

An editor agent receiving a draft cannot know which claims the writer researched, which were uncertain, or which alternatives were rejected. **That context is lost at the boundary — the exact failure mode Cognition warned about.**

---

## 5. Flow Module (`src/crewai/flow/`)

### 5.1 Purpose

`Flow` is a **Python-native workflow DAG with event-driven routing**. Three decorators:

- `@start()` — entry point(s). Multiple start methods allowed.
- `@listen(condition)` — executes when `condition` method completes. Supports `or_()` and `and_()`.
- `@router(condition)` — returns a route string that dispatches to the appropriate `@listen` branch.

`FlowState` is a Pydantic `BaseModel` holding typed, mutable state persisted across all method calls.

### 5.2 Relationship to Crew

Flow and Crew are parallel abstractions at different levels of granularity:

- `Crew.kickoff()` orchestrates a fixed task list across agents — the right primitive for "run these 5 tasks in order."
- `Flow` orchestrates arbitrary Python methods with typed state, conditional branching, and parallel execution. A Crew can be called inside a Flow method.

**Critically, `AgentExecutor` (the experimental P&E executor) extends `Flow[AgentExecutorState]` directly** — the Plan-and-Execute control flow is itself implemented using `@start`/`@router`/`@listen`. **This is the clearest signal from the CrewAI team about where they see the future of complex execution: in Flow, not in Crew.**

---

## 6. Memory, Knowledge, RAG (brief)

`Crew` supports `memory=True` for short/long/entity/external-term memory. `Agent` and `Crew` both accept `knowledge_sources` for RAG retrieval. Standard infrastructure concerns orthogonal to execution model.

---

## 7. Direct Comparison with AgentScope

### 7.1 Plan representation

| Dimension | CrewAI default | CrewAI experimental | AgentScope |
|-----------|---------------|---------------------|-----------|
| Plan primitive | None (tasks are user-written) | **`TodoList[TodoItem]` with status** | `Plan` + `SubTask` Pydantic models |
| Plan generation | `CrewPlanner` prepends text to task descriptions | **`AgentReasoning` → `PlanStep` list at runtime** | LLM calls `create_plan` tool at runtime |
| Plan storage | None (mutates `task.description` in place) | **`AgentExecutorState.todos` in Flow state** | `PlanNotebook` with `InMemoryPlanStorage` |
| Plan visibility | Text concatenated into prompt | **Typed `TodoList`, observable per-step** | Typed `Plan`, observable from outside |

### 7.2 Plan generation: user-defined vs LLM-generated

**Default CrewAI with `planning=True`** (`crew.py:1317-1343`):
1. `CrewPlanner._handle_crew_planning()` runs a planning agent
2. The plan text is **appended to `task.description`** (`crew.py:1338`): `task.description += plan_map[task_number]`
3. Task execution is unchanged — each task's agent runs the same ReAct loop with an enriched prompt

**The "plan" is prompt augmentation, not structural control.** The task list structure is immutable.

**AgentScope with `PlanNotebook`**: the LLM dynamically calls `create_plan` during execution to create typed `Plan`/`SubTask` objects. The notebook enforces ordering at the tool level.

**CrewAI experimental (`AgentExecutor` + `PlanningConfig`)**: `AgentReasoning` generates typed `PlanStep` objects from a single LLM call, converted to `TodoItem` objects with dependency tracking, with `PlannerObserver` providing post-step analysis and replanning. **Structurally closest to AgentScope's PlanNotebook — both produce typed step artifacts at runtime.**

### 7.3 Enforcement mechanism

| | CrewAI default | CrewAI experimental | AgentScope |
|--|---------------|---------------------|-----------|
| Task order | Immutable list index | `TodoList.get_ready_todos()` (dependency graph) | Sequential state machine |
| Who controls order | Python (list iteration) | Framework (dependency check) | Framework (ordering constraint) |
| Can LLM reorder? | No | **Yes (replan)** | Yes (`revise_current_plan` tool) |

---

## 8. Position on the Agent Architecture Spectrum

### 8.1 Static Plan vs Dynamic Plan — a new nuance

**Static Plan** = the user defines all tasks before `kickoff()`. The LLM never generates the task structure. Architecturally equivalent to a workflow where each step happens to use an LLM.

**Dynamic Plan** = an LLM generates the task structure at runtime in response to the goal. The task list itself is a product of reasoning.

**CrewAI in its default mode is static Plan** for both process types: the user writes all tasks, the framework runs them. Even with `planning=True`, the planning step only augments prompts — **the task list structure is not changed**.

**The hierarchical process feels like dynamic planning** because the manager decides how to delegate, but this is shallow: the manager receives the same fixed task list and decides execution routing for pre-defined tasks. It is not generating sub-tasks.

**The only path to dynamic Plan in CrewAI is the experimental `AgentExecutor`** with `PlanningConfig` — where `AgentReasoning` generates a `TodoList` at runtime. **This is genuinely ④b at the single-agent level.**

### 8.2 CrewAI's actual category

**CrewAI occupies two distinct positions** depending on which executor is used:

**Default (`CrewAgentExecutor`)**: **④a with a static multi-agent wrapper.**

The "multi-agent" aspect is architectural sugar. Each Task is processed by an agent running a standard ReAct loop. The Crew is a sequential scheduler, not an intelligence layer. The role/backstory prompting is **precisely the "CEO/CTO/Analyst" roleplay that Cognition criticizes**. The framework provides no structural guarantee that tasks form a coherent plan.

**Experimental (`AgentExecutor` + `PlanningConfig`)**: **genuine ④b at the single-agent level**, with Crew as an optional macro-scheduler above it.

`AgentExecutor` implements genuine Plan-and-Execute: plan generation → typed todo list → isolated step execution → post-step observation → conditional replanning with dependency-aware parallel execution. **It cites arxiv 2503.09572 in its module docstring.**

**The marketed CrewAI** — the "build your AI team" framework of most tutorials and documentation — is at the **③/④a boundary**: a static workflow where each step is an LLM call with ReAct tool use, wrapped in role-based prompt engineering. The "team" metaphor obscures what is architecturally a sequential text pipeline.

---

## Key Architectural Insights

1. **Role is prompt, not capability.** `role`, `goal`, and `backstory` are string fields injected into the system prompt. They shape the LLM's persona but impose no structural constraints.

2. **Hierarchical mode does not generate tasks.** `_run_hierarchical_process()` still takes the same `self.tasks` list and routes each task through the manager. **The task list is immutable across both process modes.**

3. **Context passing is text concatenation.** `_get_context()` calls `aggregate_raw_outputs_from_task_outputs()` — a string join of `TaskOutput.raw`. **No structured data, no tool call history, no reasoning trace crosses task boundaries.** This is the "lost context" pattern Cognition describes.

4. **🎯 The experimental executor is a real P&E implementation.** `AgentExecutorState` has `plan: str`, `todos: TodoList`, `replan_count: int`, `observations: dict[int, StepObservation]`. `StepExecutor` runs one isolated LLM call per step. `PlannerObserver` provides structured post-step analysis with replanning triggers. **This is a complete implementation of the Plan-and-Act paper (arxiv 2503.09572).**

5. **Flow is the correct layer for complex orchestration.** `AgentExecutor` itself is a `Flow` subclass, signaling that the CrewAI team reaches for Flow when building structured control flow.

6. **`planning=True` on Crew is misnamed.** It augments task prompts with execution hints — it does not generate a Plan in the P&E sense. **Crew-level `planning` is prompt enrichment; agent-level `planning_config=PlanningConfig()` is runtime plan generation. These are completely different mechanisms sharing an overloaded term.**

---

## Conclusion

CrewAI is best described as a **role-based static-workflow framework with a ReAct execution core and an opt-in P&E mode**. Its marketed value proposition — "build your AI team" — is primarily prompt engineering. Users pre-define the workflow as a task list. The framework then runs each task through a standard ReAct agent loop, passing raw text between tasks.

**The hierarchical process does not make CrewAI a ④b framework.** The manager agent routes tasks to workers using delegation tools, but the task list itself is static and user-authored.

**For the blog's Agent Map**:

- **Default CrewAI (what 95% of tutorials show)**: ③/④a boundary — static workflow, each step is LLM ReAct, wrapped in role-based prompts. The "team" metaphor obscures what is architecturally a sequential text pipeline. **This is also the clearest public example of the "role-based multi-agent" pattern Cognition specifically criticized.**

- **🎯 CrewAI with `AgentExecutor` + `PlanningConfig`**: **genuine ④b** — runtime plan generation, typed todo tracking, isolated step execution, post-step replanning. A credible P&E implementation that in some respects exceeds AgentScope's PlanNotebook in structural completeness (dependency-aware todo execution, multi-tier reasoning effort).

**Critical caveat**: The P&E implementation lives in `src/crewai/experimental/` — **the same "experimental" namespace where LangChain's P&E lived before being dropped entirely**. The historical pattern is clear: **major frameworks gate P&E behind experimental flags**. Whether CrewAI's experimental executor graduates to main or gets dropped (like LangChain's) is an open question.

Comparing with AgentScope's `PlanNotebook`: both enable LLM-generated plans with typed step artifacts. **AgentScope's plan is a sidecar to an otherwise-standard ReAct loop; CrewAI's experimental executor wraps the entire execution loop in a P&E Flow.** CrewAI's P&E is more structurally complete, but AgentScope's is the default experience; CrewAI's requires opting in to an `experimental` module.
