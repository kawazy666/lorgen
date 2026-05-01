# `.lorgen/` schema

Authoritative spec for the directory layout, file front-matter, and
configuration keys. **Read this before reading or writing any `.lorgen/` file.**

## Directory layout

```
<repo-root>/
└── .lorgen/
    ├── config.yaml             # tracked
    ├── .gitignore              # tracked, excludes cache/ + logs/ + state.json
    ├── knowledge/              # tracked — small, trigger-retrievable items
    │   ├── conventions/<slug>.md
    │   ├── decisions/<slug>.md
    │   ├── runbooks/<slug>.md
    │   ├── facts/<slug>.md
    │   └── lessons/<slug>.md
    ├── wiki/                   # tracked — coarser per-area pages
    │   ├── overview.md
    │   ├── architecture.md
    │   └── modules/<area>.md
    ├── metrics/                # TRACKED — per-session aggregate (paths, counts, timestamps only)
    │   └── <YYYYMMDD>-<HHMMSS>-<invocation_id>.json
    ├── logs/                   # ignored — raw JSONL, per-machine, source for metrics compile
    │   └── <YYYYMMDD>-<HHMMSS>-<invocation_id>.jsonl
    ├── cache/                  # ignored
    │   ├── sources/            # raw PR / issue / ADR fetches
    │   │   ├── pr/<num>.json
    │   │   └── issue/<num>.json
    │   └── llm/                # response cache for repeated questions
    └── state.json              # ignored — onboarding cursor, last-scanned commits
```

What goes in `.lorgen/.gitignore` (write this exact content):

```
cache/
state.json
logs/
```

**Why logs are gitignored, metrics are tracked**: raw events
(`logs/<session>.jsonl`) include `query`, `candidate_summary`, source
content — anything PR/issue/comment authors wrote. They may carry
secrets or PII. Metrics (`metrics/<session>.json`) carry only paths,
counts, and timestamps — privacy by aggregation.

Per-session naming makes both merge-safe; we still gitignore raw logs
because aggregating into metrics happens before commit, and the
information loss is intentional. See `metrics.md` for the compile
contract.

## Knowledge item — front-matter schema

`.lorgen/knowledge/<category>/<slug>.md`

```markdown
---
trigger: "Decimal, money, float, monetary calculation"
kind: convention
scope: repo
tags: [money, billing]
sources:
  - {type: pr, ref: "#42", url: "https://github.com/owner/repo/pull/42"}
  - {type: commit, ref: "abc1234", url: "https://github.com/owner/repo/commit/abc1234"}
  - {type: adr, path: ".lorgen/adr/0003-decimal.md"}
created: 2026-04-29
updated: 2026-04-29
---

## Why we use Decimal for money

`monetary` 値は `decimal.Decimal` を必須とする。

2026-01 の incident — float の累積誤差で請求額が 1 円ズレた。
PR #42 で `src/billing/` 全体を Decimal に移行。test では
`Decimal('1.00') == Decimal('1.0')` で比較する。
```

### Field reference

| Field | Required | Type | Notes |
|---|---|---|---|
| `trigger` | yes | string | Comma-separated phrases / keywords. Used by retrieval to decide whether this item is relevant. Make it specific enough to match real queries. |
| `kind` | yes | enum | `convention` \| `decision` \| `runbook` \| `fact` \| `lesson`. Determines default category folder. |
| `scope` | yes | enum | `repo` (default — only relevant in this repo) or `global` (general lesson, repo-agnostic). |
| `tags` | no | list[string] | Free-form. Used for grouping in `lorgen wiki`-style summaries. |
| `sources` | yes (≥1) | list[object] | Each source has `type` ∈ `pr` / `issue` / `commit` / `adr` / `code` / `user-record` / `notion`, plus `ref`/`path`/`url`/`line` as appropriate. |
| `created` | yes | ISO date | First write date. |
| `updated` | yes | ISO date | Last write date. Update whenever the item is edited. |

### `kind` → default category mapping

