# Lorgen architecture

> Long-form architecture overview for Lorgen. Single living document, not
> an ADR — discrete decision records (one decision per file, immutable)
> live in `.lorgen/knowledge/decisions/`.

## What Lorgen is

Lorgen is a **Claude Code plugin** containing:

- A subagent (`agents/lorgen.md`) — the entry point and role definition.
- A skill bundle (`skills/lorgen/`) — the operating manual, split into
  `SKILL.md` and topic sub-files (`schema`, `retrieval`, `accumulation`,
  `sources`, `onboard`, `logging`, `metrics`).
- A plugin manifest (`.claude-plugin/plugin.json`) — name, version,
  description for Claude Code's plugin manager.
- Hooks (`hooks/hooks.json`) and helper scripts (`bin/lorgen-*`) — the
  mechanical secret-redaction guard and the deterministic metrics
  compile.

The data Lorgen curates lives in `.lorgen/` inside each user repo:

- `.lorgen/knowledge/**/*.md` — small, trigger-retrievable Knowledge items (tracked)
- `.lorgen/wiki/**/*.md` — coarser per-area pages (tracked)
- `.lorgen/adr/<NNNN>-<slug>.md` — ADR mirrors of `decision`-kind
  Knowledge, written only when `outputs.write_adr: true` or per-record
  opt-in (tracked). Industry-standard MADR-lite format. Default
  location; override via `outputs.adr_dir` if your repo follows a
  different convention (e.g. `docs/adr/`, `docs/decisions/`).
- `.lorgen/config.yaml`, `.lorgen/.gitignore` (tracked)
- `.lorgen/metrics/<YYYYMMDD>-<HHMMSS>-<id>.json` — per-session compiled
  aggregate (paths, counts, timestamps only) (tracked)
- `.lorgen/logs/<YYYYMMDD>-<HHMMSS>-<id>.jsonl` — raw per-session audit
  trail (gitignored — contains query / source content with possible
  secrets; the compile step distills it into the tracked metrics)
- `.lorgen/{cache,state.json}/` — gitignored runtime data

The two halves are physically separated:

- **Plugin source** lives in this repo (the marketplace), gets installed
  via `/plugin install lorgen@...` to `.claude/plugins/lorgen/` (project)
  or `~/.claude/plugins/lorgen/` (global).
- **Data store** lives in each user repo's `.lorgen/` directory, created
  on first use and curated by Lorgen. Tracked in git.

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
serve those use cases against the same `.lorgen/` data model.

## Why split into `agents/` + `skills/`

A single subagent file would have to hold ~500+ lines of operating
instructions, which is hard to maintain and hard for the agent to load
selectively.

- `agents/lorgen.md` — short, role definition, entry point. Tells Lorgen
  its identity and where to find the manual.
- `skills/lorgen/SKILL.md` — operating manual entry. Loaded when the
  task requires it.
- `skills/lorgen/{schema,retrieval,accumulation,sources,onboard,logging}.md`
  — sub-files loaded only when relevant.

Same idea as `claude-api`, `update-config`, etc. — Anthropic's own
recommended pattern for non-trivial skills.

## Why `.lorgen/` lives inside the user repo

Two alternatives were considered:

- **Per-user global store** (`~/.lorgen/<repo-hash>/`) — works for solo
  dev but doesn't survive collaboration. Knowledge accumulated by one
  contributor is lost to the rest of the team.
- **Separate "knowledge repo"** — clean separation but requires a
  second repository, sync mechanism, and breaks the "the repo is the
  source of truth" principle.

In-repo wins on team sharing, version-control alignment, and review
workflow (PR diffs of `.lorgen/` changes are visible alongside code
changes). The `.gitignore` carve-outs (`cache/`, `logs/`, `state.json`)
keep individual-machine noise out of the shared store.

## Why no commits from Lorgen

Lorgen writes files but does not run `git add` / `git commit` /
`gh pr create`. The human commits via normal git, after reviewing
`git status` / `git diff .lorgen/`. Reasoning:

- Treats `.lorgen/` writes the same as any other code change — same
  review loop, same merge culture.
