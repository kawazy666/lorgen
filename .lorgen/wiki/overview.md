---
title: Lorgen — overview
summary: What this repo is, what's inside .lorgen/, and where to start reading.
updated: 2026-04-29
---

# Lorgen — overview

Lorgen is a Claude Code plugin that captures and retrieves the **"why"**
behind a codebase. The plugin's source (subagent + skill) lives at the
repo root in `agents/` and `skills/lorgen/`. The data Lorgen curates as
you use it lives in `.lorgen/`.

This Wiki page is the top-level entry. Drill into:

- **[Architecture](architecture.md)** — high-level design and the trade-offs
  that produced the current shape (pointer to `docs/architecture.md`).
- **Knowledge index** — `.lorgen/knowledge/` is grouped by `kind`:
  - `conventions/` — rules the codebase follows
  - `decisions/` — discrete choices with rationale
  - `runbooks/` — operational know-how
  - `facts/` — non-obvious context
  - `lessons/` — generalizable insights, often from incidents

## Plugin source (this repo, not the data store)

| Path | Tracked? | What |
|---|---|---|
| `.claude-plugin/plugin.json` | yes | Plugin manifest (name / version / etc.) |
| `agents/lorgen.md` | yes | Subagent definition |
| `skills/lorgen/*.md` | yes | Operating manual (multi-file skill, includes `metrics.md`) |
| `hooks/hooks.json` | yes | PreToolUse Write/Edit redaction guard |
| `bin/lorgen-*` | yes | Pretool guard, redact check / wrapper, metrics compile |

## Data store — `.lorgen/`

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

### ADR mirror

| Path | Tracked? | What |
|---|---|---|
| `.lorgen/adr/<NNNN>-<slug>.md` | yes | Optional ADR mirrors of `decision`-kind Knowledge. Default location (Lorgen-curated, lives inside `.lorgen/`). Off by default; enable per record (`@lorgen record --adr ...`) or globally (`outputs.write_adr: true`). Override `outputs.adr_dir` to e.g. `docs/adr/` if your repo follows that convention. |

## Reading order for a new contributor

1. `README.md` — what Lorgen is and how to install via `/plugin install`
2. `docs/architecture.md` — why it's shaped this way
3. `skills/lorgen/SKILL.md` — operating manual entry, with sub-files
4. Browse `.lorgen/knowledge/` and `.lorgen/wiki/` to learn what Lorgen
   itself has accumulated about this repo
