# Mastra Agent Architecture Research

Last Updated: 2026-04-11

> **Research Methodology**: Source-level analysis of the Mastra monorepo at `/Users/linguanguo/dev/llm-memory-research/mastra/mastracode` (Git submodule of `github.com/mastra-ai/mastra`). This complements the earlier `llm-memory-research/mastra.research.md` which covered Observational Memory only — this file covers the Agent execution architecture.

## Sources

**Core Agent Files**
- `packages/core/src/agent/agent.ts:147-4773` — Main Agent class and generate/stream API
- `packages/core/src/agent/agent.types.ts:134-337` — AgentConfig interface definition
- `packages/core/src/agent/workflows/prepare-stream/index.ts:46-142` — Agent execution workflow composition
- `packages/core/src/agent/workflows/prepare-stream/stream-step.ts:37-106` — Stream invocation to LLM loop

**Execution Loop**
- `packages/core/src/llm/model/model.loop.ts:101-346` — MastraLLMVNext.stream() method
- `packages/core/src/loop/loop.ts:11-161` — Instantiates execution model output
- `packages/core/src/loop/workflows/stream.ts:16-260` — workflowLoopStream() ReadableStream creation
- `packages/core/src/loop/workflows/agentic-loop/index.ts:20-259` — Multi-iteration .dowhile() loop control
- `packages/core/src/loop/workflows/agentic-execution/index.ts:12-136` — Single ReAct iteration structure

**Workflow & Step Definitions**
- `packages/core/src/workflows/workflow.ts:156-200` — createStep() and Agent integration API
- `packages/core/src/workflows/types.ts` — Step and Workflow type definitions

**Examples**
- `examples/basics/agents/system-prompt/index.ts` — Basic single-agent example
- `examples/basics/agents/multi-agent-workflow/index.ts:7-77` — Sequential agent workflow composition

---

## 1. Agent Definition (API Layer)

Mastra's **Agent** is defined as a TypeScript class in `packages/core/src/agent/agent.ts:147-250`:

```typescript
export class Agent<
  TAgentId extends string = string,
  TTools extends ToolsInput = ToolsInput,
  TOutput = undefined,
  TRequestContext extends Record<string, any> | unknown = unknown,
> extends MastraBase {
```

**AgentConfig** (types.ts:134-337) requires:
- `id`, `name`: identification
- `instructions`: string, array, or function returning SystemMessage
- `model`: MastraModelConfig or array of models with fallback support (types.ts:156-200)
- `tools`: Record of Mastra/Vercel AI SDK tools
- Optional: `memory`, `workspace`, `voice`, `agents`, `scorers`, `inputProcessors`, `outputProcessors`

The Agent exposes two main execution methods (agent.ts:4659-4820):

1. **`generate(messages, options?)`** — Non-streaming completion, returns `FullOutput<OUTPUT>`
2. **`stream(messages, options?)`** — Streaming completion, returns `MastraModelOutput<OUTPUT>` (ReadableStream wrapper)

### Minimal Example

```typescript
const agent = new Agent({
  id: 'cat-expert',
  name: 'Cat Expert',
  instructions: 'You are a helpful cat expert...',
  model: openai('gpt-4o-mini'),
  tools: { catFact },
});

const result = await agent.generate('Tell me a cat fact');
```

---

## 2. Execution Model: Pure ReAct Loop

Mastra implements a **pure ReAct (Reasoning + Acting) loop** — the agent iteratively decides to generate text or call tools, based on the LLM's own output. **There is no separate planning phase and no Plan artifact.**

The execution chain is:

1. User calls `agent.generate(messages)` or `agent.stream(messages)` (agent.ts:4686, 4802)
2. Private `#execute()` (agent.ts:4108) creates a Workflow
3. Workflow calls `workflowLoopStream()` (loop/workflows/stream.ts:16)
4. `workflowLoopStream` creates a `ReadableStream<ChunkType>` that invokes `createAgenticLoopWorkflow()`
5. The agentic loop runs a `.dowhile()` (agentic-loop/index.ts:72), calling `createAgenticExecutionWorkflow()` per iteration
6. Each iteration: LLM call → tool calls (parallel) → tool results → check stop condition → loop or stop

