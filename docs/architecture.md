# Mimir architecture

> Long-form architecture overview for Mimir. Single living document, not
> an ADR — discrete decision records (one decision per file, immutable)
> live in `.mimir/knowledge/decisions/`.

## What Mimir is

Mimir is a **Claude Code plugin** containing:

- A subagent (`agents/mimir.md`) — the entry point and role definition.
- A skill bundle (`skills/mimir/`) — the operating manual, split into
  `SKILL.md` and topic sub-files (`schema`, `retrieval`, `accumulation`,
  `sources`, `onboard`, `logging`, `metrics`).
- A plugin manifest (`.claude-plugin/plugin.json`) — name, version,
  description for Claude Code's plugin manager.
- Hooks (`hooks/hooks.json`) and helper scripts (`bin/mimir-*`) — the
  mechanical secret-redaction guard and the deterministic metrics
  compile.

The data Mimir curates lives in `.mimir/` inside each user repo:

- `.mimir/knowledge/**/*.md` — small, trigger-retrievable Knowledge items (tracked)
- `.mimir/wiki/**/*.md` — coarser per-area pages (tracked)
- `.mimir/adr/<NNNN>-<slug>.md` — ADR mirrors of `decision`-kind
  Knowledge, written only when `outputs.write_adr: true` or per-record
  opt-in (tracked). Industry-standard MADR-lite format. Default
  location; override via `outputs.adr_dir` if your repo follows a
  different convention (e.g. `docs/adr/`, `docs/decisions/`).
- `.mimir/config.yaml`, `.mimir/.gitignore` (tracked)
- `.mimir/metrics/<YYYYMMDD>-<HHMMSS>-<id>.json` — per-session compiled
  aggregate (paths, counts, timestamps only) (tracked)
- `.mimir/logs/<YYYYMMDD>-<HHMMSS>-<id>.jsonl` — raw per-session audit
  trail (gitignored — contains query / source content with possible
  secrets; the compile step distills it into the tracked metrics)
- `.mimir/{cache,state.json}/` — gitignored runtime data

The two halves are physically separated:

- **Plugin source** lives in this repo (the marketplace), gets installed
  via `/plugin install mimir@...` to `.claude/plugins/mimir/` (project)
  or `~/.claude/plugins/mimir/` (global).
- **Data store** lives in each user repo's `.mimir/` directory, created
  on first use and curated by Mimir. Tracked in git.

## Why a Claude Code plugin (not standalone or Python CLI)

We considered three forms:

| Form | LLM cost | Setup | Distribution | Update | Verdict |
|---|---|---|---|---|---|
| **Python CLI** (Anthropic SDK directly) | separate API key, double-charge | `pip install`, venv, env vars | manual / pypi | manual | **rejected** — too heavy |
| **Standalone subagent + skill** (`.claude/agents/`, `.claude/skills/`, install.sh) | free (in caller's session) | `cp -r` + `install.sh` symlinks | manual | manual re-cp | **rejected after iteration** — manual install does not scale to many repos |
| **Claude Code plugin** | free | `/plugin install` 1 command | marketplace / GitHub | auto-update | **adopted** |

The plugin form wins on every dimension that matters for "publish so
others can use it easily". The trade-off is hard dependency on Claude
Code 2.0.70+; non-Claude-Code agents (Cursor, aider, CI/cron) cannot
consume the plugin form. We accept this — a future Roadmap CLI can
serve those use cases against the same `.mimir/` data model.

## Why split into `agents/` + `skills/`

A single subagent file would have to hold ~500+ lines of operating
instructions, which is hard to maintain and hard for the agent to load
selectively.

- `agents/mimir.md` — short, role definition, entry point. Tells Mimir
  its identity and where to find the manual.
- `skills/mimir/SKILL.md` — operating manual entry. Loaded when the
  task requires it.
- `skills/mimir/{schema,retrieval,accumulation,sources,onboard,logging}.md`
  — sub-files loaded only when relevant.

Same idea as `claude-api`, `update-config`, etc. — Anthropic's own
recommended pattern for non-trivial skills.

## Why `.mimir/` lives inside the user repo

Two alternatives were considered:

- **Per-user global store** (`~/.mimir/<repo-hash>/`) — works for solo
  dev but doesn't survive collaboration. Knowledge accumulated by one
  contributor is lost to the rest of the team.
- **Separate "knowledge repo"** — clean separation but requires a
  second repository, sync mechanism, and breaks the "the repo is the
  source of truth" principle.

In-repo wins on team sharing, version-control alignment, and review
workflow (PR diffs of `.mimir/` changes are visible alongside code
changes). The `.gitignore` carve-outs (`cache/`, `logs/`, `state.json`)
keep individual-machine noise out of the shared store.

## Why no commits from Mimir

Mimir writes files but does not run `git add` / `git commit` /
`gh pr create`. The human commits via normal git, after reviewing
`git status` / `git diff .mimir/`. Reasoning:

- Treats `.mimir/` writes the same as any other code change — same
  review loop, same merge culture.
- Avoids the failure mode where Mimir auto-commits hallucinated history
  and pollutes the audit log.
- Keeps Mimir's tool surface minimal (no `gh` CLI dependency for the
  default install).

