# Logging — `.lorgen/logs/<session-file>.jsonl` schema

Lorgen writes a structured audit log for every invocation. This file is the
**authoritative schema** — anything Lorgen writes to `.lorgen/logs/` MUST follow
the shape defined here.

The log is consumed by:
- **the user**, when they want to answer "why did Lorgen do X?" (e.g.
  `cat .lorgen/logs/<latest-session>.jsonl | jq`)
- **the metrics compile step** (`metrics.md`) — every invocation ends
  by aggregating its own log into `.lorgen/metrics/<session>.json`. The
  compiled metrics are tracked in git; the raw log is not.
- **future Lorgen features** — `lorgen consolidate`, hot-item promotion,
  stale detection — all read **metrics**, not raw logs. Roadmap, not
  MVP, but the log → metrics shape exists now so signal accumulates
  from day one.

## File layout

```
.lorgen/logs/<YYYYMMDD>-<HHMMSS>-<invocation_id>.jsonl
                        # one file PER LORGEN INVOCATION, append-only JSONL
                        # example: 20260429-102345-a3f9c2.jsonl
```

- **Per-session, not per-day.** Two contributors running Lorgen on the
  same day produce two distinct files — no merge conflicts, no
  cross-process append races.
- **Gitignored by default.** `.lorgen/.gitignore` lists `logs/`. Raw
  events stay per-machine; team-wide signals are derived via the
  metrics compile step (`metrics.md`) and only the *compiled* metrics
  are committed.
- Why not track raw logs even with merge-safe naming? Raw events
  include `query`, `candidate_summary`, and source content that may
  carry PII or secrets. Metrics carry only paths, counts, and
  timestamps — privacy by aggregation.
- One event per line, one JSON object per line.
- UTF-8, ASCII-friendly (no NUL, no embedded newlines inside the JSON).

### Filename format

`<YYYYMMDD>-<HHMMSS>-<invocation_id>.jsonl`, all UTC.

- `<YYYYMMDD>-<HHMMSS>` — the moment the invocation started. Sortable
  with plain `ls`. UTC always.
- `<invocation_id>` — short random hex (6–8 chars), the same value
  used inside every event of this invocation.
- `.jsonl` — Newline-delimited JSON.

The whole filename is determined **at the start of the invocation** and
reused for every append within it.

## Common envelope (every event)

Every event line includes these fields:

```json
{
  "ts": "2026-04-29T10:23:45.123Z",
  "invocation_id": "a3f9c2",
  "event": "<event-type>",
  "...": "event-specific fields"
}
```

| Field | Type | Notes |
|---|---|---|
| `ts` | string | ISO 8601 UTC, second-or-better precision (`Z` suffix). |
| `invocation_id` | string | Short random id (e.g. 6-8 hex chars). Same value for every event in one Lorgen invocation, lets you slice "what happened in one `@lorgen ask`". |
| `event` | string | One of the event types below. |

## Event types

### Retrieval

#### `knowledge.referenced`

Emit once per Knowledge file Lorgen **read** while answering a question
(per `retrieval.md` Step 2 — full-read of a candidate).

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "knowledge.referenced",
  "path": ".lorgen/knowledge/conventions/decimal-money.md",
  "query": "なぜ Decimal を使ってる?",
  "helpful": true
}
```

- `helpful` — set to `true` if the file's content materially contributed
  to the final answer; `false` if you read it but it turned out
  off-topic. Best-effort judgment by Lorgen at the end of the answer
  synthesis. This field powers future stale-detection (high-read /
  low-helpful = trigger problem). **Required.** A missing `helpful`
  field is treated as `false` by `lorgen-compile-metrics` (so the
  helpful-ratio degrades visibly rather than silently understating
  the trigger problem) — but you should always emit the explicit
  value. Omitting it is a logging bug, not a valid "unknown".

#### `wiki.referenced`

Same shape as `knowledge.referenced`, but for `.lorgen/wiki/**/*.md`.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "wiki.referenced",
  "path": ".lorgen/wiki/modules/auth.md",
  "query": "auth モジュールについて",
  "helpful": true
}
```

### Source gathering

#### `sources.gathered`

