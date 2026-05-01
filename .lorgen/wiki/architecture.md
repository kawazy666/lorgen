---
title: Architecture
parent: overview
summary: Pointers into Lorgen's design rationale (docs/architecture.md is the canonical record).
updated: 2026-04-29
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
- **`agents/` + `skills/lorgen/` at repo root, `.lorgen/` for data only** —
  see `docs/architecture.md` → "What Lorgen is"
- **In-repo `.lorgen/`, not per-user global store** — see
  `docs/architecture.md` → "Why `.lorgen/` lives inside the user repo"
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

## Related Knowledge

The same rationale, scoped to retrieval-friendly Knowledge items:

- [decision: pivot to subagent](../knowledge/decisions/pivot-to-subagent.md)
- [decision: package as plugin](../knowledge/decisions/package-as-plugin.md)
