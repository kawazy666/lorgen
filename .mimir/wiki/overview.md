---
title: Mimir — overview
summary: What this repo is, what's inside .mimir/, and where to start reading.
updated: 2026-04-29
---

# Mimir — overview

Mimir is a Claude Code plugin that captures and retrieves the **"why"**
behind a codebase. The plugin's source (subagent + skill) lives at the
repo root in `agents/` and `skills/mimir/`. The data Mimir curates as
you use it lives in `.mimir/`.

This Wiki page is the top-level entry. Drill into:

- **[Architecture](architecture.md)** — high-level design and the trade-offs
  that produced the current shape (pointer to `docs/architecture.md`).
- **Knowledge index** — `.mimir/knowledge/` is grouped by `kind`:
  - `conventions/` — rules the codebase follows
  - `decisions/` — discrete choices with rationale
  - `runbooks/` — operational know-how
  - `facts/` — non-obvious context
  - `lessons/` — generalizable insights, often from incidents

## Plugin source (this repo, not the data store)

| Path | Tracked? | What |
|---|---|---|
| `.claude-plugin/plugin.json` | yes | Plugin manifest (name / version / etc.) |
| `agents/mimir.md` | yes | Subagent definition |
| `skills/mimir/*.md` | yes | Operating manual (multi-file skill, includes `metrics.md`) |
| `hooks/hooks.json` | yes | PreToolUse Write/Edit redaction guard |
| `bin/mimir-*` | yes | Pretool guard, redact check / wrapper, metrics compile |

## Data store — `.mimir/`

| Path | Tracked? | What |
|---|---|---|
| `config.yaml` | yes | Team-shared settings |
| `.gitignore` | yes | Excludes `cache/`, `state.json`, `logs/` |
| `knowledge/<kind>/*.md` | yes | Knowledge items |
| `wiki/*.md` | yes | Wiki pages (this directory) |
| `metrics/<YYYYMMDD>-<HHMMSS>-<id>.json` | **yes** | Compiled per-session aggregate (paths, counts, timestamps only) |
| `logs/<YYYYMMDD>-<HHMMSS>-<id>.jsonl` | no | Raw per-session events; gitignored. Compiled into `metrics/` at end of run. |
| `cache/` | no | Source / LLM response caches |
| `state.json` | no | Onboard cursor, indexing state |

### ADR mirror (outside `.mimir/`)

| Path | Tracked? | What |
|---|---|---|
| `docs/adr/<NNNN>-<slug>.md` | yes | Optional ADR mirrors of `decision`-kind Knowledge. Off by default; enable per record (`@mimir record --adr ...`) or globally (`outputs.write_adr: true`). |

## Reading order for a new contributor

1. `README.md` — what Mimir is and how to install via `/plugin install`
2. `docs/architecture.md` — why it's shaped this way
3. `skills/mimir/SKILL.md` — operating manual entry, with sub-files
4. Browse `.mimir/knowledge/` and `.mimir/wiki/` to learn what Mimir
   itself has accumulated about this repo