| `kind` | Default folder |
|---|---|
| `convention` | `knowledge/conventions/` |
| `decision` | `knowledge/decisions/` |
| `runbook` | `knowledge/runbooks/` |
| `fact` | `knowledge/facts/` |
| `lesson` | `knowledge/lessons/` |

Pick `kind` based on what the item *is*:
- **convention** — a rule the codebase follows ("monetary 値は Decimal")
- **decision** — a discrete choice with rationale ("Postgres を Aurora に移行した理由")
- **runbook** — operational know-how ("デプロイ手順", "incident response")
- **fact** — non-obvious context ("entrypoint は `src/server.py`")
- **lesson** — generalizable insight, often from incidents

### Slug rules

- Lowercase ASCII, hyphen-separated, derived from a short title.
- **MUST match regex**: `^[a-z0-9][a-z0-9-]{0,63}$`
  - Starts with a-z or 0-9 (no leading hyphen)
  - 1-64 chars total
  - Only `a-z`, `0-9`, `-`
- **MUST NOT** contain `/`, `..`, NUL, whitespace, or any non-ASCII.
- File name: `<slug>.md`, e.g. `decimal-money.md`, `aurora-migration.md`.
- If a slug collides with an existing file, suffix `-2`, `-3`, ... — but
  prefer to update the existing item (see `accumulation.md` → dedup).

This regex is **enforced at write time** by `accumulation.md` Stage 4 —
LLM-generated slugs from possibly-malicious source content (PR titles,
issue bodies) are validated before any `Write` call. See `accumulation.md`
→ "Slug validation" for the exact bash check.

## Wiki page — front-matter schema

`.lorgen/wiki/<area>.md` (or `wiki/<parent>/<child>.md` for nested pages)

```markdown
---
title: Auth module
parent: architecture
summary: Session issuance, storage, refresh, and revocation.
updated: 2026-04-29
---

## Overview

[Body — Markdown. Diagrams (Mermaid), code references, links to Knowledge items.]
```

| Field | Required | Type | Notes |
|---|---|---|---|
| `title` | yes | string | Human-readable page title. |
| `parent` | no | string | Slug of the parent page. Omit for top-level. |
| `summary` | no | string | One-line description, used by `lorgen wiki list`. |
| `updated` | yes | ISO date | Last regeneration / hand-edit date. |

### Page links

In Wiki body, link to Knowledge items using their relative paths from
`.lorgen/`:

```markdown
See [convention: Decimal for money](../knowledge/conventions/decimal-money.md)
for the rationale.
```

This makes `git grep` and editor jump-to-file work.

## `.lorgen/config.yaml` — schema

```yaml
# Tracked in git. Team-shared settings.

model: claude-opus-4-7
effort: high           # low | medium | high | xhigh | max
max_tokens: 16000

sources:
  git: true
  code_comments: true
  github:
    enabled: true
    # repo: owner/name        # auto-detected from `git remote` when unset
    pr_lookback_days: 30
    pr_lookback_count: 50
  adr_dirs:
    - .lorgen/adr
    - docs/adr
    - docs/decisions
  # notion:                    # roadmap, not yet wired
  #   pages: []
  #   databases: []

outputs:
  write_adr: false             # auto-mirror every decision Knowledge as an ADR
  adr_dir: .lorgen/adr          # destination when ADRs are written

wiki:
  # Steering for `lorgen onboard` / wiki refresh.
  repo_notes: []
  pages: []
```