### The Agentic Execution Step Chain

In `packages/core/src/loop/workflows/agentic-execution/index.ts:82-136`, a single ReAct iteration is a Workflow with this step chain:

```typescript
return createWorkflow(...)
  .map(async capture response model message count)
  .then(llmExecutionStep)
  .map(async add response messages to messageList)
  .map(async extract toolCalls from output)
  .foreach(toolCallStep, { concurrency: sequentialExecutionRequired ? 1 : toolCallConcurrency })
  .then(llmMappingStep)        // merge tool results back into messages
  .then(isTaskCompleteStep)    // check if task complete
  .commit();
```

Key steps:
- **llmExecutionStep**: Calls LLM, returns StepResult with text, toolCalls, finishReason
- **toolCallStep**: Executes each tool in parallel (or serially if `requireApproval` or `suspendSchema`)
- **llmMappingStep**: Adds tool results to message history
- **isTaskCompleteStep**: Evaluates Mastra's structured check for task termination

### The Loop Control (`dowhile`)

In `packages/core/src/loop/workflows/agentic-loop/index.ts:72-257`, the outer loop is a `.dowhile()`:

```typescript
.dowhile(agenticExecutionWorkflow, async ({ inputData }) => {
  accumulatedSteps.push(currentStep);

  if (rest.stopWhen && typedInputData.stepResult?.isContinued && accumulatedSteps.length > 0) {
    const hasStopped = conditions.some(condition => condition);
    hasFinishedSteps = hasFinishedSteps || hasStopped;
  }

  if (rest.onIterationComplete) {
    const iterationResult = await rest.onIterationComplete(iterationContext);
    if (iterationResult) {
      // Handle feedback, continue/stop flags
    }
  }

  return typedInputData.stepResult?.isContinued ?? false;
})
```

**Loop termination conditions**:
1. `stepResult.isContinued = false` (LLM returned `finishReason='stop'`)
2. `stopWhen` callback returns true (default: `stepCountIs(5)`, max 5 turns)
3. User-provided `onIterationComplete` callback returns `{ continue: false }`
4. Tool approval required and human declines

---

## 3. Workflow Concept

**Workflow** (`packages/core/src/workflows/workflow.ts`) is a **DAG execution engine** for deterministic, step-wise tasks. Agents are wrapped as Steps in Workflows.

- **Workflow**: `.then(step1).then(step2).parallel([step3, step4]).foreach(step5)...commit()`
- **Agent**: Embedded in a Workflow Step via `createStep(agent, options)`

### Agent + Workflow Composition

From `examples/basics/agents/multi-agent-workflow/index.ts:60-71`:

```typescript
const myWorkflow = createWorkflow({
  id: 'my-workflow',
  inputSchema: z.object({ topic: z.string() }),
  outputSchema: z.object({ finalCopy: z.string() }),
});

const copywriterStep = createStep({
  id: 'copywriterStep',
  inputSchema: ...,
  outputSchema: ...,
  execute: async ({ inputData }) => {
    const result = await copywriterAgent.generate(`Create a blog post about ${inputData.topic}`);
    return { copy: result.text };
  },
});

myWorkflow.then(copywriterStep).then(editorStep).commit();
```

**Can Agents call Workflows?** Yes, via a Tool wrapper:

```typescript
const weatherWorkflowTool = createTool({
  id: 'weather-workflow',
  execute: async (input) => {
    const workflow = mastra.getWorkflow('weather-workflow');
    const run = await workflow.createRun();
    const result = await run.start({ inputData: input });
    return result.result;
  }
});

const agent = new Agent({
  tools: { weatherWorkflowTool },
  ...
});
```

### Workflow Operators

