# Metrics — `.lorgen/metrics/<session>.json` schema and compile contract

Lorgen compiles raw per-session logs into per-session aggregates. The
**raw logs are gitignored**; the **compiled metrics are tracked**. This
is privacy-by-aggregation: paths, counts, and timestamps survive into
git; query text, source bodies, and LLM summaries do not.

## File layout

```
.lorgen/metrics/<YYYYMMDD>-<HHMMSS>-<invocation_id>.json
                 # one file per Lorgen invocation, paired 1:1 with
                 # .lorgen/logs/<same-stem>.jsonl
```

Per-session naming → merge-safe across contributors (no two metrics
files ever collide).

## Schema (v1)

```json
{
  "schema_version": 1,
  "session": "20260429-102345-a3f9c2",
  "started_at": "2026-04-29T10:23:45Z",
  "ended_at": "2026-04-29T10:30:12Z",
  "compile_status": "complete",

  "knowledge_refs": {
    ".lorgen/knowledge/conventions/decimal-money.md": {
      "refs": 3,
      "helpful_true": 2,
      "helpful_false": 1,
      "first_ts": "2026-04-29T10:24:01Z",
      "last_ts":  "2026-04-29T10:28:55Z"
    }
  },

  "wiki_refs": {
    ".lorgen/wiki/architecture.md": {
      "refs": 1,
      "helpful_true": 1,
      "helpful_false": 0,
      "first_ts": "...",
      "last_ts":  "..."
    }
  },

  "retrieved_but_unused": 2,

  "gates": {
    "save": 2,
    "drop": 5
  },

  "dedup": {
    "update": 1,
    "create": 1,
    "conflict": 0,
    "none": 0
  },

  "sources": {
    "pr":      4,
    "issue":   1,
    "git_log": 12,
    "git_blame": 0,
    "adr":     2,
    "code":    6,
    "user_record": 0
  },

  "writes": {
    "knowledge_created": 1,
    "knowledge_updated": 1,
    "knowledge_deleted": 0,
    "wiki_created": 0,
    "wiki_updated":  1,
    "adr_created":   0,
    "adr_superseded": 0,
    "user_explicit":  0
  },

  "review": {
    "runs": 1,
    "findings_total": 5,
    "findings_by_severity": {
      "critical": 1,
      "warning":  2,
      "info":     2
    },
    "outdated_signals": 1,
    "lessons_added":    1,
    "sources_added":    2
  },

  "errors": 0
}
```

The `review` key is **omitted entirely** when no `review.*` event was
logged in the session — keeps non-review sessions' metrics small and
forward-compatible. Readers MUST treat a missing `review` key as
"no reviews ran in this session", not as zeros.

### Field reference

| Field | Type | Notes |
|---|---|---|
| `schema_version` | int | Currently `1`. Bump when fields change incompatibly. Readers MUST check and skip incompatible files rather than silently misinterpret. |
| `session` | string | Same value as the matching log's `invocation_id`. |
| `started_at` / `ended_at` | ISO 8601 UTC | Window the invocation actually ran. |
| `compile_status` | enum | `complete` (log was fully processed and the invocation finished cleanly) or `partial` (compiled from a log that crashed mid-invocation — see "Orphan logs" below). |
| `knowledge_refs` | map | Per-file aggregate. Key is the full path relative to repo root. Slug in the path **must match** `^[a-z0-9][a-z0-9-]{0,63}\.md$` — compile rejects files with malformed paths to prevent metrics-via-log poisoning. |
| `*.helpful_true` / `*.helpful_false` | int | **Split, not summed.** Trigger health depends on `helpful_false / refs` ratio; a single `helpful_count` collapses the signal. `helpful_false` includes both explicit-false events AND events with `helpful` missing — the compile coerces missing to false (per `logging.md`) so the ratio degrades visibly when callers forget to emit the field. Invariant: `helpful_true + helpful_false == refs`. |
| `wiki_refs` | map | Same shape as `knowledge_refs`, scoped to `.lorgen/wiki/**/*.md`. |
| `retrieved_but_unused` | int | Knowledge / Wiki files Lorgen read while answering, but whose content did not appear in the final answer. Distinct from `helpful: false`, which is the per-file flag. The total is convenient. |
| `gates` | map | Counts of `gate.decision` events by `decision`. |
| `dedup` | map | Counts of `dedup.decision` events by `action`. |
| `sources` | map | Counts of source items consulted, summed per type from `sources.gathered` events. |
| `writes` | map | Counts of write events. `user_explicit` separately counts `record.user_explicit` events so cross-session tally of user-driven records survives raw-log gitignore. |
| `review` | map (optional) | Aggregated from `review.*` events. **Omitted entirely** when the session ran no reviews. `runs` counts `review.completed` events. `findings_total` and `findings_by_severity.*` come from `review.comparator_finding` events. `outdated_signals` / `lessons_added` / `sources_added` come from `review.completed` event aggregates so cross-session trend analysis works without re-walking per-finding events. |
| `errors` | int | Count of `error` events with `fatal: false`. Fatal errors mean the invocation crashed — see Orphan logs. |

