---
description: OpenClaw agent design notes and comparison with Claude Code-style agents
last_updated: 2026-05-21
---

# OpenClaw Agent Design

This topic records the discussion about OpenClaw's agent architecture from the
perspective of agent control primitives.

Primary research note:

- `/Users/linguanguo/dev/llm-agent-research/openclaw-agent-design.case.md`

## Summary

OpenClaw's inner agent loop is close to a standard model-tool loop, backed by Pi
agent sessions. Its distinguishing value is the outer runtime shell: session
routing, channel/account identity, tool policy, sandboxing, cron/heartbeat,
subagents, delivery, and approval boundaries.

Compared with Claude Code-style coding agents, OpenClaw is less centered on a
coding workspace and more centered on long-running personal-agent operation.

## Key Points

- The core loop is standard: model chooses tools, runtime returns observations,
  and the next model turn continues.
- OpenClaw's default tools include personal-agent surfaces such as messaging,
  sessions, subagents, cron/wake, gateway, nodes, TTS, browser, web fetch/search,
  image, and PDF.
- Memory is model-visible mainly through `memory_search` and `memory_get`.
  Internally the memory subsystem is more complex, but normal recall timing is
  prompt-driven rather than scheduler-driven.
- Skills are exposed as an available-skills listing plus reading the selected
  `SKILL.md`, unlike Claude Code's more explicit `SkillTool` expansion path.
- OpenClaw's plan remains soft/model-mediated. The runtime has strong hard
  gates around identity, tools, subprocesses, subagents, cron, heartbeat, and
  approvals, but it does not appear to use a typed executable plan as the main
  task contract.

## Relevance

For the internal Agent sharing document, OpenClaw is useful as the
personal-agent product case. It contrasts with Dayfold Agent, whose core
pressure is stable business task execution through typed plans, deterministic
scheduling, typed capabilities, and dataflow bindings.

For the current Agent architecture research, OpenClaw and Claude Code-style
agents are not very different at the inner-loop level. Both are model-driven
tool loops with soft planning and optional subagent delegation. OpenClaw's
distinctive part is the longer-lived outer lifecycle: session/identity routing,
context continuity, tool policy, side-effect gates, subagent registry,
cron/heartbeat/wake, and delivery back to the right channel or session.

UI and IM channels should not be treated as core agent intelligence. UI is
mostly observability and interaction. IM channels matter when they define
identity, authorization, session routing, and delivery boundaries.