Emit once per **batch** of source fetches (not per individual source).
Aggregate by source type so the log stays compact.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "sources.gathered",
  "query": "なぜ Decimal を使ってる?",
  "sources": [
    {"type": "git_log", "count": 12},
    {"type": "pr",       "refs": ["#42", "#43"]},
    {"type": "issue",    "refs": ["#37"]},
    {"type": "adr",      "paths": [".lorgen/adr/0003-decimal.md"]},
    {"type": "code",     "paths": ["src/billing/calc.py:42"]}
  ]
}
```

### Accumulation

#### `gate.decision`

Stage 2 ("worth saving?") result for each candidate.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "gate.decision",
  "candidate_summary": "PR #42: Decimal migration after 1¥ rounding incident",
  "decision": "save",
  "reason": "non-obvious why, references incident, codebase-specific"
}
```

`decision` ∈ `save` | `drop`. Always include `reason` (one sentence) so
future analysis can audit the gate's accuracy.

#### `dedup.decision`

Stage 3 (update vs create) result.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "dedup.decision",
  "action": "update",
  "existing_path": ".lorgen/knowledge/conventions/decimal-money.md",
  "new_path": null,
  "reason": "same topic, same conclusion; appending PR #99 as new source"
}
```

`action` ∈ `update` | `create` | `conflict` | `none`.
- `update` — `existing_path` is non-null, `new_path` is null
- `create` — `existing_path` is null, `new_path` is non-null
- `conflict` — both non-null (existing covers same topic but conflicts);
  reason explains, user is asked in stdout
- `none` — candidate dropped at this stage (rare, usually filtered earlier)

### Writes

#### `knowledge.created`

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "knowledge.created",
  "path": ".lorgen/knowledge/decisions/aurora-migration.md",
  "kind": "decision",
  "trigger": "Aurora, Postgres, database migration",
  "sources": [
    {"type": "pr",     "ref": "#42"},
    {"type": "commit", "ref": "abc1234"}
  ]
}
```

#### `knowledge.updated`

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "knowledge.updated",
  "path": ".lorgen/knowledge/conventions/decimal-money.md",
  "added_sources": [{"type": "pr", "ref": "#99"}],
  "reason": "new source extends incident detail"
}
```

#### `knowledge.deleted`

Only if Lorgen ever deletes (currently never, but logged for symmetry).

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "knowledge.deleted",
  "path": ".lorgen/knowledge/conventions/old-thing.md",
  "reason": "superseded by ..."
}
```

#### `wiki.created` / `wiki.updated`

Mirror of the knowledge events.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "wiki.updated",
  "path": ".lorgen/wiki/modules/auth.md",
  "reason": "added link to decision: aurora-migration"
}
```

#### `adr.created`

When ADR mirror writes a new ADR (per `accumulation.md` Stage 5).

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "adr.created",
  "path": ".lorgen/adr/0001-decimal-money.md",
  "number": 1,
  "knowledge_path": ".lorgen/knowledge/conventions/decimal-money.md"
}
```

#### `adr.superseded`

The single allowed mutation on existing ADRs (status change).

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "adr.superseded",
  "path": ".lorgen/adr/0003-old-decision.md",
  "superseded_by": ".lorgen/adr/0007-new-decision.md"
}
```

### Onboard

#### `onboard.chunk_completed`

One per chunk, mirrors the stdout progress line.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "onboard.chunk_completed",
  "chunk_id": "2026-04",
  "range": ["2026-04-01", "2026-04-30"],
  "prs_scanned": 12,
  "knowledge_added": 4,
  "knowledge_updated": 1,
  "knowledge_skipped": 7
}
```

### Review (Knowledge-grounded multi-agent code review)

> **SECURITY INVARIANT** (mirror of `bin/lorgen-compile-metrics`):
> the raw event JSON below contains free-text fields (`rule_id`,
> `rule_source`, `location`, `candidate`, `severity`) that originate
> from sub-agent JSON output and may carry attacker-chosen content.
> These fields stay in the **gitignored** `.lorgen/logs/<session>.jsonl`
> only. `bin/lorgen-compile-metrics` MUST NEVER emit any of them
> into the **tracked** `.lorgen/metrics/<session>.json` — counts only.
> Any future "include top rule_source by frequency" feature MUST
> add path-key validation matching the existing `KNOWLEDGE_RE` /
> `WIKI_RE` enforcement.

