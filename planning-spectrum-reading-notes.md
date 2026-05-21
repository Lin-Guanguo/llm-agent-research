# Planning Spectrum Reading Notes

Last Updated: 2026-05-19

## Purpose

Working notes for the next research part on Agent planning, workflow, and control-flow primitives. This file captures reading impressions before updating the main synthesis.

## Current Impression

The most useful framing so far comes from:

1. OpenAI, *A practical guide to building agents*
2. Anthropic, *Building effective agents*

Both sources are useful because they do not treat "agent architecture" as a fixed list of named patterns. They instead make architecture a matter of choosing the right orchestration shape for the task complexity.

## OpenAI Practical Guide

Source: https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/

Useful framing:

- Start with a single agent and evolve to multi-agent systems only when needed.
- Multi-agent systems are described through two broad orchestration categories:
  - Manager: agents as tools under a central manager.
  - Decentralized: handoffs between peer agents.
- The guide explicitly links architecture choice to complexity level and production guardrails.

Relevance to this research:

- The Manager pattern is close to Anthropic's Orchestrator-workers.
- Programmatic P&E can be interpreted as a compiled or deterministic variant of Manager / Orchestrator-workers:
  - LLM generates the plan.
  - Programmatic runtime validates, schedules, and executes.
  - Specialized workers are capability nodes rather than open-ended agents.

## Anthropic Building Effective Agents

Source: https://www.anthropic.com/engineering/building-effective-agents

Useful framing:

- The workflow / agent distinction is practical rather than metaphysical:
  - Workflows route LLMs and tools through predefined code paths.
  - Agents dynamically direct their own processes and tool usage.
- Orchestrator-workers is especially relevant: a central LLM dynamically breaks down tasks, delegates to workers, and synthesizes results.
- The article's broader advice is to choose the simplest pattern that solves the task.

Relevance to this research:

- The current production system resembles Orchestrator-workers in decomposition shape, but differs in coordinator authority.
- Current form:
  - Planner LLM creates structured plan.
  - Deterministic coordinator handles dependency readiness, scheduling, and execution.
- Anthropic form:
  - Orchestrator LLM repeatedly decides delegation and synthesis.

The important distinction is not whether workers exist, but whether mid-run coordination remains model-mediated or becomes programmatic.

## Working Synthesis

Agent architecture should be described as a combination of control primitives, not as a fixed menu of named patterns.

The current working claim:

> Early Agent development was a discovery period for control structures. The field explored tools, plans, routers, memory, multi-agent delegation, validators, and loops under many names. These primitives are now clearer. The next competition is business-specific composition, model-fit, and evaluation quality.

For the programmatic P&E direction:

> Todo-list planning is likely the mainstream planning interface for general-purpose agents. Programmatic P&E is better understood as a business-runtime strategy: externalize the plan into a typed contract, let deterministic code validate and schedule it, and spend model calls where semantic judgment is actually needed.

## Follow-Up Questions

1. Where does Anthropic's Orchestrator-workers end and programmatic P&E begin?
2. Is OpenAI's Manager pattern best understood as LLM-mediated Manager, while programmatic P&E is compiled Manager?
3. Which sources provide evidence that todo-list planning is becoming the default for general-purpose agents?
4. Which sources provide evidence that executable plans are still useful for safety, cost, and stability?
5. What evaluation method would prove that programmatic P&E is worth the extra engineering complexity for a specific business domain?
