# Mimir — repository knowledge curator

> Captures the **"why"** behind code — design decisions, history, trade-offs,
> lessons — that source code alone cannot tell. Stores everything as
> Markdown inside the repo so humans and AI agents both read and grow the
> same knowledge base.

Named after **Mimir** (ミミル), the Norse oracle whose preserved head was
consulted by Odin for wisdom — same role this plugin plays for your repo.

Mimir is distributed as a **Claude Code plugin**. Install once, use across
every repo where you want to capture and retrieve the "why".

---

## Architecture

### This repo (`mimir`) — plugin source + self-hosted data

```
mimir/
├── .claude-plugin/
│   └── plugin.json                          # plugin manifest (name, version, ...)
├── agents/
│   └── mimir.md                             # subagent definition
├── skills/
│   └── mimir/                               # operating manual (multi-file skill)
│       ├── SKILL.md
│       ├── schema.md
│       ├── retrieval.md
│       ├── accumulation.md
│       ├── sources.md
│       ├── onboard.md
│       ├── logging.md
│       └── metrics.md                       # compile contract for logs → metrics
├── hooks/
│   └── hooks.json                           # PreToolUse: Write|Edit redaction guard
├── bin/
│   ├── mimir-pretool-guard                  # invoked by hooks/hooks.json
│   ├── mimir-redact-check                   # secret-pattern scanner (exit 1 = blocked)
│   ├── mimir-redact                         # redact wrapper (stdout)
│   └── mimir-compile-metrics                # jq-only deterministic compile
├── .mimir/                                  # data store (this repo's own, dogfood)
│   ├── config.yaml                          (tracked)
│   ├── .gitignore                           (tracked) excludes cache/ + state.json + logs/
│   ├── knowledge/                           (tracked)
│   │   ├── conventions/
│   │   └── decisions/
│   ├── wiki/                                (tracked)
│   │   ├── overview.md
│   │   └── architecture.md
│   ├── metrics/                             (tracked) per-session aggregate JSON
│   │   └── 20260429-102345-a3f9c2.json
│   ├── logs/                                (gitignored) raw per-session JSONL
│   ├── cache/                               (gitignored)
│   └── state.json                           (gitignored)
├── docs/
│   ├── architecture.md                      # long-form arch overview (living doc)
│   └── adr/                                 # only if outputs.write_adr=true
│       └── 0001-<slug>.md
└── README.md
```

### Your app repo, after `/plugin install mimir`

```
your-app-repo/
├── .claude/
│   └── plugins/
│       └── mimir/                           # installed plugin (project scope)
│           ├── .claude-plugin/plugin.json
│           ├── agents/mimir.md
│           ├── skills/mimir/...
│           ├── hooks/hooks.json
│           └── bin/mimir-*                  # redact-check, compile-metrics, etc.
├── .mimir/                                  # Mimir's data store, created on first use
│   ├── config.yaml                          (tracked) team-shared settings
│   ├── .gitignore                           (tracked) excludes cache/ + state.json + logs/
│   ├── knowledge/                           (tracked) Knowledge items
│   │   ├── conventions/
│   │   ├── decisions/
│   │   ├── runbooks/
│   │   ├── facts/
│   │   └── lessons/
│   ├── wiki/                                (tracked) Wiki pages
│   ├── metrics/                             (tracked) per-session compiled aggregates
│   │   ├── 20260429-102345-a3f9c2.json     #   paths, counts, timestamps only
│   │   ├── 20260429-110512-c8f2d4.json     #   privacy by aggregation
│   │   └── 20260430-091205-b1e3a7.json
│   ├── logs/                                (gitignored) raw per-session JSONL
│   │   └── 20260429-102345-a3f9c2.jsonl    #   compiled into metrics/ at end of run
│   ├── cache/                               (gitignored) raw PR/issue/LLM caches
│   └── state.json                           (gitignored) onboard cursor, indexing state
├── docs/
│   └── adr/                                 # only when outputs.write_adr=true
│       ├── 0001-decimal-money.md            (tracked) ADRs Mimir generated
│       └── 0002-aurora-migration.md
└── (your application code)
```

Three halves:
- **Plugin source** (`agents/`, `skills/mimir/`, `hooks/`, `bin/`,
  `.claude-plugin/plugin.json`) lives only in this repo. `/plugin install`
  drops a copy into `.claude/plugins/mimir/` (project) or
  `~/.claude/plugins/mimir/` (user).