- **`.then(step)`** — Sequential execution
- **`.parallel([s1, s2])`** — Concurrent execution
- **`.foreach(step, { concurrency })`** — Loop with concurrency control
- **`.dowhile(step, condition)`** — Conditional loop (used internally for ReAct iteration)
- **`.map(fn)`** — Transform data between steps
- **`.branch(condition, [paths])`** — Conditional branching

---

## 4. Agent × Workflow Organization

### Multi-Agent Patterns

**Pattern 1: Sequential Pipeline** (example: multi-agent-workflow)

```
User Input
  → CopywriterAgent (generate draft)
  → EditorAgent (edit draft)
  → Output
```

Each agent is wrapped in a Workflow step and chained.

**Pattern 2: Agent-to-Agent Delegation**

From agent.ts:418-450, Agents support an `agents` config:

```typescript
const mainAgent = new Agent({
  id: 'main',
  agents: {
    researcher: researcherAgent,
    writer: writerAgent,
  },
  tools: {
    delegate_to_researcher: createTool({
      execute: async (query) => {
        const agents = await mainAgent.listAgents();
        return agents.researcher.generate(query);
      }
    }),
  },
  ...
});
```

The `listAgents()` method (agent.ts:419-450) returns agents either statically or via a dynamic function.

**Handoff mechanism**: Agents call sub-agents by explicitly invoking them (via tools), not via implicit message passing.

---

## 5. Control & Predictability Mechanisms

### `stopWhen` & `maxSteps`

From `llm/model/model.loop.ts:104`, `model.loop.types.ts`, and `agentic-loop/index.ts:124-137`:

```typescript
const llm = agent.stream({
  messages,
  maxSteps: 5,
  stopWhen: stepCountIs(5),
  onIterationComplete: async (ctx) => {
    console.log(ctx.iteration, ctx.text, ctx.toolCalls, ctx.isFinal);
    return { continue: true };
  }
});
```

**Stop conditions**:
- **`stepCountIs(N)`** — Stop after N LLM turns (default: 5)
- **Custom `stopWhen` function** — Inspect accumulated steps, return boolean
- **`onIterationComplete`** — Human-in-the-loop control per iteration
- **Task completion detection** — `isTaskCompleteStep` checks if task is complete

### Input/Output Processors

From `agent.ts:304-314` and `agent/types.ts:304-314`:

```typescript
const agent = new Agent({
  inputProcessors: [
    {
      id: 'sanitize-input',
      processInputStep: async (input) => {
        return { part: input, blocked: false };
      }
    }
  ],
  outputProcessors: [
    {
      id: 'filter-output',
      processOutputResult: async (result) => {
        return { part: result, blocked: result.text.includes('forbidden') };
      }
    }
  ],
  maxProcessorRetries: 3,
});
```

Processors run **before/after** LLM execution:
- **Input processors**: Before any LLM call
- **Output processors**: After LLM generates response (run on each chunk)

If a processor blocks, it triggers a retry loop with a feedback message back to the LLM (`agentic-loop/index.ts:171-196`).

### Resumable Execution

From `agent.ts:4108` and `loop/workflows/stream.ts:196-207`:

```typescript
const { runId, suspendPayload } = await agent.stream({
  messages,
  requireToolApproval: true,
});

await agent.resumeNetwork(resumeData, { runId });
```

Workflows support `.suspend()` (step.ts) and `.resume()` (workflow.ts), allowing suspension for human-in-the-loop or external async operations.

### No Pre-Execution Plan

**Unlike a Plan-and-Execute design**, Mastra **does not produce a plan artifact**. The LLM generates text and tool calls directly each turn. Stopping depends on:
- LLM's own stop signal (`finishReason='stop'`)
- External `stopWhen` condition
- `onIterationComplete` hook
- Processor feedback loop

There is **no explicit plan structure** (no "Plan" message type, no separate planning phase, no pre-execution cost prediction).

---

## 6. Multi-Agent Support

### Configuration

