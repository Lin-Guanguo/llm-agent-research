# Case: Claude Code Dynamic Workflows

Last Updated: 2026-06-01

## Purpose

This note records the control-flow significance of Claude Code Dynamic
Workflows: Claude writes a JavaScript orchestration script, then a runtime
executes that script across many subagents.

The research question is:

> Did Claude Code move from a pure turn-by-turn coding-agent loop toward a
> runtime-authored workflow model, and what does the generated JavaScript look
> like?

Short answer: yes, for workflow-triggered tasks. Dynamic Workflows add a
second execution mode beside the ordinary model-tool loop. The model still
authors the plan, but the plan is no longer only a todo or conversation
artifact. It becomes JavaScript code that holds loops, branching, fan-out,
intermediate results, verification votes, and final synthesis.

## Sources

Official sources:

- [Claude Code Docs: Dynamic workflows](https://code.claude.com/docs/en/workflows) (accessed: 2026-06-01), mirrored at `official-sources/claude-dynamic-workflows-2026-05-31/claude-code-workflows.md`.
- [Claude Code Docs: Run agents in parallel](https://code.claude.com/docs/en/agents) (accessed: 2026-06-01), mirrored at `official-sources/claude-dynamic-workflows-2026-05-31/claude-code-agents.md`.
- [Claude Code Docs: Tools reference](https://code.claude.com/docs/en/tools-reference.md) (accessed: 2026-06-01), mirrored at `official-sources/claude-dynamic-workflows-2026-05-31/claude-code-tools-reference.md`.
- [Introducing dynamic workflows in Claude Code](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code) (accessed: 2026-06-01).
- Download manifest: `official-sources/claude-dynamic-workflows-2026-05-31/README.md`.

Public workflow scripts and secondary analysis:

- [Bun PR #30412: Rewrite Bun in Rust](https://github.com/oven-sh/bun/pull/30412) (accessed: 2026-06-01).
- [Bun `.claude/workflows` at merge commit `23427dbc12fdcff30c23a96a3d6a66d62fdc091d`](https://github.com/oven-sh/bun/tree/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows) (accessed: 2026-06-01).
- [Bun `phase-a-port.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-a-port.workflow.js) (accessed: 2026-06-01).
- [Bun `lifetime-classify.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/lifetime-classify.workflow.js) (accessed: 2026-06-01).
- [Bun `phase-h-diff-review.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-diff-review.workflow.js) (accessed: 2026-06-01).
- [azukiazusa.dev: Trying Dynamic Workflow in Claude Code](https://azukiazusa.dev/en/blog/claude-code-dynamic-workflow/) (accessed: 2026-06-01).
- [vblanco20-1/stb_image_jai `.claude/workflows`](https://github.com/vblanco20-1/stb_image_jai/tree/46c3ca7d2da333f7235c0615344abe224bb3acc3/.claude/workflows) (accessed: 2026-06-01).
- [QuintinShaw/pi-dynamic-workflows](https://github.com/QuintinShaw/pi-dynamic-workflows) (accessed: 2026-06-01). This is an unofficial reimplementation / extension, not Anthropic code.
- [alexanderop/defineworkflow](https://github.com/alexanderop/defineworkflow) (accessed: 2026-06-01). This is an unofficial workflow engine inspired by the same pattern.

## Core Finding

Claude Code Dynamic Workflows are best described as:

```text
Plan-as-code for coding agents:
Claude authors a JavaScript orchestration script;
the workflow runtime executes that script;
each agent() call spawns a normal Claude subagent session;
intermediate state stays in script variables;
only the final synthesized result returns to the parent conversation.
```

This is not the same as a hand-authored LangGraph-style graph. It is also not
the old Claude Code todo list. The plan is LLM-authored, but after approval the
runtime follows the script instead of asking the parent Claude turn by turn what
to spawn next.

The official docs make this control-transfer explicit. They define a workflow
as a JavaScript script written by Claude and run in the background by a runtime
(`official-sources/claude-dynamic-workflows-2026-05-31/claude-code-workflows.md:15`).
Their comparison table says the script decides what runs next and keeps
intermediate results in script variables, while subagents and skills leave
those decisions and results in Claude's context window
(`official-sources/claude-dynamic-workflows-2026-05-31/claude-code-workflows.md:28`).
The same page states the core architectural change directly: the workflow moves
the plan into code, and the script holds loops, branching, and intermediate
results so Claude's context holds only the final answer
(`official-sources/claude-dynamic-workflows-2026-05-31/claude-code-workflows.md:39`).

## Generated Script Shape

The most accurate source is the raw script itself, not a reconstructed
pseudocode block. Public examples show that the generated artifact is ordinary
JavaScript with a small injected orchestration API:

| Script | Exact evidence |
|---|---|
| Bun `phase-a-port.workflow.js` | Metadata header at [lines 1-9](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-a-port.workflow.js#L1-L9); `args`/path setup at [lines 11-29](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-a-port.workflow.js#L11-L29); Implement -> Verify -> Fix `pipeline` at [lines 148-208](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-a-port.workflow.js#L148-L208). |
| Bun `lifetime-classify.workflow.js` | Metadata header at [lines 1-9](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/lifetime-classify.workflow.js#L1-L9); Classify `pipeline` at [lines 79-107](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/lifetime-classify.workflow.js#L79-L107); 3-vote Verify `parallel` at [lines 111-143](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/lifetime-classify.workflow.js#L111-L143); Synthesize return object at [lines 150-165](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/lifetime-classify.workflow.js#L150-L165). |
| Bun `phase-h-diff-review.workflow.js` | Two reviewer agents per file and blocker filtering at [lines 47-83](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-diff-review.workflow.js#L47-L83); conditional fixer agent at [lines 84-100](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-diff-review.workflow.js#L84-L100). |

An accurate normalized shape, derived from those scripts, is:

```js
export const meta = { name, description, phases };

const input = args || {};
const items = normalize(input.items);

phase("Implement");
const results = await pipeline(
  items,
  item => agent(implementPrompt(item), { label, phase: "Implement", schema }),
  (impl, item) => agent(verifyPrompt(item, impl), { label, phase: "Verify", schema }),
  (verdict, item) => needsFix(verdict)
    ? agent(fixPrompt(item, verdict), { label, phase: "Fix", schema })
    : verdict,
);

return summarize(results);
```

This normalized block is not quoted from one file. It captures the observed
API shape across the linked real scripts.

Observed primitives:

| Primitive | Observed role |
|---|---|
| `export const meta` | Literal workflow metadata: name, description, phases. |
| `args` | Input supplied to the workflow invocation. |
| `phase(title)` | Progress grouping for the `/workflows` UI. |
| `log(message)` | Workflow-level progress note. |
| `agent(prompt, opts)` | Spawn a subagent; `opts.schema` asks for structured output. |
| `parallel(thunks)` | Run independent thunks concurrently and return ordered results. |
| `pipeline(items, ...stages)` | Run each item through staged async processing while items fan out. |
| Ordinary JS | Constants, functions, `for` loops, `if`, arrays, `Map`, `Set`, `JSON.stringify`, early returns. |

The Bun scripts show the same shape repeatedly. `phase-a-port.workflow.js`
starts with `export const meta`, declares JSON schemas, builds prompt functions,
and runs an Implement -> Verify -> Fix `pipeline` over `FILES`
([source](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-a-port.workflow.js#L1-L9),
[source](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-a-port.workflow.js#L31-L73),
[source](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-a-port.workflow.js#L148-L208)).

`lifetime-classify.workflow.js` classifies files in a pipeline, then runs a
three-vote adversarial verification pass, then synthesizes aggregate TSV output
([source](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/lifetime-classify.workflow.js#L79-L107),
[source](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/lifetime-classify.workflow.js#L111-L143),
[source](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/lifetime-classify.workflow.js#L150-L165)).

`phase-h-diff-review.workflow.js` uses two reviewer agents per changed file,
deduplicates findings in script state, filters for blocking severities, and
spawns a fixer only when blocking findings exist
([source](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-diff-review.workflow.js#L47-L83),
[source](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-diff-review.workflow.js#L84-L100)).

## Runtime Boundary

The official docs define the runtime boundary more clearly than the runtime
implementation. A workflow script runs in an isolated environment separate from
the conversation, and intermediate results remain in script variables
(`official-sources/claude-dynamic-workflows-2026-05-31/claude-code-workflows.md:187`).
The workflow itself has no direct filesystem or shell access; agents do the
reading, writing, and command execution
(`official-sources/claude-dynamic-workflows-2026-05-31/claude-code-workflows.md:193`).
The published limits are 16 concurrent agents and 1,000 total agents per run
(`official-sources/claude-dynamic-workflows-2026-05-31/claude-code-workflows.md:195`).

This boundary is visible in public scripts. The `stb_image_jai` porting workflow
explicitly comments that the workflow runner does not provide filesystem access,
so it uses a manifest-loader subagent to read `drafts.manifest.json`
([source](https://github.com/vblanco20-1/stb_image_jai/blob/46c3ca7d2da333f7235c0615344abe224bb3acc3/.claude/workflows/10-port-sections.workflow.js#L16-L18),
[source](https://github.com/vblanco20-1/stb_image_jai/blob/46c3ca7d2da333f7235c0615344abe224bb3acc3/.claude/workflows/10-port-sections.workflow.js#L47-L64)).

The official docs also state that launch approval and subagent permissions are
separate. The workflow launch prompt depends on permission mode, but workflow
subagents run in `acceptEdits` and inherit the tool allowlist
(`official-sources/claude-dynamic-workflows-2026-05-31/claude-code-workflows.md:145`,
`official-sources/claude-dynamic-workflows-2026-05-31/claude-code-workflows.md:166`).

## Resume And State

Official docs say the runtime tracks each agent result as the run progresses,
which enables same-session resume
(`official-sources/claude-dynamic-workflows-2026-05-31/claude-code-workflows.md:189`).
Stopping and resuming reuses completed agents' cached results and runs the rest
live, but exiting Claude Code starts a fresh workflow in the next session
(`official-sources/claude-dynamic-workflows-2026-05-31/claude-code-workflows.md:208`).

Unofficial reimplementations show a plausible implementation strategy, but
they should not be treated as Anthropic code. `pi-dynamic-workflows` caches
agent results by deterministic call index plus a call hash, uses a limiter for
concurrency, and injects runtime functions into a Node `vm` context
([source](https://github.com/QuintinShaw/pi-dynamic-workflows/blob/main/src/workflow.ts#L27-L33),
[source](https://github.com/QuintinShaw/pi-dynamic-workflows/blob/main/src/workflow.ts#L226-L239),
[source](https://github.com/QuintinShaw/pi-dynamic-workflows/blob/main/src/workflow.ts#L321-L368),
[source](https://github.com/QuintinShaw/pi-dynamic-workflows/blob/main/src/workflow.ts#L399-L429)).

This clone is useful because it makes the implied runtime design concrete:
parse metadata, execute a sandboxed script, inject `agent`/`parallel`/`pipeline`
as runtime capabilities, journal agent outputs, and replay completed calls on
resume. The official docs support the same high-level semantics, but not the
exact implementation details.

## Canonical Bun Examples

The strongest Bun examples are not small one-file pipelines. They are
coordination programs that a parent Claude Code conversation could try to
drive manually, but would be brittle because the parent context would have to
remember queue state, worktree boundaries, verification votes, patches, and
termination conditions across many turns. In these scripts, that state lives in
JavaScript variables and the runtime executes the next step.

All Bun script links in this section are from the merge-commit workflow tree
listed in Sources and were accessed on 2026-06-01.

| Workflow | Why it is a canonical workflow case | What the script owns |
|---|---|---|
| [`phase-d-build-queue.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-d-build-queue.workflow.js#L1-L13) | Treats `cargo build` as the work queue: survey the current compile frontier, fix per-file in parallel, re-survey, repeat until link. | Round state and early exit ([lines 70-89](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-d-build-queue.workflow.js#L70-L89)), frontier prioritization by unseen files and error count ([lines 92-101](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-d-build-queue.workflow.js#L92-L101)), and a per-file fix -> 2-vote verify -> bugfix pipeline ([lines 108-180](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-d-build-queue.workflow.js#L108-L180)). |
| [`phase-f-test-swarm.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-f-test-swarm.workflow.js#L1-L7) | Runs 24 isolated worktrees, one per test area, then converges their commits back into the main branch. | The area matrix ([lines 10-35](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-f-test-swarm.workflow.js#L10-L35)), parallel worktree agents ([lines 53-83](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-f-test-swarm.workflow.js#L53-L83)), and sequential cherry-pick convergence with conflict handling ([lines 85-122](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-f-test-swarm.workflow.js#L85-L122)). |
| [`phase-g-mega-swarm.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-g-mega-swarm.workflow.js#L1-L25) | Turns test triage into a repeated control system: one rebuild per round, broad test survey, up to 30 fix agents, two reviewers, correction application, then loop. | The per-round build gate ([lines 109-123](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-g-mega-swarm.workflow.js#L109-L123)), cached baseline/probe/classification survey ([lines 125-163](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-g-mega-swarm.workflow.js#L125-L163)), and fix -> 2-vote review -> apply pipeline with reward-hacking checks ([lines 170-248](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-g-mega-swarm.workflow.js#L170-L248)). |
| [`phase-h-dedup.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-dedup.workflow.js#L1-L10) | Performs whole-codebase deduplication at roughly "50M tokens" of reading, with no `git`/`cargo` until a final compile agent. | Thirty-ish shard agents for exhaustive candidate discovery ([lines 212-248](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-dedup.workflow.js#L212-L248)), cross-reference clustering ([lines 252-277](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-dedup.workflow.js#L252-L277)), 2-vote verification ([lines 279-316](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-dedup.workflow.js#L279-L316)), parallel edits, and one final compile/fix/commit gate ([lines 318-365](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-dedup.workflow.js#L318-L365)). |
| [`phase-h-unsafe-wrap.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-unsafe-wrap.workflow.js#L1-L10) | Converts a safety refactor into a staged audit: classify every `unsafe {}` by pattern, coalesce strategies, apply abstractions, review soundness, then compile. | Sharded classification over unsafe sites ([lines 202-232](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-unsafe-wrap.workflow.js#L202-L232)), strategy synthesis ([lines 234-278](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-unsafe-wrap.workflow.js#L234-L278)), broad application ([lines 280-315](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-unsafe-wrap.workflow.js#L280-L315)), 2-vote soundness review, and final compile/fix ([lines 317-371](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-unsafe-wrap.workflow.js#L317-L371)). |
| [`phase-h-main-parity.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-main-parity.workflow.js#L1-L10) | Tracks semantic parity between upstream `.zig` commits and the Rust port. This is bookkeeping-heavy and easy to lose in a normal conversation. | Surveying upstream commits ([lines 74-81](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-main-parity.workflow.js#L74-L81)), per-commit check plus 2-vote adversarial verification ([lines 83-116](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-main-parity.workflow.js#L83-L116)), and optional porting of missing changes and tests ([lines 118-137](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-main-parity.workflow.js#L118-L137)). |
| [`phase-h-windows-testfix.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-windows-testfix.workflow.js#L1-L9) | Batch-fixes Windows CI failures by grouping signatures, assigning each signature to an isolated worktree, and returning reviewed patches instead of direct commits. | Failure sharding from a results JSON ([lines 80-101](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-windows-testfix.workflow.js#L80-L101)), isolated fix agents that return `git diff` patches ([lines 103-133](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-windows-testfix.workflow.js#L103-L133)), and 2-vote read-only patch review ([lines 134-173](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-h-windows-testfix.workflow.js#L134-L173)). |
| [`phase-c-panic-swarm.workflow.js`](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-c-panic-swarm.workflow.js#L1-L8) | Runs a crash-finding feedback loop: link, probe a command suite in parallel, deduplicate by panic location, then fix each unique root cause in parallel. | The link/probe/fix loop ([lines 91-112](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-c-panic-swarm.workflow.js#L91-L112)), parallel probes with structured panic extraction ([lines 113-131](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-c-panic-swarm.workflow.js#L113-L131)), failure deduplication ([lines 142-150](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-c-panic-swarm.workflow.js#L142-L150)), and parallel root-cause fixes ([lines 152-183](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/phase-c-panic-swarm.workflow.js#L152-L183)). |

The practical lesson is that Dynamic Workflows make Claude Code better at
"control-plane" work: durable queues, fan-out/fan-in, multi-round convergence,
isolation boundaries, and independent verification. They do not make each
subtask deterministic; each `agent()` call is still a model-driven coding task.
The improvement is that the orchestration no longer depends on the parent
model repeatedly remembering and re-deciding the whole run.

## Quality Patterns In The Scripts

Dynamic Workflows are not just "run many agents." The interesting move is that
the script can encode repeatable quality patterns that would otherwise require
the parent model to remember and coordinate across turns.

Common patterns observed:

| Pattern | Concrete shape | Evidence |
|---|---|---|
| Fan-out / fan-in | One agent per file, shard, source, or claim, then flatten and aggregate. | `phase-a-port.workflow.js` runs one implement/verify/fix pipeline per file. |
| Structured subagent output | JSON Schema on each `agent()` call, then ordinary JS consumes fields. | `phase-a-port.workflow.js` defines implement, verify, and fix schemas before the pipeline. |
| Adversarial verification | Independent verifier agents try to refute or reject candidate outputs. | `lifetime-classify.workflow.js` runs three verification votes per selected classification. |
| Conditional fixer | Spawn fixer agents only when verifier findings pass a severity gate. | `phase-h-diff-review.workflow.js` filters blocking findings before calling a fixer. |
| Iterative repair loop | Normal JS `for` loop surveys current failures, fixes a frontier, verifies, rebuilds, and retests. | `50-fix-divergences.workflow.js` loops across rounds and returns early when clean ([source](https://github.com/vblanco20-1/stb_image_jai/blob/46c3ca7d2da333f7235c0615344abe224bb3acc3/.claude/workflows/50-fix-divergences.workflow.js#L113-L160), [source](https://github.com/vblanco20-1/stb_image_jai/blob/46c3ca7d2da333f7235c0615344abe224bb3acc3/.claude/workflows/50-fix-divergences.workflow.js#L196-L253)). |
| External-state probing through agents | Because the script has no direct FS/shell access, a probe agent reads files or runs commands and returns structured state. | `10-port-sections.workflow.js` uses a manifest-loader agent. |

The built-in `/deep-research` workflow shown by azukiazusa.dev follows the same
pattern: scope the question, run multiple search agents, fetch and extract
claims, run three-vote adversarial verification, and synthesize only surviving
claims. This is a strong secondary source for the built-in command's actual raw
script, but it is not an official Anthropic code publication.

## Architecture Position

Dynamic Workflows change the earlier Claude Code control-flow classification:

| Mode | Who decides next? | Plan artifact | Runtime role |
|---|---|---|---|
| Ordinary Claude Code loop | Parent Claude, turn by turn. | Todo list, plan-mode document, conversation state. | Guard tools, normalize results, continue the model-tool loop. |
| Subagent delegation | Parent Claude decides when to spawn workers. | Subagent prompt and optional subagent definition. | Run child agent loops and return compact results. |
| Dynamic Workflow | JavaScript script decides next after launch. | LLM-authored JS orchestration script. | Execute script, spawn subagents, cache results, expose progress/resume. |
| Dayfold-style typed plan | Runtime schedules typed steps after validation. | Typed execution plan / DAG. | Validate, bind, schedule, execute, recover. |

This places Dynamic Workflows between ordinary coding-agent loops and fully
typed business workflow systems.

It is more executable than a todo list because the runtime follows actual code.
It is less typed than Dayfold-style plan execution because most task semantics
still live in natural-language prompts inside `agent()` calls. The outer control
flow is programmatic; the inner work remains model-driven.

One concise label:

```text
LLM-authored orchestration code around model-driven subagents.
```

This is close to "plan as macro tool argument," but the macro argument is a JS
program rather than a typed JSON plan.

## Reliability Notes

Evidence reliability:

- High confidence: official docs for product semantics, limits, permission
  model, and high-level runtime behavior.
- High confidence for observed script shape: public `.workflow.js` files in the
  Bun PR and other repositories.
- Medium confidence: azukiazusa.dev's raw-script screenshots/text for
  `/deep-research`; useful, but secondary.
- Low-to-medium confidence for runtime internals: unofficial clones such as
  `pi-dynamic-workflows` and `defineworkflow`; useful to understand one
  plausible implementation, not proof of Anthropic internals.

Specific caveat: public workflow scripts show ordinary JavaScript flexibility.
For example, Bun's `lifetime-classify.workflow.js` uses `Math.random()` for
sampling high-confidence classifications
([source](https://github.com/oven-sh/bun/blob/23427dbc12fdcff30c23a96a3d6a66d62fdc091d/.claude/workflows/lifetime-classify.workflow.js#L111-L114)).
Some unofficial clones forbid nondeterministic calls to support durable replay
([source](https://github.com/QuintinShaw/pi-dynamic-workflows/blob/main/src/workflow.ts#L131-L132),
[source](https://github.com/QuintinShaw/pi-dynamic-workflows/blob/main/src/workflow.ts#L452-L459)).
Do not infer that Anthropic's current runtime has exactly the clone's
determinism rules.

## Open Questions

- Is the actual Claude Code workflow runtime implemented as a Node/V8 sandbox,
  a custom interpreter, or another embedded JS runtime?
- What exact globals are available beyond the observed `agent`, `parallel`,
  `pipeline`, `phase`, `log`, and `args`?
- How are structured outputs enforced inside subagents: prompt-only, tool-based
  structured output, schema repair loop, or a model API feature?
- Are saved workflows guaranteed source-compatible across Claude Code versions,
  or is the JS DSL still preview-only and unstable?
- Does ultracode generate multiple workflows sequentially through the parent
  loop, or can workflows call/nest other workflows in official Claude Code?

## Takeaways

1. Dynamic Workflows are the clearest example so far of Claude Code turning an
   LLM-authored plan into executable runtime code.
2. The generated artifact is not a declarative DAG. It is JavaScript with
   injected orchestration primitives and ordinary control flow.
3. The workflow script owns loops, branching, fan-out, verification votes,
   intermediate aggregation, and final synthesis.
4. Subagents remain ordinary Claude sessions with tools; the script coordinates
   them and keeps their intermediate outputs outside the parent context.
5. This narrows the gap between coding agents and workflow-centric systems, but
   it does not make Claude Code a typed business workflow runtime. The hard task
   semantics still sit in prompts and schema-shaped subagent outputs.