- Avoids the failure mode where Lorgen auto-commits hallucinated history
  and pollutes the audit log.
- Keeps Lorgen's tool surface minimal (no `gh` CLI dependency for the
  default install).

All Lorgen writes stay under `.lorgen/` by default, including ADR
mirrors at `.lorgen/adr/<NNNN>-<slug>.md` (when ADR generation is opted
in per-record or via `outputs.write_adr: true`). Users can override
`outputs.adr_dir` to point at a repo-existing ADR convention like
`docs/adr/` if they prefer; the redaction guard then extends scope to
that path automatically.

## Why ripgrep + LLM selection, not vector search

Cognition's argument for grep over vectors in code-heavy workloads
applies here: deterministic, fails loudly, no preprocessing, no false
positives. `.lorgen/knowledge/` is structured Markdown with explicit
`trigger:` keywords specifically designed for keyword search. Vector
search would add preprocessing complexity and a failure mode (silent
mis-retrieval) without a corresponding accuracy gain.

If `.lorgen/knowledge/` ever grows past hundreds of items per repo,
hybrid keyword + semantic retrieval can be added without changing the
file format.

## Why "read = accumulate"

Onboarding (`@lorgen onboard`) and ad-hoc question-answering (`@lorgen
ask`) both read the same sources (git, gh, ADRs, comments). The
accumulation side-effect is built into the question-answering path so:

1. Lorgen's value compounds with use — every question that needed fresh
   sources leaves the next caller better off.
2. Users don't have to remember a separate "ingest" step.
3. The cost of accumulation is amortised over the value of the answer.

The accumulation pipeline (`accumulation.md`) has explicit gates to
prevent the natural failure mode (noise accumulation) — source-type
filter, LLM worth-saving judgment, and dedup.

## Why raw logs are gitignored, compiled metrics are tracked

Lorgen writes JSONL events to
`.lorgen/logs/<YYYYMMDD>-<HHMMSS>-<invocation_id>.jsonl` for every read,
write, gate decision, and source fetch (full schema in `logging.md`).
**One file per invocation** — per-session naming would make the logs
merge-safe even if tracked, but **raw logs stay gitignored** because
they contain `query` text, `candidate_summary` of source content, and
LLM intermediate output — anything PR/issue/comment authors wrote may
include secrets, PII, or other content that must not flow into git.

At the **end of every invocation**, Lorgen runs `bin/lorgen-compile-metrics`
on the session's log and emits `.lorgen/metrics/<session>.json`. The
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

1. **Source content as untrusted data**: `agents/lorgen.md` instructs
   Lorgen to never act on directives in PR/issue/commit/comment/ADR
   bodies — they are data, not instructions (mitigates prompt
   injection that could steer Lorgen into writing secrets verbatim).
2. **PreToolUse hook** (`hooks/hooks.json` → `bin/lorgen-pretool-guard`
   → `bin/lorgen-redact-check`): scans every `Write|Edit` to
   `.lorgen/**` (excluding `logs/` and `cache/`) and to the configured
   ADR directory for known secret patterns (AWS / GitHub / Slack /
   bearer / PEM / HTTPS-with-credentials). Any match blocks the write
   at the hook layer.
3. **Redact wrapper** (`bin/lorgen-redact`): the skill instructs Lorgen
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

## Why a 4-role multi-agent team for review

`/lorgen:review` checks a code change against the curated `.lorgen/`
knowledge. A naive single-prompt implementation — "here's the diff
and all of `.lorgen/`, find the conflicts" — fails for two reasons:

1. **Context bloat.** Even modest repos accumulate dozens of
   Knowledge / Wiki / ADR files. Stuffing everything into one prompt
   blows the context budget and produces shallow reasoning across
   too much material.
2. **Task interleaving.** The model has to simultaneously decide what
   is relevant, read it carefully, and judge code conformance — three
   different cognitive modes that interfere with each other.

The team splits the work into independent passes, each in its own
sub-context (spawned via the `Task` tool):

