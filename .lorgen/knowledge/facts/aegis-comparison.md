---
trigger: "Aegis, RAG vs DAG, Exocortex, Zenn fuwasegu, context compiler, deterministic retrieval"
kind: fact
scope: repo
tags: [meta, comparison, retrieval, future]
sources:
  - {type: user-record, ref: "Zenn 2026-03-26: https://zenn.dev/yumemi_inc/articles/a61de3467bc182"}
  - {type: user-record, ref: "Aegis repo: https://github.com/fuwasegu/aegis"}
  - {type: user-record, ref: "Exocortex repo: https://github.com/fuwasegu/exocortex"}
---

# Fact — Aegis (DAG-based context compiler) comparison

## What Aegis is

Aegis (ふわせぐ / yumemi_inc, Zenn 2026-03-26) is an MCP server that
supplies AI coding agents with documentation by **graph-resolving**
dependencies (SQLite + recursive CTE on a DAG of doc relationships)
rather than embedding-based RAG. Reported impact on a 140+ UseCase
Laravel project:

- Tokens **~12× reduction** (10.3k → 1.8k)
- Tool calls **90% reduction** (55 → 6)
- Latency **3.5× faster** (152s → 43s)

Predecessor **Exocortex** (same author) used KùzuDB + vectors + a
"painful memory" priority score; Aegis is the author's pivot away
from RAG-style retrieval after running into stability/cost issues.

## Architectural ideas worth noting (not yet adopted in Lorgen)

| Aegis idea | Where Lorgen currently differs |
|---|---|
| DAG of doc dependencies; deterministic resolution from a file path | Lorgen retrieval is `ripgrep + LLM ranking` — probabilistic |
| `Agent Surface` (read-only, 6 tools) vs `Admin Surface` (24 mgmt tools) physically separated | Lorgen's `@lorgen` subagent holds read + write + delete in one surface |
| "Observation → Proposal → Human approval" pipeline before knowledge-base mutation | Lorgen's `accumulation.md` Stage 4 writes directly after the LLM gate |
| `deploy-adapters` injects a `<!-- aegis:start -->` block into `CLAUDE.md` | Lorgen has no adapter into the host repo's `CLAUDE.md` (cold-start coverage gap) |
| Distributed as MCP server (`npx`) | Lorgen distributed as Claude Code plugin (`/plugin install`) |

## Lessons that may inform Lorgen's roadmap

1. **Determinism beats relevance ranking for code-context retrieval**
   when the dependency graph is knowable. Worth experimenting: add
   `requires:` / `extends:` to Knowledge front-matter, build a small
   DAG, optionally bypass LLM ranking when DAG resolution returns a
   complete set.
2. **Proposal/approval > direct write** for AI-curated knowledge
   bases. A `.lorgen/proposals/` staging area committed by humans
   would slot cleanly into the existing accumulation pipeline as a
   new Stage 4'.
3. **Surface separation** is a force-multiplier when an agent has
   destructive capability. Lorgen's `/lorgen:review` already split
   review out of the main subagent; the same pattern can extend to
   write operations (read agent vs admin agent).
4. **`CLAUDE.md` injection** is a free win for cold-start agents
   that can't / won't load the full Lorgen subagent.

## Status: observed, not yet adopted

These are noted as future-design inputs. No code or config changes
have been made in response to this comparison as of this Knowledge
item's creation. Decisions to adopt (or explicitly NOT adopt) any of
these patterns will get their own `decisions/` records.
