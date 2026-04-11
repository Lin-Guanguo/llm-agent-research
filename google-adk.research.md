# Google ADK (Agent Development Kit) Research

Last Updated: 2026-04-11

> **Research Methodology**: Direct WebFetch verification against `https://github.com/google/adk-python` README (primary source) plus web search cross-reference. Not yet source-level analyzed at the depth of `mastra.research.md` — this is a README-level + examples-level survey. Deeper source-level analysis remains TODO.

## Sources

- [Google ADK GitHub repository](https://github.com/google/adk-python) (accessed 2026-04-11)
- [Google ADK official documentation](https://adk.dev/) (accessed 2026-04-11)
- [Google ADK Python Quick Start](https://adk.dev/get-started/python/) (accessed 2026-04-11)
- [Google ADK Multi-Agent Systems Guide](https://adk.dev/agents/multi-agents/) (accessed 2026-04-11)
- [Google ADK Runtime and Event Loop](https://adk.dev/runtime/event-loop/) (accessed 2026-04-11)

## Overview

**Exact opening description (from README)**:

> *"an open-source, code-first Python framework for building, evaluating, and deploying sophisticated AI agents with flexibility and control"*

**Project basics**:
- **Maintained by**: Google (official)
- **License**: Open source
- **Language**: Python
- **Stars**: 18.9k (as of 2026-04-11)
- **Forks**: 3.2k
- **Release cadence**: Bi-weekly (as of early 2026)
- **Models supported**: Model-agnostic in principle; Gemini-first (`gemini-flash-latest` is the default in examples), with support for OpenAI-compatible APIs

## 1. Agent Definition (API Layer)

### Core Classes

- `BaseAgent` — abstract base (in `src/google/adk/agents/base_agent.py`)
- `Agent` — standard LLM-driven agent (inherits from BaseAgent)
- Workflow agents for orchestration:
  - `SequentialAgent` — runs sub-agents one after another
  - `ParallelAgent` — runs sub-agents concurrently
  - `LoopAgent` — repeats a sequence of sub-agents until a stop condition

### Minimal Example (from official Quick Start)

```python
from google.adk import Agent

root_agent = Agent(
    name="search_assistant",
    model="gemini-2.5-flash",
    instruction="You are a helpful assistant...",
    description="An assistant that can search the web.",
    tools=[google_search]
)
```

**Configuration parameters**:
- `name` — unique identifier, must be a valid Python identifier
- `model` — model string or model object
- `instruction` — system prompt defining behavior
- `tools` — list of tools available to the agent
- `description` — optional human-readable description
- `sub_agents` — optional list of sub-agents (creates a tree structure)

### Project Initialization

ADK ships with a CLI:
- `adk create <project_name>` — scaffolds a new project with `agent.py` containing a `root_agent`
- `adk run` — executes the agent in CLI mode
- `adk web` — runs a web-based interaction UI

## 2. Execution Model: Pure ReAct

**Verified via README and official docs**: Google ADK uses a **pure ReAct loop** as its core execution model. The official docs describe the pattern as:

> "ReAct pattern is powerful for complex problem-solving tasks where the agent needs to reason, gather information, and act iteratively."

### The Loop (per official runtime docs)

For a single `Agent`:

1. **Reason** — LLM reads the current context (instruction + conversation history + tool results) and reasons about the next action
2. **Act** — LLM emits a tool_call or final response
3. **Observe** — If tool_call, the Runner executes the tool and appends the result to context
4. **Repeat** — Until LLM emits a final response or a stop condition triggers

The loop is driven by a `Runner` class that:
- Loads session and invocation context
- Calls the agent's `run_async()` method
- Yields events (tool calls, responses, state deltas)
- Persists session state
- Prepares the next iteration

**No separate "planning phase"**. No pre-execution Plan artifact. Planning is implicit in the LLM's reasoning at each step.

## 3. Workflow / Orchestration (Multi-Agent Composition)

Google ADK's multi-agent model uses **Workflow Agents** as the orchestration primitive. Unlike Mastra (which uses a `Workflow` DAG with arbitrary operators), ADK provides three fixed orchestrators:

| Orchestrator | Behavior |
|---|---|
| `SequentialAgent` | Runs sub-agents one after another, sharing `session.state` |
| `ParallelAgent` | Runs sub-agents concurrently, shared state |
| `LoopAgent` | Repeats a sequence of sub-agents until `max_iterations` or an escalation signal |

### Example: Generator-Critic Loop

```python
LoopAgent(
    name="generator_critic",
    sub_agents=[generator_agent, critic_agent, refinement_agent],
    max_iterations=3
)
```

Each sub-agent inside the loop is still a **pure ReAct agent**. The LoopAgent only controls the outer iteration.

**Important subtlety**: `LoopAgent` is often marketed as "planning + execution," but it is **not** Plan-and-Execute in the architectural sense:
- No structured Plan artifact is produced before execution
- Each sub-agent still decides its own actions via ReAct
- The loop is just flow-control over multiple ReAct agents

### Multi-Agent Communication Mechanisms

1. **Shared state** (`session.state`) — passive communication via a shared dictionary
2. **LLM-driven delegation** (`transfer_to_agent(agent_name)`) — one agent's LLM can dynamically route to another agent (this is the closest ADK gets to "dynamic routing")
3. **AgentTool wrapping** — wrap an agent as a tool that another agent can call

## 4. Control & Predictability

### Stop Conditions

- Per-agent: implicit context-window limit (no explicit `max_iterations` for individual `Agent` by default)
- `LoopAgent`: explicit `max_iterations` parameter
- Escalation: a sub-agent can set `escalate=True` in its event action to break out of a loop

### State Management

- **Event-based model**: state changes are applied via `Event` objects with `state_delta`
- **Atomicity**: partial (streaming) events do NOT apply state changes; only complete events do
- **Dirty reads**: same-invocation code can read local uncommitted state (convenience for multi-step coordination)

### Type Validation

- Agent `name` must be a valid Python identifier (validated by Pydantic)
- Tools are defined via Python function signatures and type hints
- **No explicit schema validation** comparable to Mastra's Zod-based Typed Output
- No flow-level type safety

### Human-in-the-Loop

- **Callback system**: callbacks can intercept agent execution before/after
- From docs: *"When a list of callbacks is provided, callbacks will be called in order until one returns non-None"* — callbacks can interrupt or modify execution
- Official example: `a2a_human_in_loop` demonstrates a human confirmation step for tool calls

### No Plan Artifact

- No standalone Plan object
- No `plan_agent` / `execute_agent` separation
- Planning happens implicitly inside each agent's reasoning steps
- **Google ADK does not support Plan-and-Execute as a first-class architectural primitive**

## 5. Comparison with Mastra

| Dimension | Google ADK | Mastra |
|---|---|---|
| **Core execution** | Pure ReAct | Pure ReAct |
| **Loop control** | `LoopAgent` with `max_iterations` | `.dowhile()` with arbitrary condition function |
| **Multi-agent orchestration** | Fixed: Sequential / Parallel / Loop | General Workflow DAG with `.then()` / `.parallel()` / `.foreach()` / `.dowhile()` |
| **State management** | Event-based + `session.state` | Message history + optional memory system |
| **Plan artifact** | None | None |
| **Type validation** | Python type hints (Pydantic) | Zod schemas (boundary-level) |
| **Human-in-the-loop** | Callback system + `a2a_human_in_loop` example | `.suspend()` / `.resume()` on workflow steps |
| **Tools** | User-defined Python functions | User-defined via `createTool()` |
| **Language** | Python | TypeScript |

**Assessment**: Google ADK and Mastra are **architectural twins in different languages**. Both use ReAct as the core execution model, both provide workflow-level orchestration as the outer layer, both lack a Plan artifact, both rely on boundary validation rather than flow-level types.

The main differences are in language choice, orchestration primitive design (Mastra's DAG is more flexible than ADK's fixed three), and runtime implementation details. **They are not fundamentally different architectures.**

## 6. Position in the Agent Map

**Google ADK belongs to category ④a (Code Framework, ReAct school).**

Rationale:
1. ✅ Core execution is ReAct: each agent decides its next action via LLM reasoning at every step
2. ✅ No global planning phase: `LoopAgent` is flow-control orchestration, not a Plan-Execute separation
3. ✅ Code framework: Python library, not a low-code or cloud platform
4. ✅ Open source, self-hostable (also deployable to Vertex AI, but this doesn't change the architecture)

**Not ④b** because there is no structured Plan artifact generated before execution.
**Not ③** because there is no visual/low-code interface.
**Not ①②** because ADK is a framework you build with, not a ready-to-use product.

## 7. Implications for the Agent Map

Google ADK's placement in ④a (confirmed) combined with AgentScope's placement in ④a (also confirmed) means:

- **All four major "code-first Agent frameworks"** (Mastra, Google ADK, AgentScope, LangGraph's `create_react_agent`) are ReAct-based
- The ④b (P&E school) category has **zero verified public representatives**
- The gap between "framework you can `pip install`" and "production P&E system like the user's Agent 2.0" is **wider than originally thought**

This is a significant finding. It means when someone looks for a framework to build a production Agent system with cost predictability, audit, and Plan-artifact-based control, they will find:
- Plenty of ReAct frameworks (③ ④a)
- Zero P&E frameworks
- And will therefore have to build custom (⑤)

The user's Agent 2.0 is not "a slightly different architecture from the mainstream" — it is **filling a gap in the public framework landscape**.

## TODO

- [ ] Source-level analysis (check `src/google/adk/agents/base_agent.py` and `runtime/*.py`)
- [ ] Compare `LoopAgent` implementation to Mastra's `.dowhile()` at the source level
- [ ] Check if there is any hidden P&E-like structure in ADK's upcoming roadmap