### Path keys: validation contract

Every key under `knowledge_refs` and `wiki_refs` is a path. Compile
**MUST** validate each key against:

```
^\.lorgen/(knowledge/(conventions|decisions|runbooks|facts|lessons)|wiki(/[a-z0-9][a-z0-9-]{0,63})*)/[a-z0-9][a-z0-9-]{0,63}\.md$
```

If a key fails validation, compile drops that key (does NOT include it
in metrics) and emits an `error` event into the **next** session's log.
This prevents poisoned slugs from ever reaching tracked files.

## Compile contract

### When

Lorgen runs the compile at the **end of every invocation**, in the
SKILL.md operating loop. The skill instructs:

> After answering and before returning, run the metrics compile step
> on this invocation's log.

If a compile fails (e.g. log file unreadable), Lorgen reports the
failure in stdout; the invocation still completes its primary work.

### Implementation: deterministic, jq-only

Compile **MUST** be a pure transformation: log events in → metrics out.
No LLM calls. No network.

> **Canonical implementation lives in `bin/lorgen-compile-metrics`.**
> The bash + jq snippet below is **illustrative only** — read the
> script for the authoritative behaviour (helpful-missing coercion,
> sorted timestamp pick, `writes.user_explicit`, path-key
> validation, atomic write). When the script and this sketch
> diverge, the script wins.

Sketch — what the script does end-to-end:

```bash
#!/usr/bin/env bash
# Authoritative source: bin/lorgen-compile-metrics.
# This sketch omits: validation regex anchoring, timestamp sorting
# for started_at/ended_at, helpful-missing coercion, user_explicit
# count, atomic temp-file rename. Refer to the script.
LOG="$1"
SESSION="$(basename "$LOG" .jsonl)"
jq -n --slurpfile evs "$LOG" --arg session "$SESSION" '
  ($evs // []) as $events
  | ($events | map(select(.event == "knowledge.referenced"))) as $kref
  | ($events | map(select(.event == "wiki.referenced")))      as $wref
  | {
      schema_version: 1,
      session: $session,
      knowledge_refs: ($kref | group_by(.path) | map({
          key: .[0].path,
          value: {
            refs: length,
            helpful_true:  map(select(.helpful == true)) | length,
            helpful_false: map(select(.helpful != true)) | length
          }
      }) | from_entries),
      # ... full shape and event handlers in bin/lorgen-compile-metrics
    }
'
```

This is intentionally portable bash + jq. No LLM. No shell expansion of
attacker-controlled values.

### Path-key validation step

The compile script applies the path-key regex during the main jq pass
(`map(select((.path // "") | test($krx)))` with `--arg krx`), so any
key that fails validation is dropped before reaching the output.
Equivalent post-filter form (for ad-hoc analysis of an existing
metrics file):

```bash
jq -c '
  .knowledge_refs |= with_entries(select(.key | test("^\\.lorgen/knowledge/(conventions|decisions|runbooks|facts|lessons)/[a-z0-9][a-z0-9-]{0,63}\\.md$")))
  | .wiki_refs    |= with_entries(select(.key | test("^\\.lorgen/wiki(/[a-z0-9][a-z0-9-]{0,63})*/[a-z0-9][a-z0-9-]{0,63}\\.md$")))
'
```

### Idempotency

Compile is a pure function of the input log. Re-running compile on the
same log file MUST produce byte-identical output (modulo `jq`'s key
ordering, which is stable in 1.6+). The skill instruction: "if
`metrics/<session>.json` already exists, overwrite it (idempotent
re-compile)."

### Orphan logs (crashed invocations)

A crashed invocation leaves `.lorgen/logs/<X>.jsonl` without a matching
`.lorgen/metrics/<X>.json`. The next Lorgen invocation runs an
**orphan-log sweep** at startup (after generating its own session id):