- **Data store** (`.mimir/`) is created on first use inside each user
  repo. Knowledge / Wiki / Metrics tracked, raw logs gitignored.
- **ADR mirrors** (`docs/adr/`) are only written when `outputs.write_adr`
  is true (or per-record opt-in). Off by default.

**Logs vs metrics**: raw per-session logs live under `.mimir/logs/`
(gitignored, per-machine — they contain query text, source bodies, LLM
summaries that may carry secrets). At the end of every invocation,
Mimir compiles the log into `.mimir/metrics/<session>.json` (tracked,
team-shared) — paths, counts, timestamps only, privacy by
aggregation. The improvement loop (stale detection, hot-item
promotion, trigger health) reads metrics, never raw logs. See
[`skills/mimir/metrics.md`](skills/mimir/metrics.md).

**Mechanical secret guard**: a PreToolUse hook
(`hooks/hooks.json` + `bin/mimir-pretool-guard`) scans every Write/Edit
to `.mimir/**` (excluding `.mimir/logs/**` and `.mimir/cache/**`) and
to the configured ADR directory for known secret patterns; matches
block the write at hook time. The skill instructions also pipe content
through `bin/mimir-redact` as defence-in-depth. See
[`bin/mimir-redact-check`](bin/mimir-redact-check) for the pattern set.

---

## What Mimir does

| You ask | Mimir does |
|---|---|
| `なぜ Decimal を money に使ってる?` | Searches `.mimir/knowledge/` first; if not found, reads `git log`, PR descriptions, ADRs, code comments; synthesises an answer with citations; persists the new finding to `.mimir/knowledge/conventions/decimal-money.md` so the next question lands instantly. |
| `auth モジュールの設計判断履歴を教えて` | Reads `.mimir/wiki/modules/auth.md` and linked Knowledge items; if gaps, follows references into git/PR/issue history and updates the Wiki. |
| `この PR の判断を残しておいて` | Distills the PR description and review comments into a new `decision`-kind Knowledge item with full source citations. |
| `この repo を onboard して` | Performs a thorough first-pass scan: README, CLAUDE.md, AGENTS.md, ADRs, recent merged PRs (default 30 days / 50 PRs), code comments → produces an initial Wiki overview, per-module pages, and seed Knowledge items. |
| `この決定を ADR にして` / `@mimir record --adr "..."` | Mirrors the Knowledge item to `docs/adr/NNNN-<slug>.md` in industry-standard MADR-lite format. Auto-numbered, immutable, cross-linked with the Knowledge file. Off by default; opt in per record or globally via `outputs.write_adr: true`. |
| `過去 1 年遡って onboard して` | Same, but in monthly chunks with state-file resume support. |

Output lands on stdout in Markdown with inline citations
(`(PR #42)`, `(commit abc1234)`, `(docs/adr/0003-decimal.md)`). Files
are written into `.mimir/`. Mimir **never** commits or pushes — the human
reviews `git diff .mimir/` and commits when ready.

---

## Why Mimir, not just Claude Code alone

| Without Mimir | With Mimir |
|---|---|
| Every new session re-reads README, re-greps for conventions | Knowledge accumulated once, retrieved instantly thereafter |
| "Why is this here?" answered from current code only | Answered from git history, PR rationale, ADRs, past lessons — distilled and persisted |
| Knowledge dies with the conversation | Knowledge lives in `.mimir/`, survives sessions, survives contributors, version-controlled, team-shared |
| Same question costs the same tokens every time | Repeat questions hit the cached `.mimir/knowledge/` and are nearly free |

