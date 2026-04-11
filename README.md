# LLM Agent Architecture Research

Last Updated: 2026-04-11

A systematic research project studying **Agent control-flow architectures** — how different frameworks, products, and platforms decide who controls execution (the LLM, the developer, or a visual workflow), and what that means for production systems.

Sister repository to [llm-memory-research](https://github.com/Lin-Guanguo/llm-memory-research), which covers the orthogonal axis of memory and context management.

## Scope

This repository focuses on the **control flow** dimension:

- **Who decides what happens next?** — LLM autonomy vs. programmatic workflow vs. hybrid
- **Is there a Plan artifact?** — Pre-execution planning vs. per-step reactive decisions
- **How are multiple steps composed?** — ReAct loop, Plan-and-Execute, Workflow DAG, Multi-Agent handoff
- **What validation exists?** — Runtime schema checks, typed dataflow, programmatic gating

Out of scope (covered elsewhere):
- Memory/context management → [llm-memory-research](https://github.com/Lin-Guanguo/llm-memory-research)
- Model capability benchmarks
- Product feature comparisons

## Related CyberMnema Files

This repository grew out of Agent architecture research originally done in `CyberMnema`. Those files remain in place as historical work logs and are cross-referenced here:

- `timeline/2026/04/W15/Agent架构调研-2026-04-07.md` — Initial survey of Agent paradigms and 50-year dataflow lineage (~40k chars)
- `timeline/2026/04/W15/Agent复杂度取舍-2026-04-08.md` — Cognition vs MAST evidence triangulation on multi-agent complexity

## Research Directions

1. **Code-First Agent Frameworks** — Mastra, LangGraph, Vercel AI SDK, Google ADK, Alibaba ADK/AgentScope, OpenAI Assistants
2. **Domestic (Chinese) Low-Code Agent Platforms** — Dify, Coze, Alibaba Bailian, Tencent Yuanqi, FastGPT, Baidu AppBuilder, Volcengine AgentKit
3. **Production Agent Systems** — Personal Agent 2.0 (Plan-and-Execute + typed dataflow)
4. **Coding CLI Agents** (cross-referenced) — Claude Code, Codex, Cursor, Devin, Manus (detailed research in `llm-memory-research`)

## Summary Documents

| File | Scope | Content |
|------|-------|---------|
| [summary.md](./summary.md) | All directions | Full synthesis: the Agent Map, control-flow taxonomy, cross-project findings |
| [findings.md](./findings.md) | Cross-project | Key insights that span multiple categories |

## Research Files

| File | Target | Status |
|------|--------|--------|
| [mastra.research.md](./mastra.research.md) | Mastra TS framework (agent architecture, not memory) | ✅ Done |
| [domestic-platforms.research.md](./domestic-platforms.research.md) | 7 Chinese Agent platforms (Dify, Coze, Bailian, etc.) | ✅ Done |
| `google-adk.research.md` | Google ADK Python (`github.com/google/adk-python`) | 📋 Planned |
| `alibaba-adk.research.md` | Alibaba Bailian ADK / AgentScope | 📋 Planned |
| `langgraph.research.md` | LangChain LangGraph | 📋 Planned |
| `vercel-ai-sdk.research.md` | Vercel AI SDK | 📋 Planned |
| `openai-assistants.research.md` | OpenAI Assistants/Responses API | 📋 Planned |
| `my-agent-2.0.research.md` | Personal production system (P&E + typed dataflow) | 📋 Planned |

## Blog Output

Blogs drafted from this research live in the user's main work directory `~/dev/CyberMnema/`, following the existing `blog-*.md` convention. This repo only holds research artifacts.

| Blog | Location | Status |
|------|----------|--------|
| 到底什么是 Agent | `/Users/linguanguo/dev/CyberMnema/timeline/2026/04/W15/blog-到底什么是Agent-2026-04-11.md` | 📝 Outline ready, drafting next |

## Repository Structure

```
llm-agent-research/
├── README.md                              # This file
├── CLAUDE.md                              # Instructions for Claude
├── summary.md                             # Full synthesis
├── findings.md                            # Cross-project findings
│
├── mastra.research.md                     # Mastra agent architecture
├── domestic-platforms.research.md         # Chinese platforms (7 products)
├── google-adk.research.md                 # (planned)
├── alibaba-adk.research.md                # (planned)
├── langgraph.research.md                  # (planned)
├── vercel-ai-sdk.research.md              # (planned)
├── openai-assistants.research.md          # (planned)
└── my-agent-2.0.research.md               # (planned)
```

## License

Personal research project.