```bash
# `lorgen-compile-metrics` is on PATH because the plugin's bin/ is
# auto-prepended when Lorgen is active. Do NOT use bin/lorgen-... or
# ${CLAUDE_PLUGIN_ROOT}/bin/... here — neither resolves correctly
# from a subagent Bash call (cwd is the user repo).
mkdir -p .lorgen/metrics
for log in .lorgen/logs/*.jsonl; do
  metric=".lorgen/metrics/$(basename "$log" .jsonl).json"
  if [[ ! -f "$metric" ]]; then
    lorgen-compile-metrics "$log"  # writes $metric atomically; idempotent
  fi
done
```

The sweep:
- Compiles only logs that lack a matching metric (idempotent on
  successful sessions).
- Marks orphan-derived metrics with `compile_status: partial` (the
  compile script's status logic detects fatal errors / abrupt end).
- Never touches an already-compiled metric (an old crashed session
  re-compiled later carries its own session id; no cross-session key
  contamination).

### Schema versioning

Bumping `schema_version` is allowed but every change MUST:

1. Add or rename fields, never repurpose existing ones.
2. Update this file's "Schema" section with a clear `## v2` heading.
3. Make the compile script emit the new shape.
4. Document a `lorgen consolidate --re-aggregate` recipe to recompute
   any cross-session aggregates that depend on changed fields.

Readers (Roadmap `lorgen consolidate`, hot/stale detectors) MUST guard
on `schema_version` and skip files with versions they don't understand.

### Regression test surface

`bin/lorgen-compile-metrics-smoke` is the canonical regression test
for the compile contract. It builds JSONL fixtures in a tmpdir and
asserts: multi-event populates `knowledge_refs` / `wiki_refs` with
correct `helpful_true` / `helpful_false` split, idempotency
(byte-identical hash on re-run), empty log → valid empty schema,
path-traversal entries dropped, missing-`helpful` coerced to false,
and `record.user_explicit` count emitted into `writes.user_explicit`.
**Run it after any change to `bin/lorgen-compile-metrics`** —
expected output: `OK: 5 smoke checks passed`.

## Reading metrics — recipes

Every reader MUST guard on `schema_version` (per the contract above)
so that a future v2 file does not silently mislead a v1 reader.

```bash
# Hot Knowledge in the last 30 days (top 20 by refs)
jq -s '
  [.[] | select(.schema_version==1) | .knowledge_refs | to_entries[]]
  | group_by(.key)
  | map({path: .[0].key, refs: ([.[].value.refs] | add), helpful_ratio: ([.[].value.helpful_true] | add) / ([.[].value.refs] | add)})
  | sort_by(-.refs) | .[:20]
' .lorgen/metrics/*.json

# Stale Knowledge — last_ts older than 90 days, refs > 0
jq -s --argjson cutoff $(date -u -v-90d +%s 2>/dev/null || date -u --date='-90 days' +%s) '
  [.[] | select(.schema_version==1) | .knowledge_refs | to_entries[]]
  | group_by(.key)
  | map({path: .[0].key, last_ts: ([.[].value.last_ts] | max), refs: ([.[].value.refs] | add)})
  | map(select(.refs > 0 and ((.last_ts | fromdateiso8601) < $cutoff)))
' .lorgen/metrics/*.json

# Trigger health — high-read, low-helpful Knowledge (candidates for trigger refinement)
jq -s '
  [.[] | select(.schema_version==1) | .knowledge_refs | to_entries[]]
  | group_by(.key)
  | map({
      path: .[0].key,
      refs: ([.[].value.refs] | add),
      helpful_ratio: (([.[].value.helpful_true] | add) / ([.[].value.refs] | add))
    })
  | map(select(.refs >= 5 and .helpful_ratio < 0.5))
  | sort_by(.helpful_ratio)
' .lorgen/metrics/*.json

# Unread Knowledge — items on disk that were never referenced in any
# tracked metrics file. The recipes above are blind to refs==0 because
# `knowledge_refs` only contains files that were actually read; this
# diff is the matching "stale because never touched" surface.
comm -23 \
  <(find .lorgen/knowledge -type f -name '*.md' | sort -u) \
  <(jq -r 'select(.schema_version==1) | .knowledge_refs | keys[]' \
       .lorgen/metrics/*.json 2>/dev/null | sort -u)
```

These recipes are the MVP trigger-health / stale-detection surface.
The Roadmap `lorgen consolidate` formalises them into an interactive tool.