| Key | Type | Default | Purpose |
|---|---|---|---|
| `model` | string | `claude-opus-4-7` | Model used by the calling agent for synthesis. Surfaces here so the team can pin a model per repo. |
| `effort` | enum | `high` | `low`/`medium`/`high`/`xhigh`/`max`. Single-shot work — `xhigh` is overkill for most. |
| `max_tokens` | int | `16000` | Per-call cap. |
| `sources.git` | bool | `true` | Enable git log/blame collectors. |
| `sources.code_comments` | bool | `true` | Enable TODO/FIXME/HACK/why: scanner. |
| `sources.github.enabled` | bool | `true` | Enable `gh` CLI for PR/issue. Disable on repos without GitHub remote. |
| `sources.github.repo` | string | (auto) | `owner/name`. Auto-detected from `git remote get-url origin` when unset. |
| `sources.github.pr_lookback_days` | int | `30` | Default `lorgen onboard` window. |
| `sources.github.pr_lookback_count` | int | `50` | Cap on PR count per onboard run. |
| `sources.adr_dirs` | list | `[.lorgen/adr, docs/adr, docs/decisions]` | Directories to scan for ADR Markdown. |
| `outputs.write_adr` | bool | `false` | When `true`, every `decision`-kind Knowledge Lorgen writes is auto-mirrored as an ADR file at `outputs.adr_dir`. When `false`, ADRs are written only on per-record opt-in. |
| `outputs.adr_dir` | string | `.lorgen/adr` | Where ADR files go when written. Override to a project-wide convention (e.g. `docs/adr`, `docs/decisions`) if you prefer ADRs to live alongside the rest of `docs/`. |
| `wiki.repo_notes` | list[string] | `[]` | Free-form notes that steer `lorgen onboard` wiki generation (à la `.devin/wiki.json`). |
| `wiki.pages` | list[object] | `[]` | Explicit `[{title, purpose, parent?}]` list of wiki pages to generate. When set, only these pages are produced. |

## ADR files (optional)

Lorgen can also write industry-standard ADR files at
`.lorgen/adr/NNNN-<slug>.md` (default; the canonical Lorgen-curated
location) or wherever `outputs.adr_dir` points. **Off by default**;
enabled per-record (`@lorgen record --adr "..."`) or project-wide
(`outputs.write_adr: true` in `config.yaml`).

ADR files default to inside `.lorgen/` because they are Lorgen-curated
artefacts; the team can override `outputs.adr_dir` to e.g. `docs/adr/`
when they want ADRs surfaced alongside the rest of `docs/` (read by
humans, code review tools, other ADR tooling). Lorgen's role for ADRs:

- **Generate** new ADRs in MADR-lite format (Title / Status / Context /
  Decision / Consequences). Numbering auto-derived from existing files.
- **Cross-link** with the corresponding `.lorgen/knowledge/decisions/<slug>.md`
  item so retrieval can find either side.
- **Treat as immutable** — only edit existing ADRs to mark them
  `superseded by NNNN`; never rewrite the body.

Full generation rules: see `accumulation.md` → "Stage 5 (optional) — ADR
mirror".

## Audit log — `.lorgen/logs/<YYYYMMDD>-<HHMMSS>-<invocation_id>.jsonl`

Append-only JSONL, one event per line, one file per Lorgen invocation.
Authoritative event-type schema lives in `logging.md` — read that file
before writing any log line.

**Raw logs are gitignored** (per the `.gitignore` snippet above); they
contain query / candidate_summary / source content and stay per-machine.
At the end of every invocation Lorgen runs the metrics compile step
(see `metrics.md`) which reads the log and emits a sanitised
`.lorgen/metrics/<session>.json` — that compiled file is what's tracked
in git and feeds the improvement loop (stale detection, hot-item
promotion, trigger health).

## Compiled metrics — `.lorgen/metrics/<YYYYMMDD>-<HHMMSS>-<invocation_id>.json`

Tracked. Per-session aggregate of the matching log file. Schema and
compile contract (deterministic, idempotent, orphan-log handling)
live in `metrics.md`.

## `.lorgen/state.json` — internal state

```json
{
  "schema_version": 1,
  "onboard": {
    "last_completed_until": "2026-04-29T00:00:00Z",
    "last_chunk_completed": "2026-04",
    "in_flight": false
  },
  "indexing": {
    "last_scanned_commit": "abc1234"
  }
}
```

Used by `onboard.md`'s resume logic and to skip work that's already been
done. Not committed.