All Mimir writes stay under `.mimir/` by default, including ADR
mirrors at `.mimir/adr/<NNNN>-<slug>.md` (when ADR generation is opted
in per-record or via `outputs.write_adr: true`). Users can override
`outputs.adr_dir` to point at a repo-existing ADR convention like
`docs/adr/` if they prefer; the redaction guard then extends scope to
that path automatically.

## Why ripgrep + LLM selection, not vector search

Cognition's argument for grep over vectors in code-heavy workloads
applies here: deterministic, fails loudly, no preprocessing, no false
positives. `.mimir/knowledge/` is structured Markdown with explicit
`trigger:` keywords specifically designed for keyword search. Vector
search would add preprocessing complexity and a failure mode (silent
mis-retrieval) without a corresponding accuracy gain.

If `.mimir/knowledge/` ever grows past hundreds of items per repo,
hybrid keyword + semantic retrieval can be added without changing the
file format.

## Why "read = accumulate"

Onboarding (`@mimir onboard`) and ad-hoc question-answering (`@mimir
ask`) both read the same sources (git, gh, ADRs, comments). The
accumulation side-effect is built into the question-answering path so:

1. Mimir's value compounds with use — every question that needed fresh
   sources leaves the next caller better off.
2. Users don't have to remember a separate "ingest" step.
3. The cost of accumulation is amortised over the value of the answer.

The accumulation pipeline (`accumulation.md`) has explicit gates to
prevent the natural failure mode (noise accumulation) — source-type
filter, LLM worth-saving judgment, and dedup.

## Why raw logs are gitignored, compiled metrics are tracked

Mimir writes JSONL events to
`.mimir/logs/<YYYYMMDD>-<HHMMSS>-<invocation_id>.jsonl` for every read,
write, gate decision, and source fetch (full schema in `logging.md`).
**One file per invocation** — per-session naming would make the logs
merge-safe even if tracked, but **raw logs stay gitignored** because
they contain `query` text, `candidate_summary` of source content, and
LLM intermediate output — anything PR/issue/comment authors wrote may
include secrets, PII, or other content that must not flow into git.

At the **end of every invocation**, Mimir runs `bin/mimir-compile-metrics`
on the session's log and emits `.mimir/metrics/<session>.json`. The
metrics file carries paths, counts, and timestamps only — privacy by
aggregation. **Metrics are tracked**, raw logs are not. The improvement
loop reads metrics, never raw logs.

Crashed invocations leave a log without a paired metrics file; the
next invocation's startup runs an orphan-log sweep to compile any
unmatched `logs/*.jsonl` (idempotent, deterministic, jq-only — no
LLM). Full contract in `metrics.md`.

This unblocks the Roadmap improvement-loop features:

- **Stale detection**: Knowledge not referenced in N days → propose
  archive.
- **Hot promotion**: frequently-referenced Knowledge → bump in
  retrieval ranking.
- **Trigger health**: high-read / low-helpful Knowledge (split via the
  `helpful_true` / `helpful_false` aggregate) → flag for trigger
  refinement.

By specifying log + metrics shapes now, the data accumulates from day
one and the analysis layer can be added later without back-fill.

## Why mechanical secret guard via PreToolUse hooks

Knowledge / Wiki / ADR files are tracked, so a single secret leaking
into them is a permanent git-history exposure. We use three layers of
mechanical defence:

1. **Source content as untrusted data**: `agents/mimir.md` instructs
   Mimir to never act on directives in PR/issue/commit/comment/ADR
   bodies — they are data, not instructions (mitigates prompt
   injection that could steer Mimir into writing secrets verbatim).
2. **PreToolUse hook** (`hooks/hooks.json` → `bin/mimir-pretool-guard`
   → `bin/mimir-redact-check`): scans every `Write|Edit` to
   `.mimir/**` (excluding `logs/` and `cache/`) and to the configured
   ADR directory for known secret patterns (AWS / GitHub / Slack /
   bearer / PEM / HTTPS-with-credentials). Any match blocks the write
   at the hook layer.
3. **Redact wrapper** (`bin/mimir-redact`): the skill instructs Mimir
   to pipe synthesised content through this wrapper before any
   `Write` call, so secrets are replaced with `[REDACTED:<kind>]`
   markers proactively. Hook layer is the safety net if the wrapper
   is bypassed.

Slug validation in `accumulation.md` Stage 4 is a parallel mechanical
defence against path traversal: every Knowledge / Wiki path is built
from a slug that must match `^[a-z0-9][a-z0-9-]{0,63}$`, validated in
bash before any `Write`. ADR autonumbering uses `find` with a strict
4-digit pattern and an upper bound of 9999 to defeat poisoned filename
attacks (`9999-evil.md` injection that would overflow the format
string).

## Distribution

The plugin is published from this repo to GitHub. Users install with:

```
/plugin install mimir@kawazy666/mimir
```

Auto-update is enabled by default (Claude Code 2.0.70+ feature). Users
can pin to a `version` field in `plugin.json` (semver) or follow the
HEAD commit SHA.

A submodule-based "central Mimir" was considered and rejected — plugin
manager handles distribution and versioning natively, no need for git
submodules in user repos.