Mimir is **inspired by Devin's Knowledge + DeepWiki** features but lives
locally in your repo, costs nothing extra (runs in Claude Code's session),
and is fully transparent — the entire data model is plain Markdown you
can read, edit, or rewrite.

---

## Install

```bash
# Project-local install (plugin lives in .claude/plugins/mimir/ inside your repo)
/plugin install mimir@kawazy666/mimir

# Or user-global install (plugin lives in ~/.claude/plugins/mimir/)
/plugin install mimir@kawazy666/mimir --user
```

To pin a version, append `@v0.1.0`; to follow latest, omit the version
(defaults to current commit SHA). If you fork or publish your own
copy, replace `kawazy666/mimir` with your GitHub `owner/repo`.

After install, restart Claude Code (or `/reload-plugins`) so the new
agent and skill are picked up. Then in any project:

```
@mimir 初期化して
```

Mimir creates `.mimir/config.yaml`, `.mimir/.gitignore`, and the empty
`knowledge/` / `wiki/` directories. Commit those when ready.

---

## Use

In Claude Code, invoke Mimir as the subagent:

```
@mimir なぜ Decimal を money に使ってるの?

@mimir この repo を onboard して

@mimir 過去 1 年の決定を遡って蓄積して、月単位で

@mimir record "release process: tags push を main の merge 後に実行する"

@mimir この決定を ADR にして
```

Or just describe what you want; Claude Code routes to Mimir when the
intent matches its `description`.

---

## Remove

```bash
/plugin remove mimir            # remove the plugin (skill + agent)
rm -rf .mimir                   # remove the data store (your call)
```

`/plugin remove` only deletes Mimir's binary; your `.mimir/knowledge/`
and `.mimir/wiki/` are preserved unless you also `rm -rf .mimir`.

---

## File schemas (short version)

### Knowledge item — `.mimir/knowledge/<category>/<slug>.md`

```markdown
---
trigger: "Decimal, money, monetary calculation"
kind: convention            # convention | decision | runbook | fact | lesson
scope: repo
tags: [money, billing]
sources:
  - {type: pr, ref: "#42", url: "..."}
  - {type: commit, ref: "abc1234"}
  - {type: adr, path: "docs/adr/0003-decimal.md"}
created: 2026-04-29
updated: 2026-04-29
---

[Markdown body — a few sentences to a couple paragraphs.]
```

### Wiki page — `.mimir/wiki/<area>.md`

```markdown
---
title: Auth module
parent: architecture
summary: Session issuance, storage, refresh, and revocation.
updated: 2026-04-29
---

[Markdown body — overview of the module, links to Knowledge items.]
```

Full reference: [`skills/mimir/schema.md`](skills/mimir/schema.md)

---

## Configuration

`.mimir/config.yaml` (tracked, team-shared):

```yaml
model: claude-opus-4-7
effort: high

sources:
  git: true
  code_comments: true
  github:
    enabled: true
    pr_lookback_days: 30
    pr_lookback_count: 50
  adr_dirs:
    - docs/adr
    - docs/decisions

outputs:
  write_adr: false
  adr_dir: docs/adr

wiki:
  repo_notes: []
  pages: []
```

Full reference: [`skills/mimir/schema.md`](skills/mimir/schema.md)

---

## What Mimir won't do

- **Never commit, push, or open PRs** — you do that, after reviewing `git diff .mimir/`.
- **Never modify source code outside `.mimir/`** (one allowed outbound write target: `docs/adr/<NNNN>-<slug>.md` when ADR generation is opted in).
- **Never invent history** — every claim cites a real source. If sources
  don't support a confident answer, Mimir says so.
- **Never store secrets** — API keys, tokens get redacted before persisting.

---

## Architecture deep-dive

See [`docs/architecture.md`](docs/architecture.md) for design decisions,
the alternatives considered, and why the current shape was chosen
(plugin distribution, single `.mimir/` data store, ripgrep + LLM retrieval
over vectors, "read = accumulate").

---

## Status

| Component | State |
|---|---|
| Subagent (`agents/mimir.md`) | **MVP complete** |
| Skill bundle (`skills/mimir/*`) | **MVP complete** — SKILL.md + schema/retrieval/accumulation/sources/onboard/logging/metrics |
| `.mimir/` data store schema | **MVP complete** |
| Plugin manifest (`.claude-plugin/plugin.json`) | **MVP complete** |
| `mimir onboard` (recent + 1y backfill) | **MVP** (described in skill, executed by the agent) |
| ADR generation (`outputs.write_adr` + per-record opt-in) | **MVP complete** |
| Audit log schema (`.mimir/logs/<session>.jsonl`, gitignored) | **MVP complete** |
| Compiled metrics (`.mimir/metrics/<session>.json`, tracked) + `bin/mimir-compile-metrics` + smoke test | **MVP complete** |
| Mechanical secret guard (PreToolUse hook + `bin/mimir-redact{,-check}`) | **MVP complete** |
| `mimir consolidate` (Knowledge dedup pass) | **Roadmap** |
| Stale-detection + hot-item promotion (consumes metrics) | **Roadmap** |
| Notion source connector | **Roadmap** |
| Periodic auto-reindex | **Roadmap** |
| Standalone CLI for non-Claude-Code callers | **Roadmap** |

---

## License

MIT.
