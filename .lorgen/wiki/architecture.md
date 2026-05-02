---
title: Architecture
parent: overview
summary: Pointers into Lorgen's design rationale (docs/architecture.md is the canonical record).
updated: 2026-05-02
---

# Architecture

The canonical, full-length record of Lorgen's design decisions and the
alternatives considered lives in [`docs/architecture.md`](../../docs/architecture.md).

This Wiki page is a short pointer + index. For a Lorgen-using codebase
**the architecture decisions belong in Knowledge `decisions/` and ADR
files under `outputs.adr_dir`** (default `.lorgen/adr/`), not here —
Wiki pages are summaries / indexes, not the source of truth.

## Key decisions (links into the canonical record)

- **Claude Code plugin distribution, not Python CLI or standalone install** —
  see `docs/architecture.md` → "Why a Claude Code plugin"
- **Single repo serves as both marketplace and plugin** —
  see `docs/architecture.md` → "Distribution"
- **`agents/` + `skills/lorgen/` + `skills/review/` + `commands/` at
  repo root, `.lorgen/` for data only** — see `docs/architecture.md`
  → "What Lorgen is"
- **In-repo `.lorgen/`, not per-user global store** — see
  `docs/architecture.md` → "Why `.lorgen/` lives inside the user repo"
- **ADRs default to `.lorgen/adr/`** (override via `outputs.adr_dir`) —
  see `docs/architecture.md` → "Why no commits from Lorgen"
- **Lorgen never commits** — see `docs/architecture.md` → "Why no commits
  from Lorgen"
- **ripgrep + LLM selection, not vector search** — see
  `docs/architecture.md` → "Why ripgrep + LLM selection, not vector search"
- **Read = accumulate** — see `docs/architecture.md` → "Why 'read = accumulate'"
- **Raw logs gitignored, compiled metrics tracked** — see
  `docs/architecture.md` → "Why raw logs are gitignored, compiled metrics
  are tracked"
- **Mechanical secret guard via PreToolUse hooks** — see
  `docs/architecture.md` → "Why mechanical secret guard via PreToolUse hooks"
- **4-role multi-agent review (Coordinator + Searcher + parallel
  Investigators + Comparator) running in main agent context** — see
  `docs/architecture.md` → "Why a 4-role multi-agent team for review"
- **`/lorgen:review` is the only entry point — `@lorgen review` is
  unsupported** (Claude Code 2.x subagents cannot call `Task`) — see
  `docs/architecture.md` → "Why `/lorgen:review` and not `@lorgen
  review`"

## Related Knowledge

The same rationale, scoped to retrieval-friendly Knowledge items:

- [decision: pivot to subagent](../knowledge/decisions/pivot-to-subagent.md)
- [decision: package as plugin](../knowledge/decisions/package-as-plugin.md)
- [decision: rename Mimir → Lorgen](../knowledge/decisions/rename-mimir-to-lorgen.md)
- [decision: ADR default location inside `.lorgen/`](../knowledge/decisions/adr-default-location-inside-lorgen.md)
- [decision: review as main-agent skill](../knowledge/decisions/review-as-main-agent-skill.md)
- [decision: 4-role multi-agent review team](../knowledge/decisions/four-role-multi-agent-review-team.md)
- [decision: single-repo marketplace and plugin](../knowledge/decisions/single-repo-marketplace-and-plugin.md)
- [fact: Aegis (DAG-based context compiler) comparison](../knowledge/facts/aegis-comparison.md)