Review events are emitted by the Coordinator (the **main Claude Code
agent** running `skills/review/SKILL.md` via `/lorgen:review`, not
the Lorgen subagent). Sub-agents (Searcher, Investigators, Comparator)
run in isolated Task contexts and do NOT write to the log directly —
the Coordinator emits events on their behalf based on returned JSON.

Full skill: `skills/review/SKILL.md` (top-level plugin skill, not
inside `skills/lorgen/`). Metrics aggregation: `skills/lorgen/metrics.md`
→ `review:` section.

#### `review.started`

Emitted once at the very start of a review invocation, after CLI flag
parsing and target diff collection.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "review.started",
  "mode": "staged",
  "files_in_diff": 12,
  "hunks_in_diff": 234,
  "depth": "standard",
  "write_enabled": true
}
```

`mode` ∈ `staged` | `working` | `range` | `pr` | `files`.
`depth` ∈ `quick` | `standard` | `deep`.

#### `review.candidates_selected`

After Knowledge Searcher returns. Lists candidate Knowledge / Wiki /
ADR file paths and their relevance scores. Do NOT include the
Searcher's free-text reasons — keep the log compact.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "review.candidates_selected",
  "candidates": [
    {"path": ".lorgen/knowledge/conventions/decimal-money.md", "relevance": "HIGH"},
    {"path": ".lorgen/adr/0003-decimal.md", "relevance": "HIGH"},
    {"path": ".lorgen/wiki/architecture.md", "relevance": "MED"}
  ]
}
```

#### `review.investigation_completed`

One per Investigator. Logged after each parallel Task returns. Carries
counts only — the rules array stays in the in-memory pipeline.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "review.investigation_completed",
  "candidate": ".lorgen/knowledge/conventions/decimal-money.md",
  "rules_extracted": 3,
  "outdated_signals": 0
}
```

#### `review.comparator_finding`

One per finding the Comparator emits. Severity drives metrics
aggregation.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "review.comparator_finding",
  "severity": "critical",
  "location": "src/billing/calc.py:42",
  "rule_id": "decimal-money/no-float",
  "rule_source": ".lorgen/knowledge/conventions/decimal-money.md"
}
```

`severity` ∈ `critical` | `warning` | `info`.

#### `review.completed`

Final closure event for one review run. Mirrors `review.started` plus
aggregates so that a session's review summary can be reconstructed
from this single line.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "review.completed",
  "mode": "staged",
  "candidates_consulted": 8,
  "findings_total": 5,
  "findings_critical": 1,
  "findings_warning": 2,
  "findings_info": 2,
  "outdated_signals": 1,
  "lessons_added": 1,
  "sources_added": 2
}
```

`lessons_added` and `sources_added` are 0 when `--no-write` was used.

### LLM cost

#### `llm.call`

Emit per Anthropic API call your harness makes on Lorgen's behalf — when
that information is available. Always include purpose so cost can be
attributed.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "llm.call",
  "purpose": "answer.synthesis",
  "model": "claude-opus-4-7",
  "tokens": {"input": 3200, "output": 580, "cache_read": 1200, "cache_write": 0}
}
```

`purpose` ∈ `gate.decision` | `dedup.decision` | `answer.synthesis` |
`onboard.chunk` | `wiki.refresh` | `other`.

If exact tokens aren't available (e.g. Lorgen is running inside Claude
Code's session and per-call counts aren't surfaced), omit the `tokens`
field rather than guessing.

### Explicit user actions

#### `record.user_explicit`

Emitted when the user invoked `@lorgen record "..."` — the accumulation
gate is bypassed, the user is the source of truth.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "record.user_explicit",
  "kind": "decision",
  "path": ".lorgen/knowledge/decisions/release-process.md",
  "with_adr": false
}
```

### Errors / warnings

#### `error`

Anything Lorgen wanted to do but couldn't. Don't crash the invocation —
log and degrade gracefully.

```json
{
  "ts": "...",
  "invocation_id": "a3f9c2",
  "event": "error",
  "what": "gh.fetch",
  "detail": "gh CLI not authenticated; skipping GitHub source",
  "fatal": false
}
```

## How Lorgen writes the log

At the **start** of every invocation, decide the log file once and
reuse it for the whole invocation. In Bash:

```bash
mkdir -p .lorgen/logs