From `agent.ts:278-279` and `agent.types.ts:276-279`:

```typescript
agents?: DynamicArgument<Record<string, Agent>>;
```

### Handoff Mechanism

**There is no built-in "agent-to-agent message passing"**. Instead:
- Sub-agents are invoked via **explicit tool calls**
- Only the **final result** of the sub-agent is returned (not the message history)

This matches the failure pattern described in Cognition's *Don't Build Multi-Agents*: agent A calls agent B, gets back only agent B's text output, loses the intermediate reasoning and steps.

**Mitigation**: Wrap sub-agent calls in Tools that include the full message history or step trace in their output. Mastra does not provide this by default.

---

## 7. Typing & Validation

### Structured Output

From `agent.ts:4674-4735` and `agent/types.ts:72-118`:

```typescript
const result = await agent.generate(messages, {
  structuredOutput: {
    schema: z.object({
      sentiment: z.enum(['positive', 'negative', 'neutral']),
      score: z.number().min(0).max(1),
    }),
  }
});
```

Internal mechanism (`agent.ts:4154-4168`):
- If schema is provided, Mastra wraps it to handle null→undefined transforms for OpenAI strict mode
- `StructuredOutputProcessor` (an internal agent) ensures schema compliance
- Returns `GenerateObjectResult` with an `object` field

### Request Context Validation

From `agent.ts:382-407` and `agent/types.ts:336`:

```typescript
const agent = new Agent({
  requestContextSchema: z.object({
    userId: z.string(),
    tier: z.enum(['free', 'premium']),
  }),
  ...
});

await agent.generate(messages, {
  requestContext: new RequestContext({ userId: 'user-123', tier: 'premium' })
});
```

Validation occurs at the start of `generate()`/`stream()` (agent.ts:4688, 4804), before any execution.

### No Explicit "Typed Dataflow"

Validation occurs at:
1. Agent input (requestContextSchema)
2. Tool boundaries (tool input schemas)
3. Workflow step boundaries (step inputSchema/outputSchema)
4. Processor boundaries (custom validation logic)

But there is **no type-level guarantee** that a tool's output will match the next step's input schema. Runtime validation only.

---

## 8. Difference from Coding Agent (Claude Code)

Both Mastra and Claude Code use an iterative LLM loop where the model decides what to do next based on its own reasoning. The key differences:

| Dimension | Mastra Agent | Claude Code (Coding Agent) |
|-----------|-------------|--------------------------|
| **Scope** | Generic: any tools, any domain | Specialized: code/file editing + execution |
| **Tool Semantics** | User-defined via `createTool()` | Built-in: `write_file`, `read_file`, `bash`, `python`, etc. |
| **State Management** | Message history + optional memory system | File system + execution state |
| **Planning** | None (pure ReAct) | ReAct with implicit planning through tool use |
| **Stopping** | `stepCountIs(N)`, `onIterationComplete` | Error, max iterations, user abort |
| **Memory Persistence** | Optional: `@mastra/memory` (observational) | Implicit: file system history |
| **Multi-Agent** | Via explicit tool calls | Single agent per context |
| **Suspension/Resume** | Built-in (workflow suspend/resume) | Not built-in (requires external checkpoint) |

**Conclusion**: They are **instances of the same ReAct pattern**, not fundamentally different architectures. The distinction is **tool domain** (generic vs. code-specialized), not **execution model**.

---

## 9. Representative Examples

### Example 1: Basic Agent with Tool Use

From `examples/basics/agents/system-prompt/index.ts`:

```typescript
const catFact = createTool({
  id: 'Get cat facts',
  inputSchema: z.object({}),
  execute: async () => {
    const response = await fetch('https://catfact.ninja/fact');
    return { catFact: (await response.json()).fact };
  },
});

const agent = new Agent({
  id: 'cat-expert',
  name: 'Cat Expert',
  instructions: 'You are a helpful cat expert. Always use the catFact tool...',
  model: openai('gpt-4o-mini'),
  tools: { catFact },
});

const result = await agent.generate('Tell me a cat fact');
```

