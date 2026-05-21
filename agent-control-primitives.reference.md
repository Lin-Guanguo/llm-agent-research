# Agent Control Primitives

Last Updated: 2026-05-19

## Purpose

This note records a useful framing prompted by the article *从0开发大模型的17种Agent架构演进详细拆解* and the underlying `all-agentic-architectures` project.

The article is useful as a teaching map, but its 17-pattern breakdown should not be treated as a mutually exclusive taxonomy. Many of the listed "architectures" are composable control primitives at different abstraction levels.

## Sources

- [从0开发大模型的17种Agent架构演进详细拆解](https://mp.weixin.qq.com/s/5f0I2apY4oFsHrttANBOJg) (accessed: 2026-05-19)
  - Direct automated fetch failed for the WeChat page. The title/source are discoverable through a mirror: [tool.lu article mirror](https://tool.lu/es_ES/article/7PH/detail) (accessed: 2026-05-19).
- [FareedKhan-dev/all-agentic-architectures](https://github.com/FareedKhan-dev/all-agentic-architectures) (accessed: 2026-05-19)
- [Agno: Running Workflows](https://docs.agno.com/workflows/running-workflows) (accessed: 2026-05-19)

## Short Original Excerpts

The article's most useful claim is that agent architecture is not mainly about prompt tricks:

> "Agent architecture 的本质不是 prompt engineering"

It then compresses the diagnostic method into three questions:

> "新增了什么 state？" / "router？" / "evaluator？"

This note takes that idea seriously, but reorganizes the 17 patterns into a smaller set of composable engineering primitives.

## Assessment

The article's conclusion is valuable: real agent systems evolve by making state, routing, evaluation, memory, side effects, and stopping conditions more explicit.

The 17-pattern list is less convincing as a taxonomy. Several entries are not peer categories:

- `Reflection` and `Self-Improvement` are evaluation loops.
- `Tool Use` and `Dry-Run` are effect-boundary patterns.
- `Planning` and `PEV` are control-flow externalization patterns.
- `Multi-Agent`, `Blackboard`, `Meta-Controller`, and `Ensemble` are work-allocation patterns.
- `Episodic Memory`, `Semantic Memory`, and `Graph Memory` are state-persistence patterns.
- `Tree of Thoughts`, `Mental Loop`, and `Cellular Automata` are search, simulation, or decentralized-computation strategies.

Real systems usually combine these. Treating them as 17 separate architectures can hide the actual engineering question: which control primitive did the system add, and what failure mode does it address?

## Higher-Level Primitives

| Primitive | Question It Answers | Common Forms | Failure Mode It Controls |
|---|---|---|---|
| State | What does the system explicitly know? | messages, blackboard, plan, port values, memory, trace | hidden assumptions, context loss, uninspectable progress |
| Control Flow | Who decides the next step? | fixed DAG, router, loop, scheduler, plan interpreter, LLM orchestrator | drift, deadlock, endless loops, wrong sequencing |
| Work Allocation | Who does the work? | single agent, capability workers, agents-as-tools, handoff, ensemble | role conflict, over-broad prompts, poor specialization |
| Effect Boundary | How does the system touch the world? | tools, sandbox, dry-run, approval, permissions | unsafe side effects, tool misuse, irreversible actions |
| Evaluation | How does the system know whether to continue, retry, stop, or reject? | validator, verifier, critic, judge, hard constraints | silent failure propagation, premature completion |
| Search / Simulation | How does the system explore alternatives? | ToT, MCTS, mental loop, simulator, counterfactual execution | local greediness, irreversible trial-and-error |
| Persistence / Recovery | How does the system survive time and failure? | checkpoint, event log, long-term memory, replayable trace | losing progress, non-recoverable partial execution |
| Boundary / Escalation | How does the system know it should not act? | self-model, confidence gate, human escalation, policy guardrail | overconfident action in high-risk domains |

## Mapping The 17 Patterns To Primitives

| Pattern From Article | Better Interpreted As | Main Primitive(s) |
|---|---|---|
| Reflection | Minimal quality loop | Evaluation |
| Tool Use | External effect interface | Effect Boundary |
| ReAct | Online observe-act loop | Control Flow + Effect Boundary |
| Planning | Plan externalized as state | State + Control Flow |
| PEV | Verification inserted into the main loop | Evaluation + Control Flow |
| Multi-Agent | Role decomposition | Work Allocation |
| Blackboard | Shared state plus dynamic scheduling | State + Control Flow + Work Allocation |
| Meta-Controller | One-shot entry routing | Control Flow + Work Allocation |
| Ensemble | Parallel redundancy | Work Allocation + Evaluation |
| Episodic / Semantic Memory | Long-term recall | Persistence / Recovery |
| Graph / World-Model Memory | Relational state model | State + Persistence / Recovery |
| Tree of Thoughts | Search over reasoning paths | Search / Simulation |
| Mental Loop | Counterfactual simulation before action | Search / Simulation + Effect Boundary |
| Dry-Run Harness | Side-effect preview and approval | Effect Boundary + Evaluation |
| Metacognitive Agent | Capability-boundary reasoning | Boundary / Escalation |
| Self-Improvement Loop | Iterative quality optimization | Evaluation + Persistence / Recovery |
| Cellular Automata | Decentralized local-rule computation | Search / Simulation + Control Flow |

## Relevance To Programmatic P&E

The current planning-spectrum research should avoid saying "our system is one of the 17 patterns." A better claim is:

> Programmatic Plan-and-Execute is a particular combination of primitives: explicit typed state, LLM-generated plan, deterministic scheduler, capability workers, plan validation, bounded replan, and checkpointable execution.

That combination differs from todo-mediated ReAct:

| Dimension | Todo-Mediated ReAct | Programmatic P&E |
|---|---|---|
| State | Todo list as model-visible working memory | Typed plan, port values, step state |
| Control Flow | LLM decides the next action each turn | Program scheduler interprets the plan |
| Work Allocation | Tools or subagents are called opportunistically | Capability workers are selected by the plan |
| Evaluation | Mostly soft, model-mediated | Hard validators plus targeted verifiers |
| Cost Profile | Higher repeated reasoning cost | Higher upfront design cost, lower runtime token cost |
| Best Fit | Open-ended general tasks | Bounded business workflows requiring one-shot stability |

## Writing Takeaway

For a future article introducing agents, this can become a section:

> Do not understand agent architectures as a menu of fixed patterns. Understand them as a set of composable control primitives.

This framing absorbs the useful part of the 17-pattern article while avoiding its main weakness: presenting composable mechanisms as if they were separate architecture species.

The practical diagnostic is:

1. What state became explicit?
2. What router or scheduler was added?
3. What evaluator or stop condition was added?
4. What side effect was gated?
5. What recovery path became possible?

If a proposed "new agent architecture" cannot answer these questions, it is probably a renamed combination of existing primitives rather than a new architecture.