| Role | Spawn count | Job | Why isolated |
|---|---|---|---|
| **Coordinator** | 1 (the main Claude Code agent — NOT a Lorgen subagent) | Parses CLI flags, collects diff, orchestrates the team, renders the final report, hands new lessons to accumulation. | Must be the main agent because `Task`-spawning is a main-agent-only capability on Claude Code 2.x. Holds small persistent state without bloating with file bodies. |
| **Knowledge Searcher** | 1 | Greps `.lorgen/{knowledge,wiki,adr}` to produce a ranked candidate list (path + relevance hint). Reads only headers / triggers, never bodies. | Pure relevance triage. Output is a small JSON list. |
| **Knowledge Investigator** | N (parallel) | Deep-reads ONE candidate, extracts rules / rationales / source citations / outdated signals. | Each Investigator's context holds one Knowledge file. Parallel `Task` calls in a single message run concurrently. |
| **Code Comparator** | 1 (or 2 if rules > 50) | Receives all Investigators' rules + the diff, decides per-rule conformance / warning / violation, suggests fixes. | Holds the full diff once and walks the rules list — this is where careful matching happens. |

This mirrors Cognition's "use Devin Knowledge first, then code review"
pattern, but spelled out as discrete agents so the work is explicit
and auditable in the log (`review.candidates_selected`,
`review.investigation_completed`, `review.comparator_finding`,
`review.completed`).

Trade-offs we accepted:

- **Latency**: 4 sequential phases (Searcher → Investigators →
  Comparator → Coordinator render) is slower than a single LLM call.
  The parallel Investigator phase amortises this somewhat — N
  candidates take roughly the same wall time as one. We optimise for
  quality of finding, not minimum latency.
- **Token cost**: more agents = more prompt overhead. Mitigated by
  (a) the `--depth quick|standard|deep` knob capping candidate count
  and (b) Investigators receiving only a hunks-summary of the diff,
  not the full text — the full diff goes to the Comparator only.
- **Failure modes**: any sub-agent's failure leaves the Coordinator
  with partial data. We log per-role events and degrade gracefully
  (an empty Searcher result yields a "no relevant Knowledge
  intersected this diff" report rather than a crash).

The accumulation side-effect (new lessons / source additions to
existing Knowledge) reuses Lorgen's `accumulation.md` machinery — the
Comparator's `severity: critical` finding **is** the Stage 2
worth-saving gate, Stage 3 dedup runs against
`.lorgen/knowledge/lessons/`, and Stage 4 writes through the same
`lorgen-redact` pipe and `lorgen-pretool-guard` PreToolUse hook as any
other Lorgen write. Secret guard and slug validation are uniform with
the rest of Lorgen.

### Why `/lorgen:review` and not `@lorgen review`

A subagent on Claude Code 2.x cannot grant itself the `Task` tool —
the platform restriction is documented (`Subagents cannot spawn other
subagents`). The 4-role team requires `Task` to spawn workers, so the
Coordinator must be the main agent. The slash command
(`commands/review.md` → auto-namespaced `/lorgen:review`) runs in the
main agent's context, loads `skills/review/SKILL.md`, and orchestrates
the team. We considered offering a degraded single-context
`@lorgen review` fallback and rejected it: same name, two very
different behaviours is the wrong cognitive load. A user who types
`@lorgen review` is told to use `/lorgen:review` and gets the real
implementation.

## Distribution

The repo serves as **both** a Claude Code marketplace and the single
plugin it lists. The marketplace manifest at
`.claude-plugin/marketplace.json` declares one plugin entry with
`source: "./"`, pointing at the repo root where `plugin.json` lives.

Users install in two steps:

```
/plugin marketplace add kawazy666/lorgen
/plugin install lorgen@lorgen
```

Auto-update is enabled by default (Claude Code 2.0.70+ feature). Users
can pin to a `version` field in `plugin.json` (semver), tag the marketplace
add (`@v0.1.0`), or follow the HEAD commit SHA.

A submodule-based "central Lorgen" was considered and rejected — plugin
manager handles distribution and versioning natively, no need for git
submodules in user repos.