**User profile**: Casual tool user; expects agent to use tool autonomously.
**What it solves**: Factual consistency by forcing tool use in instructions.

### Example 2: Sequential Multi-Agent Workflow

From `examples/basics/agents/multi-agent-workflow/index.ts:7-77`:

```typescript
const copywriterAgent = new Agent({ id: 'copywriter', ... });
const editorAgent = new Agent({ id: 'editor', ... });

const myWorkflow = createWorkflow({
  inputSchema: z.object({ topic: z.string() }),
  outputSchema: z.object({ finalCopy: z.string() }),
})
  .then(createStep({
    execute: async ({ inputData }) => {
      const result = await copywriterAgent.generate(`Create a blog post about ${inputData.topic}`);
      return { copy: result.text };
    },
  }))
  .then(createStep({
    execute: async ({ inputData }) => {
      const result = await editorAgent.generate(`Edit the following: ${inputData.copy}`);
      return { finalCopy: result.text };
    },
  }))
  .commit();
```

**User profile**: Pipeline builder orchestrating multiple AI agents for complex tasks.
**What it solves**: Deterministic, reproducible multi-stage processing. Agent A's output → Agent B's input, with type safety at Step boundaries.

---

## 10. Position on the Agent Architecture Spectrum

**Mastra is a generic, ReAct-based Agent framework with Workflow orchestration, optional memory, and processor-based validation — designed for composing deterministic multi-stage AI systems where agents are steps in a DAG, not autonomous entities.**

```
Generic ReAct          Workflow DAG           Code-Specific
      ↓                      ↓                       ↓

[P&E + Typed Dataflow] ← [Mastra] ← [Claude Code]
  ↑                          ↑                 ↑
  Plan artifact              Workflow as       File system,
  Strict validation          main line         code-focused
  Programmatic gates         ReAct inside      tool surface
                             steps
```

**Mastra's position**:
- **More structured than Claude Code** (Workflow DAG outer composition, processor validation)
- **Less structured than a P&E + typed dataflow system** (no Plan artifact, no pre-execution cost prediction, no flow-level types)
- **Closer to "enterprise framework" than specialized product** — Claude Code is a complete agent product; Mastra is a toolkit

## Key Architectural Insights

1. **Pure ReAct with Workflow Overlay**: Mastra's core is ReAct (LLM decides each step), but execution is wrapped in a Workflow DAG for composition and determinism.

2. **No Plan Artifact**: No "Plan" message type, no pre-execution cost prediction, no separate planning phase. Stopping relies on loop conditions plus LLM signals.

3. **Suspension-Based Human-in-the-Loop**: Via `.suspend()` on steps and `.resume()` on runs — more flexible than an approval gate but less structured.

4. **Processor-Based Validation**: Input/Output processors act as middleware layers for transformation and blocking, avoiding hard-coded if/then logic in agent instructions.

5. **Tool-Based Multi-Agent Handoff**: Sub-agents are called explicitly via tools; no implicit message-passing. This is the exact Cognition "Don't Build Multi-Agents" failure mode.

6. **Type Safety at Boundaries**: Zod schemas define Agent I/O, Tool I/O, Step I/O — but not the internal dataflow. Runtime validation catches schema mismatches, no compile-time guarantee.

---

## Conclusion

Mastra is a **generic, composable ReAct framework optimized for production determinism via Workflows**, not for pre-execution planning or cost prediction. Its strength is flexibility and developer ergonomics. Its weakness is the lack of structured execution planning and full observability into pre-execution cost/steps.

**On the Agent Map taxonomy**: Mastra sits firmly in category **④a (Code Framework, ReAct school)** — despite its Workflow DAG layer giving it a more structured appearance than Claude Code, its fundamental execution model remains per-step LLM decision-making, not pre-execution planning. Structurally, it is the TypeScript code-first version of Dify.
