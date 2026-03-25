# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> See `AGENTS.md` for full architecture, coding conventions, environment variables, and release process.

## Commands

```bash
bun install          # install dependencies
bun start            # run interactive CLI
bun dev              # run with watch mode
bun test             # run tests (Bun's built-in runner)
bun run typecheck    # type-check with tsc

# Evals (requires LANGSMITH_API_KEY)
bun run src/evals/run.ts            # full eval suite
bun run src/evals/run.ts --sample 10 # random sample of 10

# WhatsApp gateway
bun run gateway:login  # link WhatsApp account (QR scan)
bun run gateway        # start gateway
```

## Architecture

**Entry point:** `src/index.tsx` → `src/cli.ts` (Ink/React terminal UI)

**Agent loop** (`src/agent/agent.ts`): iterative tool-calling loop (default max 10 iterations). Emits typed events (`tool_start`, `tool_end`, `thinking`, `answer_start`, `done`) consumed by the CLI for real-time rendering. Final answer is generated in a separate LLM call with no tools bound.

**Context management:** When token count exceeds threshold, oldest tool results are dropped, keeping only the most recent `KEEP_TOOL_USES` entries. On context overflow errors (from the provider), retries with a reduced `OVERFLOW_KEEP_TOOL_USES=3`.

**Tools** (`src/tools/registry.ts`): Tools are conditionally registered based on env vars. `web_search` uses Exa → Perplexity → Tavily in priority order. `x_search` requires `X_BEARER_TOKEN`. Rich per-tool descriptions are injected into the system prompt (not just the tool schema).

**Skills** (`src/skills/`): Extensible workflows defined as `SKILL.md` files with YAML frontmatter. Discovered at startup via `src/skills/registry.ts`. Each skill runs at most once per query. Add new skills by dropping a `SKILL.md` into a subdirectory under `src/skills/`.

**Memory** (`src/memory/`): Vector-based persistent memory using embeddings. Embedding provider priority: OpenAI → Gemini → Ollama. Memory is flushed/indexed asynchronously after queries. Tools: `memory_search`, `memory_get`, `memory_update`.

**LLM abstraction** (`src/model/llm.ts`): Provider detected by model name prefix (`claude-` → Anthropic, `gemini-` → Google, `grok-` → xAI, `deepseek-` → DeepSeek, `kimi-` → Moonshot, `openrouter:` → OpenRouter, `ollama:` → Ollama). Anything else falls through to OpenAI. **OpenRouter models require the `openrouter:` prefix** (e.g. `openrouter:meta-llama/llama-3.3-70b-instruct:free`) — without it the request is sent to OpenAI instead. Anthropic calls use `cache_control` on the system prompt for prompt caching. Config persisted in `.dexter/settings.json`.

**Debug logs:** Every query writes a JSONL scratchpad to `.dexter/scratchpad/<timestamp>.jsonl` with all tool calls, results, and reasoning steps.

## Financial Datasets API — Free Tier Limits

The free tier of financialdatasets.ai only covers **AAPL, NVDA, and MSFT**. Requests for any other ticker (SPY, QQQ, VIX, GEV, etc.) will silently fail — Dexter displays "Called 0 data sources" when all sub-tool calls return errors. The agent falls back to web scraping in this case, which still produces analysis but without structured financial data. A paid plan is required for full ticker coverage.

## Recommended Free Setup

| Component | Free Option |
|-----------|------------|
| LLM | `gemini-2.0-flash` via Google AI Studio (aistudio.google.com) — 1500 req/day, 15 RPM |
| Web search | Exa (exa.ai) or Tavily (tavily.com) — both have free tiers |
| Financial data | Free for AAPL, NVDA, MSFT only; paid plan for other tickers |

Use `gemini-2.0-flash` over Pro models on the free tier — Pro has 2 RPM vs Flash's 15 RPM.
