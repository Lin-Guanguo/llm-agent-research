# LangGraph Family Research Note

Last Updated: 2026-05-21

## Purpose

This note records the relationship between LangGraph, LangChain agents, and
Deep Agents, and positions them against the Dayfold Agent design.

The key point is that these projects sit at different abstraction levels. The
shared brand can be confusing, but architecturally they answer different
questions:

- **LangGraph**: how to run a stateful workflow or graph.
- **LangChain agents**: how to package a common LLM tool-calling loop on top of
  LangGraph.
- **Deep Agents**: how to bundle a more opinionated long-running agent harness
  with planning, filesystem state, subagents, sandboxing, and approvals.
- **Dayfold Agent**: how to let an LLM generate a typed executable plan that is
  validated and executed by a deterministic runtime.

## Sources

Official references:

- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) (accessed: 2026-05-21)
- [LangGraph quickstart](https://docs.langchain.com/oss/python/langgraph/quickstart) (accessed: 2026-05-21)
- [Thinking in LangGraph](https://docs.langchain.com/oss/python/langgraph/thinking-in-langgraph) (accessed: 2026-05-21)
- [LangChain overview](https://docs.langchain.com/oss/python/langchain/overview) (accessed: 2026-05-21)
- [Frameworks, runtimes, and harnesses](https://docs.langchain.com/oss/python/concepts/products) (accessed: 2026-05-21)
- [Deep Agents overview](https://docs.langchain.com/oss/python/deepagents/overview) (accessed: 2026-05-21)
- [LangChain built-in middleware](https://docs.langchain.com/oss/python/langchain/middleware/built-in) (accessed: 2026-05-21)

Local source-level references:

- `/Users/linguanguo/dev/llm-agent-research/langgraph.research.md`
- `/Users/linguanguo/dev/llm-agent-research/langchain.research.md`
- `/Users/linguanguo/dev/llm-agent-research/langchain/libs/langchain_v1/langchain/agents/factory.py:691`
- `/Users/linguanguo/dev/llm-agent-research/langchain/libs/langchain_v1/langchain/agents/factory.py:707`
- `/Users/linguanguo/dev/llm-agent-research/langchain/libs/langchain_v1/langchain/agents/factory.py:809`
- `/Users/linguanguo/dev/llm-agent-research/langchain/libs/langchain_v1/langchain/agents/middleware/todo.py:162`
- `/Users/linguanguo/dev/llm-agent-research/langchain/libs/langchain_v1/langchain/agents/middleware/todo.py:165`
- `/Users/linguanguo/dev/llm-agent-research/langchain/libs/langchain_v1/langchain/agents/middleware/todo.py:170`
- `/Users/linguanguo/dev/deepagents/README.md:24`
- `/Users/linguanguo/dev/deepagents/README.md:76`
- `/Users/linguanguo/dev/deepagents/libs/deepagents/deepagents/graph.py:12`
- `/Users/linguanguo/dev/deepagents/libs/deepagents/deepagents/graph.py:69`
- `/Users/linguanguo/dev/deepagents/libs/deepagents/deepagents/graph.py:217`
- `/Users/linguanguo/dev/deepagents/libs/deepagents/deepagents/graph.py:241`
- `/Users/linguanguo/dev/deepagents/libs/deepagents/deepagents/graph.py:308`
- `/Users/linguanguo/dev/deepagents/libs/deepagents/deepagents/graph.py:671`
- `/Users/linguanguo/dev/deepagents/libs/deepagents/deepagents/graph.py:765`
- `/Users/linguanguo/dev/deepagents/libs/deepagents/deepagents/middleware/filesystem.py:592`
- `/Users/linguanguo/dev/deepagents/libs/deepagents/deepagents/middleware/subagents.py:273`
- `/Users/linguanguo/dev/deepagents/libs/deepagents/deepagents/middleware/summarization.py:1`

## 1. Layering

The current mental model:

```text
LangGraph
  = workflow runtime abstraction
  = execution flow, state, node, edge, checkpoint, interrupt, time travel

LangChain
  = generic tool-calling agent abstraction
  = provider abstraction + minimal agent loop + middleware

Deep Agents
  = opinionated long-task harness on top of LangChain create_agent
  = default prompt + todo + filesystem + subagents + context management + approval
```

| Layer | Primary Question | Control-Flow Owner | Plan Shape |
|---|---|---|---|
| LangGraph | How do we run a stateful graph or workflow? | Developer-authored graph runtime | Code-defined workflow |
| LangChain `create_agent` | How do we create a standard tool-calling agent quickly? | Prebuilt graph plus LLM tool loop | Mostly implicit; middleware can add state |
| LangChain `TodoListMiddleware` | How does an agent track complex work? | LLM updates todo state through a tool | Todo list as model-visible working memory |
| Deep Agents | How do we package a long-running general-purpose agent harness? | LLM tool loop plus harness tools and middleware | Todo, filesystem, subagents, sandbox state |
| Dayfold Agent | How do we execute business tasks predictably? | LLM plans, deterministic runtime executes | Typed executable plan with dataflow bindings |

## 2. LangGraph: Workflow Runtime

LangGraph is best understood as a stateful workflow runtime. It is not mainly a
planner, nor does it force an agentic loop. Its core abstraction is a graph of
nodes over state.

The official docs emphasize durable execution, persistence, human-in-the-loop,
memory, and state management. Those are runtime concerns. They make LangGraph a
good substrate for agents, but they do not by themselves define how the agent
plans.

The user-facing quickstarts and tutorials also reveal the intended mental model:
developers define a workflow graph and place LLM calls inside selected nodes.
The current quickstart is simple, but the "Thinking in LangGraph" guide is even
more revealing because it models a business-like email workflow. That example is
closer to workflow engineering than to a free-form autonomous agent.

The important distinction:

```text
LangGraph workflow:
developer writes graph
  -> runtime executes graph
  -> LLM may appear inside individual nodes
```

This differs from an LLM-authored execution plan. In plain LangGraph, the graph
structure is usually authored by the developer. The LLM may classify, extract,
summarize, decide a route, or produce text, but the graph topology is a program
artifact.

## 3. LangChain Agents: Prebuilt Tool Loop On LangGraph

LangChain v1 adds a higher-level agent API on top of LangGraph.

The local source confirms that `create_agent(...)` returns a
`CompiledStateGraph` (`factory.py:707`). Its docstring describes the standard
tool-calling loop:

```text
model node
  -> if tool_calls: tool node
  -> append ToolMessage
  -> model node again
  -> stop when there are no tool_calls
```

This is a useful default. Developers no longer need to manually wire a
model-tool-model loop in LangGraph. LangChain also provides:

- model and tool abstraction;
- structured output handling;
- middleware hooks;
- state and context schema extension;
- checkpointer and store integration;
- interrupt hooks before or after graph nodes.

Architecturally, this is still not a typed executable plan. It is a prebuilt
ReAct-like loop implemented as a graph. The LLM remains the local controller of
the next tool call.

## 4. TodoListMiddleware: Soft Planning As Working Memory

`TodoListMiddleware` is the most relevant LangChain piece for the current
Dayfold comparison.

The source-level behavior is clear:

- it adds a `write_todos` tool;
- it adds todo state to the agent state schema;
- it injects a system prompt that guides the agent to use todos;
- it rejects multiple parallel `write_todos` calls in one model turn because the
  tool replaces the whole todo list.

This is a planning mechanism, but it is a soft planning mechanism.

The todo list is not compiled into a runtime DAG. The runtime does not read a
todo item, validate dependencies, bind typed outputs, and dispatch a capability
because a todo item is ready. Instead, the todo list helps the model maintain
attention, expose progress to the user, and decide what to do next in future
model calls.

The key distinction:

| Dimension | TodoListMiddleware | Typed Executable Plan |
|---|---|---|
| Primary role | Working memory and progress tracking | Runtime execution contract |
| Completion owner | Mostly model-declared via todo update | Runtime step state plus validators |
| Execution owner | LLM chooses next tool call | Program scheduler chooses ready steps |
| Dependency model | Informal text/status | Explicit bindings and capability I/O |
| Best fit | Open-ended general tasks | Business workflows with stable contracts |

This is why todo-mediated agents are mainstream for general agents: they are
cheap to add, model-visible, and flexible. They avoid requiring a business DSL.
The cost is that they leave execution control mostly inside the model loop.

## 5. Deep Agents: Opinionated General-Agent Harness

Deep Agents sits above the basic LangChain agent API. It packages a collection
of capabilities that are common in long-running general-purpose agents:

- todo planning;
- filesystem-based state;
- subagents;
- sandboxed execution;
- context management;
- permission and approval controls.

Source-level reading confirms this. `deepagents.graph` imports
`langchain.agents.create_agent`, assembles a middleware stack, builds a final
system prompt, and returns `create_agent(...)`. The default `create_deep_agent`
tool surface includes:

- `write_todos`;
- `ls`;
- `read_file`;
- `write_file`;
- `edit_file`;
- `glob`;
- `grep`;
- `execute`;
- `task`.

The default middleware stack includes:

- `TodoListMiddleware`;
- `FilesystemMiddleware`;
- `SubAgentMiddleware`;
- `AsyncSubAgentMiddleware`;
- summarization middleware;
- `PatchToolCallsMiddleware`;
- `SkillsMiddleware` when skills are configured;
- `MemoryMiddleware` when memory is configured;
- `HumanInTheLoopMiddleware` when `interrupt_on` is configured;
- provider/profile-specific middleware such as Anthropic prompt caching.

So "Deep Agents" is not just LangGraph and not just `create_agent`. It is an
agent harness: a ready-made bundle of prompt, tools, middleware, state, and
subagent conventions for long tasks.

One nuance: Deep Agents is not necessarily a direct local-host filesystem agent
by default. The filesystem is backend-backed. The default backend is state-based
ephemeral storage, and shell execution through `execute` only works when the
backend implements the sandbox execution protocol. A Deep Agents application can
therefore be local, sandboxed, remote, or state-backed depending on backend
configuration.

This makes it closer to coding-agent products than to a business-specific
typed-plan runtime. It is designed to help a general agent manage long context
and complex work, not to force every plan into a product-specific executable IR.

Deep Agents is therefore a useful comparison point, but not the same design
space as Dayfold:

```text
Deep Agents:
general-purpose harness
  -> todo / filesystem / subagents / sandbox
  -> LLM continues to drive the task loop

Dayfold Agent:
business-specific typed runtime
  -> planner emits execution plan
  -> program validates, schedules, binds, dispatches
```

## 6. What LangChain Built On Top Of LangGraph

The practical answer is:

1. **LLM capability abstraction.** LangChain normalizes provider-specific model
   capabilities: message roles, tool schemas, tool-call result messages,
   structured output, streaming, usage metadata, and provider-specific quirks.
2. **A standard agent factory.** `create_agent` hides the graph construction for
   the common model-tool loop.
3. **Middleware as composable control primitives.** Middleware can modify model
   calls, tool calls, state, prompts, retries, approval, and todo tracking.
4. **Runtime integration.** LangChain exposes LangGraph concepts like
   checkpointing, stores, interrupts, and compiled graphs without requiring
   users to write the graph manually.
5. **Deep Agents as a harness.** Deep Agents assembles a more complete
   long-running agent toolkit on top of those pieces.

This is a sensible product stack:

```text
LangGraph = stateful graph runtime
LangChain agents = prebuilt agent loop and middleware framework
Deep Agents = opinionated long-task harness
```

## 7. What The Agent Loop Does Not Do

The modern `model -> tool -> model` loop is mostly protocol and runtime logic,
not a ReAct-style prompt convention.

The runtime sends the model:

- messages;
- available tool schemas;
- optional structured-output settings;
- optional system prompt or middleware-injected prompt.

The model returns an assistant message. If that message contains structured
`tool_calls`, the runtime executes the corresponding tools, appends tool results
as tool messages, and calls the model again. This repeats until the model
returns no tool call.

The loop itself does not require a built-in system prompt. In local source:

- LangGraph prebuilt `create_react_agent` has `prompt: Prompt | None = None`.
- If `prompt is None`, `_get_prompt_runnable` passes through `state["messages"]`
  without adding a system message.
- LangChain v1 `create_agent` similarly has `system_prompt=None`; it only creates
  a `SystemMessage` when the caller provides one.

This is different from older text ReAct agents, where the model had to emit
parseable text such as `Thought`, `Action`, and `Observation`. Modern tool
calling moves the action channel into provider APIs and message schemas.

Tool descriptions still matter. They are best understood as **tool-level
instructions attached to the action interface**. Tool names, descriptions, and
parameter descriptions are prompt-like, but they are passed through the
structured tool schema channel rather than through ordinary text prompts.

## 8. Middleware Mechanisms

LangChain middleware is not a single hook mechanism. It can affect an agent loop
through several channels:

| Mechanism | What It Changes | Example |
|---|---|---|
| Hook | Inserts logic before or after model/tool/agent execution. | `before_model`, `after_model`, `wrap_model_call`, `wrap_tool_call` |
| State injection | Adds fields to agent state. | `TodoListMiddleware` adds `todos`. |
| Tool injection | Adds tools to the model-visible action surface. | `TodoListMiddleware` adds `write_todos`. |
| Prompt injection | Modifies the model request or system message. | Todo instructions appended before model call. |
| Runtime control | Interrupts, limits, retries, or reroutes execution. | HITL, call limits, retry/fallback middleware. |
| Tool filtering / rewriting | Changes available tools or tool-call behavior. | Tool selection or approval policies. |
| Result transformation | Changes outputs or compacted state. | Summarization or output-fixing middleware. |

`TodoListMiddleware` is a good example because it combines multiple mechanisms:

```text
TodoListMiddleware
  -> state injection: add todos
  -> tool injection: add write_todos
  -> prompt injection: explain when to use todo
  -> hook: reject multiple parallel write_todos calls in one model turn
```

This explains why middleware is a meaningful abstraction. It lets framework
authors add cross-cutting control behavior without rewriting the main
model-tool-model loop.

The tradeoff is that system behavior becomes distributed. Some behavior lives in
the graph, some in model/tool schemas, some in prompt injection, and some in
middleware hooks.

## 9. How Similar Are Deep Agents, Claude Code, And OpenClaw?

At the core loop level, Deep Agents, Claude Code / Codex-style coding agents,
and OpenClaw are similar:

```text
model
  -> structured tool call
  -> runtime executes tool under policy
  -> tool result returns to model
  -> model continues
```

Their plans are usually soft plans: todos, plan documents, task prompts, skill
instructions, memory snippets, and context state. These are model-visible
artifacts that guide attention and behavior. They are not normally compiled into
a typed execution DAG that the runtime schedules step by step.

Assuming the same base model, the meaningful differences mostly come from the
agent shell around the model:

| Difference Surface | What Changes |
|---|---|
| System prompt | Work style, persistence, verification expectations, when to ask users, when to use todo or subagents. |
| Tool descriptions and schemas | Which actions the model sees, how it chooses them, how parameters are shaped, and how errors are represented. |
| Tool implementation | Filesystem semantics, shell sandboxing, patch application, browser/app integration, messaging, cron, or product actions. |
| Context assembly | How history, tool results, files, memory, background tasks, errors, and summaries are normalized into model-visible context. |
| Memory and skills | What persists across sessions and what reusable procedures can be loaded on demand. |
| Permissions and side-effect gates | Approval, sandbox, owner-only tools, policy filtering, and tool-call interruption. |
| Subagents and lifecycle | Short-lived task agents, background agents, persistent sessions, verifier agents, cron, wake, or heartbeat behavior. |
| Product integration | Terminal, IDE, chat channels, personal workspace, browser/app surfaces, deployment and observability. |

This means Deep Agents should be interpreted as a framework extraction of
common long-task agent primitives. It includes many of the same primitives seen
in Claude Code / Codex-style agents and OpenClaw: todo, filesystem, shell or
sandbox execution, subagents, skills, summarization, memory, and approval. What
it does not include by itself is the mature product shell of Claude Code or
OpenClaw's personal-agent runtime envelope.

The concise claim:

> Deep Agents packages the common model-driven long-task agent pattern as an
> SDK. Claude Code and OpenClaw are productized agent shells around the same
> broad pattern, with deeper product-specific integrations.

## 10. Relevance To Dayfold Agent

Dayfold uses LangGraph in the LangGraph sense: as a workflow runtime.

It does not primarily use the LangChain/DeepAgents soft-plan pattern. The core
Dayfold design is its own planner/runtime contract:

- LLM generates a typed execution plan;
- plan validation checks the structure;
- coordinator decides dependency readiness;
- input bindings materialize upstream output into downstream input;
- capability dispatcher validates and executes typed business nodes;
- runtime state records step completion and output ports.

That means Dayfold should be positioned as:

> LangGraph runtime plus a business-specific typed-plan interpreter.

It is not:

> A LangChain `create_agent` loop with a todo list.

The most important comparison:

| Question | LangGraph | LangChain Agent | Deep Agents | Dayfold Agent |
|---|---|---|---|---|
| Who defines the graph? | Developer | LangChain factory | Harness | Runtime graph plus planner-generated plan |
| Who decides next action? | Program edge/router | LLM tool loop | LLM with harness tools | Program scheduler after LLM planning |
| What is the plan? | Code workflow | Usually implicit | Todo/filesystem state | Typed executable plan |
| What is validated? | Graph/state mechanics | Tool schemas and middleware rules | Harness permissions and tools | Plan schema, bindings, capability I/O |
| Best use case | Explicit workflows | Quick tool agents | General long tasks | Product-specific business automation |

## 11. Middleware Takeaway For Dayfold

LangChain middleware is built around a generic tool-calling loop. Dayfold needs
similar cross-cutting capabilities, but the insertion points are different
because the core runtime is a typed plan interpreter.

| LangChain Middleware Surface | Dayfold Runtime Analogue |
|---|---|
| before_model | before planner call |
| wrap_model_call | planner provider/prompt policy |
| after_model | planner output validation |
| wrap_tool_call | capability dispatch boundary |
| after tool call | capability result validation and port write |
| before_agent | turn preparation |
| after_agent | result digest and finalization |
| call limits | replan attempts, step count, execution wave limits |
| HITL | side-effect capability gates |
| summarization | workflow state compaction |

This suggests a useful retrospective question:

> Which Dayfold behaviors are core runtime semantics, and which are
> middleware-like cross-cutting policies?

Planner retry, semantic validation, human approval, state compaction, metrics,
policy guards, and model fallback may be better treated as explicit hook points
than as code mixed into the planner or coordinator core.

## 12. Writing Takeaway

For the internal sharing document, the "LangGraph family" should not be treated
as one pattern. It should be introduced as a layered stack:

1. LangGraph shows the workflow-runtime baseline.
2. LangChain shows the mainstream prebuilt tool-loop agent on that runtime.
3. Todo middleware shows the mainstream soft-plan pattern.
4. Deep Agents shows how LangChain packages those primitives into a general
   agent harness.
5. Dayfold shows a different choice on the same runtime substrate: typed
   executable planning for business-specific stability and cost control.

This framing avoids the misleading conclusion that "we use LangGraph, therefore
we are close to LangChain/DeepAgents." The shared dependency is the runtime. The
planning semantics are different.