# Prefer Claude Code's session ID if exposed (forward-compat — currently
# NOT set in subagent Bash environments as of Claude Code 2.1.121).
# Fall back to a random hex tag.
INVOC="${LORGEN_INVOCATION_ID:-${CLAUDE_SESSION_ID:-$(openssl rand -hex 4)}}"
START_STAMP="${LORGEN_INVOCATION_START:-$(date -u +%Y%m%d-%H%M%S)}"  # UTC, generate ONCE
LOG_FILE=".lorgen/logs/${START_STAMP}-${INVOC}.jsonl"

# Then for every event during the invocation:
TS="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
echo "{\"ts\":\"$TS\",\"invocation_id\":\"$INVOC\",\"event\":\"knowledge.referenced\",\"path\":\".lorgen/knowledge/conventions/decimal-money.md\",\"query\":\"...\",\"helpful\":true}" >> "$LOG_FILE"
```

**Why the fallback chain on `INVOC`:**

- `LORGEN_INVOCATION_ID` — explicit override (test harness, manual
  reruns, etc.). First wins.
- `CLAUDE_SESSION_ID` — Claude Code's session UUID, if Anthropic ever
  surfaces it to subagent shells. **As of `claude-code 2.1.121` this
  var is not set** in subagent Bash environments (only hooks receive
  `session_id` via stdin). Listed here for forward compatibility —
  when it lands, our log filenames auto-line up with Claude Code's
  own session log files, so cross-referencing becomes trivial. Until
  then this falls through.
- `openssl rand -hex 4` — final fallback. 8 random hex chars, enough
  to avoid collisions in practice.

Tactics:

- **Generate `invocation_id` and `START_STAMP` ONCE** at the top of
  the invocation. Reuse both for the filename and for every event line.
- **Write events as you go**, not at the end. If Lorgen crashes
  mid-invocation, partial logs are still useful — and the per-session
  filename means a crashed invocation's log stands on its own, not
  interleaved with others.
- **Single line per event**. No pretty-printed JSON. Use `jq -c` if
  you're constructing JSON via `jq` (e.g.
  `jq -nc --arg path "$P" '{ts:..., event:"knowledge.referenced", path:$path}'`).
- **Don't log file CONTENTS** — only paths, sizes, source refs. The
  bodies live in the files themselves.
- **Don't log secrets**. Same redaction rules as `accumulation.md`'s
  "Saving secrets" anti-pattern: if a source contained an API key,
  redact before logging.

## What Lorgen does NOT log (yet)

- **`lorgen consolidate` runs** — Roadmap, not MVP.
- **Reference counts / aggregates** — derived state, computed from
  the JSONL stream when needed. The MVP keeps the log raw and forward-
  only; aggregation is a later layer.
- **User question text in full** if it contains PII — truncate to a
  query summary if needed.

## Reading the log (quick recipes)

```bash
# All events from today's invocations
jq -c . .lorgen/logs/$(date -u +%Y%m%d)-*.jsonl

# Most-referenced Knowledge files this week
jq -r 'select(.event=="knowledge.referenced") | .path' \
  .lorgen/logs/2026042{3,4,5,6,7,8,9}-*.jsonl \
  | sort | uniq -c | sort -rn | head -20

# What did Lorgen do in invocation a3f9c2?
ls .lorgen/logs/*-a3f9c2.jsonl   # find the file directly by ID
jq -c . .lorgen/logs/*-a3f9c2.jsonl

# All gate decisions where Lorgen dropped a candidate
jq -c 'select(.event=="gate.decision" and .decision=="drop")' .lorgen/logs/*.jsonl

# Latest invocation
ls -t .lorgen/logs/*.jsonl | head -1

# Stale candidates (Knowledge not referenced in 90 days)
# (Roadmap — `lorgen consolidate` will do this)
```

## Schema versioning

If we add an event type or field later, **never break existing event
shapes**. Add new optional fields, never repurpose old ones. New event
types are fine to add at any time. The reader (jq queries, Roadmap
`lorgen consolidate`) skips unknown event types gracefully.
