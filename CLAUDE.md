# LLM Agent Architecture Research — Claude Instructions

See [README.md](./README.md) for full repository structure.

## Repository Purpose

This repository studies **Agent control-flow architectures** — who decides what happens next in an Agent execution, and how different frameworks/products/platforms answer that question.

## Relationship to Sister Repositories

This repo is part of a research family:

1. **`~/dev/llm-memory-research`** (sister) — covers the orthogonal axis of memory and context management. Agent CLI research (Claude Code, Codex, Cursor, OpenClaw, Gemini CLI) lives there and is cross-referenced from here.
2. **`~/dev/CyberMnema/timeline/`** (parent context) — the user's work log. Originals of some Agent architecture research files live there and should not be moved; this repo references them by path.

When a question involves "how does Agent X handle memory/context," check `llm-memory-research` first. When it involves "how does Agent X decide control flow, compose steps, or validate state," check this repo.

## Research Conventions

- Each framework/product gets one `<name>.research.md` file at the repo root
- Each research file follows the format: **Sources → Overview → numbered sections → Conclusion / Architecture Position**
- All code claims must cite file paths and line numbers from source
- Distinguish **marketing claims** from **code-level verification** — never blur the two
- Cross-repo references use absolute paths: `/Users/linguanguo/dev/llm-memory-research/<file>`

## Writing Style

- English for research files (consistent with llm-memory-research)
- Avoid emojis in research files
- Use tables for any comparison that has ≥3 rows
- Quote official documentation exactly when citing definitions

## Blog Output

**Blogs are NOT written in this repo.** They are drafted in `~/dev/CyberMnema/timeline/YYYY/MM/W{n}/blog-<topic>-YYYY-MM-DD.md`, following the user's existing convention. This repo only holds research artifacts that blogs reference by path.

## What NOT to Put Here

- Memory / context management research → goes in `llm-memory-research`
- User's daily work logs → go in `CyberMnema/timeline/`
- Product feature lists or price comparisons — this is a technical architecture study, not a buyer's guide
