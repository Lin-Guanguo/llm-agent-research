# Tool-Loop Agent Difference Surfaces

Last Updated: 2026-05-21

## Purpose

This note records a working conclusion from comparing Claude Code / Codex-style
coding agents, OpenClaw, LangChain `create_agent`, and Deep Agents.

When these systems use the same base model and the same modern tool-calling
protocol, their core local loop is often similar:

```text
model
  -> structured tool call
  -> runtime executes tool under policy
  -> tool result returns to model
  -> model continues
```

The major product and architecture differences therefore come less from a
completely different reasoning algorithm, and more from the agent shell around
the model.

## Public Sources Checked

- [OpenAI: Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/) documents Codex as a harness around a model/tool loop. It also shows that Codex builds context from bundled model instructions, tool definitions, user/project instructions, environment context, skills, sandbox/approval instructions, and later compaction.
- [OpenAI: A practical guide to building agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/) frames agents around instructions, tools, guardrails, hooks, and orchestration. It treats tools as data/action/orchestration surfaces and describes manager agents as tools.
- [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) separates predefined workflows from model-directed agents, emphasizes simple composable patterns, and stresses agent-computer-interface design through clear tool documentation and testing.
- [Claude Code: Extend Claude Code](https://code.claude.com/docs/en/features-overview), [Memory](https://code.claude.com/docs/en/memory), [Subagents](https://code.claude.com/docs/en/sub-agents), [Skills](https://code.claude.com/docs/en/skills), and [Hooks](https://code.claude.com/docs/en/hooks) expose the product surfaces around Claude Code: CLAUDE.md, skills, MCP, subagents, hooks, permissions, and auto memory.
- [LangChain: Deep Agents overview](https://docs.langchain.com/oss/python/deepagents/overview), [Deep Agents blog](https://www.langchain.com/blog/deep-agents), [LangChain middleware overview](https://docs.langchain.com/oss/python/langchain/middleware/overview), and [prebuilt middleware](https://docs.langchain.com/oss/python/langchain/middleware/built-in) support the same conclusion: Deep Agents is an agent harness built on the normal tool-calling loop, with planning/todo, filesystem, subagents, memory, sandbox/HITL, and middleware.

## Core Difference Surfaces

| Surface | Why It Matters |
|---|---|
| System prompt | Defines work style, persistence, verification behavior, when to ask users, when to use todo, when to delegate, and when to stop. |
| Tool surface | Defines the model-visible action space and work-management surface. This includes action tools such as read/edit/shell/browser, and work-management tools such as todo, plan, notes, and task files. |
| Memory, skills, and long-lifecycle state | Determines what persists across turns or sessions, what reusable procedures can be loaded, and how prior context affects current work. |

These three surfaces are the most visible and easiest to compare across
systems. They explain much of the difference between a plain tool-calling agent,
a coding agent, and a personal agent.

After checking public material, this three-part view is still the best
high-level model for **agent experience**:

> Same model + same tool-call protocol does not make the same agent. The
> differentiator is the harness: prompt policy, tool surface, and
> memory/skills/lifecycle.

There are also important supporting infrastructure surfaces, but they are not
the main "what does this agent feel capable of doing" difference.

## Additional Important Surfaces

| Surface | Why It Matters |
|---|---|
| Permissions, sandbox, approval, and side-effect gates | Determines what actions are allowed, blocked, previewed, interrupted, or routed through human approval. The same tool name can have very different safety semantics. |
| Context assembly and message normalization | Determines how history, tool results, files, memory snippets, background tasks, errors, and summaries become model-visible context. This often becomes a major hidden product advantage. |
| Subagents, background work, and lifecycle | Determines whether work can be delegated, run in isolated context windows, continue in the background, wake later, or persist as a child session. |

These surfaces are less obvious than prompt and tools, but they are often where
production systems diverge most.

## Experience Surfaces And Infrastructure

| Surface | Layer | Examples | Notes |
|---|---|---|---|
| Prompt / policy | Experience core | system/developer prompt, CLAUDE.md, AGENTS.md, workflow instructions | Shapes behavior, communication style, persistence, verification, and stopping rules. |
| Tool surface | Experience core | read/edit/shell/browser/search/message; todo, plan, notes, task files | The main capability surface. Includes both action tools and work-management tools. |
| Memory / skills / lifecycle | Experience core | memory, skills, subagents, background tasks, hooks, cron/wake, sessions | Turns a one-turn loop into a durable product with reusable procedures and continuity. |
| Context assembly and compaction | Hidden infrastructure | project instructions, memory snippets, environment context, tool results, summaries, compaction | Decides what the model can see and remember under token budget. |
| Runtime gates and permissions | Safety infrastructure | sandbox modes, approval prompts, HITL middleware, owner-only tools, tool-call limits | Decides what the runtime allows regardless of model intent. |

This keeps the user-facing conclusion compact:

- **Prompt / policy** tells the model how to work.
- **Tool surface** tells the model what it can do and how it can manage work.
- **Memory / skills / lifecycle** tells the system how to persist, reuse, and
  continue across time.

The supporting infrastructure still matters, especially in production:

- **Hard runtime authority**: permission, sandbox, approval, retry, fallback,
  limits, and interruption behavior.
- **Context engineering**: which instructions, memories, files, summaries, and
  tool results are placed into model context, in what order, and under what
  budget.

Those are not just implementation details. They directly change model behavior,
cost, safety, and recoverability.

## Comparison Framing

| System Type | Likely Differentiator |
|---|---|
| Plain LangChain `create_agent` | Provider abstraction plus minimal model-tool loop. |
| Deep Agents | Default prompt, todo, filesystem, sandbox execution, subagents, summarization, skills, memory, and HITL middleware. |
| Claude Code / Codex-style coding agents | Mature coding tool surface, filesystem/shell integration, todo/plan UX, verification loops, permissions, and CLI/editor product shell. |
| OpenClaw / personal agents | Channel identity, personal memory, messaging/tool integrations, cron/wake/heartbeat, persistent sessions, and owner/security policies. |

## Main Takeaway

For modern model-driven tool-loop agents:

> With the same model, the meaningful differences mostly come from prompt,
> tool surface, and memory/skills/lifecycle.

This does not mean all agents are the same. It means the engineering center of
gravity is often outside the base loop. The base loop is becoming a commodity
protocol; the agent shell determines whether the system behaves like a coding
agent, a personal agent, or a research agent.

## Public Evidence By Surface

### 1. Prompt Policy Is Still A Major Differentiator

OpenAI's agent guide treats instructions as a core design foundation: clear
actions, edge cases, and decomposed routines reduce ambiguity. The Codex loop
post shows that Codex has bundled model-specific instructions and additional
developer/user instruction layers. Claude Code uses CLAUDE.md, rules, skills,
and subagent prompts. Deep Agents also ships opinionated system prompts.

The practical conclusion is that prompt is not "just prompt engineering." In
agent products, prompt is the soft policy layer that tells the model how to use
the runtime.

### 2. Tool Surface Is Agent-Computer Interface Design

Anthropic's ACI framing is directly relevant: the model's reliability depends
on tool names, descriptions, argument schemas, path semantics, result format,
and examples. OpenAI similarly divides tools into data, action, and
orchestration tools. Codex exposes tools like shell, plan update, web search,
and MCP tools through the model API tool list. LangChain middleware can inject
tools such as `write_todos`, shell, filesystem, file search, and subagent task
tools.

The tool surface includes two important subtypes:

| Tool Type | Examples | Role |
|---|---|---|
| Action tools | read, edit, shell, browser, search, message | Let the agent act on the outside world. |
| Work-management tools | TodoWrite, `update_plan`, `write_todos`, notes, task files | Let the agent manage progress, attention, and user-visible work state. |

This supports a stronger claim:

> The tool list is not only capability exposure. It is also a behavioral prior.

Changing `edit_file` semantics, shell sandbox behavior, or result shape can
change the agent as much as changing the system prompt.

### 3. Runtime Gates Are The Difference Between Guidance And Control

Claude Code documents permission modes, MCP scoping, hooks, and managed
settings. Codex documents approval/sandbox instructions as part of prompt
construction and enforces shell sandboxing at runtime. LangChain exposes
HumanInTheLoop, model/tool call limits, PII middleware, fallback, retry, and
other hooks.

So there are two layers:

- **Model-visible guidance**: "ask before doing X."
- **Runtime authority**: "the call cannot execute unless policy allows it."

For business systems, this distinction matters. Anything involving side effects
should usually live in runtime authority, not only in prompt.

### 4. Context Assembly Is A Hidden Product Advantage

The Codex loop post makes prompt/input construction a first-class topic:
instructions, tools, AGENTS.md, skills, environment context, tool results, and
compaction are all part of the harness. Claude Code similarly has a detailed
memory hierarchy: CLAUDE.md, rules, auto memory, skills, and on-demand loading.
Deep Agents treats filesystem and summarization as context-management
infrastructure.

This means context assembly should be evaluated as an architecture surface, not
as incidental plumbing. It controls:

- what the model knows;
- what it forgets;
- which parts of prior work become durable;
- whether long runs degrade or stay coherent;
- how much token cost is spent on scaffolding.

### 5. Todo / Plan Tools Are Work-Management Tools

LangChain's public Deep Agents blog is unusually explicit here: it says the
core algorithm remains an LLM tool loop, and that stronger long-task behavior
comes from detailed prompts, a planning/todo tool, subagents, and filesystem.
LangChain's `TodoListMiddleware` adds `write_todos` plus prompt guidance.
Codex exposes an `update_plan` tool in the harness.

These tools are not side infrastructure. They are part of the core agent
experience because they affect how the agent organizes work, marks progress,
keeps attention, and exposes status to the user.

In common coding or research agents, todo/plan tools are usually still soft
control: the runtime stores and displays them, but the model remains the main
local decision-maker.

### 6. Long-Lifecycle Features Change Product Identity

Claude Code's features page groups persistent context, skills, MCP, subagents,
agent teams, hooks, and plugins as extension surfaces. OpenClaw adds personal
agent concerns: identity, channels, session routing, memory, cron/wake,
heartbeats, background subagents, and delivery. Deep Agents adds a reusable SDK
version of todo, filesystem, subagents, skills, memory, and sandbox/HITL.

This is why a coding agent, a personal agent, and a business workflow agent can
share a core loop but feel like different products. Their long-lifecycle shell
is different.

## Boundary With Workflow Discussion

This note should stay focused on general tool-loop agents. Programmatic
business workflow design should be discussed in the workflow section, not mixed
into this generic comparison.

The useful boundary is:

- For **general agents**, plan/todo usually appears as a work-management tool
  inside the broader tool surface.
- For **workflow systems**, plan can become part of the workflow design and
  execution model.

This keeps the external-agent section simple: Claude Code, Codex-style agents,
OpenClaw, and Deep Agents mainly differ in prompt/policy, tool surface, and
memory/skills/lifecycle. Workflow-specific planning can be introduced later
when discussing our own design.

## What This Means For Future Research

For the internal sharing document, the next useful research should not be "find
more named agent patterns." The better questions are:

1. How do top coding agents assemble model context before each turn?
2. How do they design file/edit/shell tools so the model makes fewer mistakes?
3. Which policies are prompt-only, and which are runtime-enforced?
4. How do they evaluate long-running agent quality beyond final answer quality?
5. Which work-management tools are enough for open-ended tasks, and when should
   planning move into a workflow architecture?

These questions connect external agent products back to the later workflow
discussion without forcing Dayfold's design into the generic tool-loop category.
