---
title: Lorgen — overview
summary: What this repo is, what's inside .lorgen/, and where to start reading.
updated: 2026-05-02
---

# Lorgen — overview

Lorgen is a Claude Code plugin that captures and retrieves the **"why"**
behind a codebase. The plugin's source (subagent + review skill +
slash command + hooks) lives at the repo root. The data Lorgen
curates as you use it lives in `.lorgen/`.

The repo doubles as a **Claude Code marketplace and the single plugin
it lists** — install via `/plugin marketplace add kawazy666/lorgen`
followed by `/plugin install lorgen@lorgen` (the right-hand
`lorgen` is the marketplace name, not `owner/repo`). See
[decisions/single-repo-marketplace-and-plugin.md](../knowledge/decisions/single-repo-marketplace-and-plugin.md).

This Wiki page is the top-level entry. Drill into:

- **[Architecture](architecture.md)** — high-level design and the trade-offs
  that produced the current shape (pointer to `docs/architecture.md`).
- **Knowledge index** — `.lorgen/knowledge/` is grouped by `kind`:
  - `conventions/` — rules the codebase follows
  - `decisions/` — discrete choices with rationale (see recent: rename
    Mimir→Lorgen, ADR location, review skill, 4-role team, marketplace)
  - `runbooks/` — operational know-how
  - `facts/` — non-obvious context (see: aegis-comparison)
  - `lessons/` — generalizable insights, often from incidents

## Plugin source (this repo, not the data store)

| Path | Tracked? | What |
|---|---|---|
| `.claude-plugin/plugin.json` | yes | Plugin manifest (name / version / etc.) |
| `.claude-plugin/marketplace.json` | yes | Marketplace manifest pointing at this same repo as `source: "./"` |
| `agents/lorgen.md` | yes | Subagent definition |
| `skills/lorgen/*.md` | yes | Operating manual (multi-file skill, includes `metrics.md`) |
| `skills/review/SKILL.md` | yes | Knowledge-grounded multi-agent code review skill (loaded by main agent, not the Lorgen subagent) |
| `commands/review.md` | yes | `/lorgen:review` slash command — entry point for review |
| `hooks/hooks.json` | yes | PreToolUse Write/Edit redaction guard |
| `bin/lorgen-*` | yes | Pretool guard, redact check / wrapper, metrics compile |
| `assets/logo.png` | yes | Project logo (Spiral Logic AI) shown in README header |

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

1. `README.md` — what Lorgen is and the 2-step install
   (`/plugin marketplace add` → `/plugin install`)
2. `docs/architecture.md` — why it's shaped this way (long-form,
   alternatives considered)
3. `skills/lorgen/SKILL.md` — operating manual entry for the curator
   subagent, with sub-files (`schema`, `retrieval`, `accumulation`,
   `sources`, `onboard`, `logging`, `metrics`)
4. `skills/review/SKILL.md` — operating manual for the
   `/lorgen:review` multi-agent review (Coordinator + Searcher +
   parallel Investigators + Comparator)
5. Browse `.lorgen/knowledge/decisions/` for the why-history of
   Lorgen's own design (Lorgen dogfoods on itself)
