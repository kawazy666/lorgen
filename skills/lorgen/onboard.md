# Onboard — initial repo scan and backfill

The user can ask Lorgen to "onboard" a repo, which is a heavy first-pass
scan that produces the initial Knowledge and Wiki set. This is meant to
be run once per repo (and re-run after a long absence or when significant
history is missed).

There is no separate `lorgen onboard` shell command — the user simply asks
the `lorgen` agent to "onboard this repo" or "scan the last year". You
detect that intent and run this procedure.

## Default behaviour

Recent + shallow:

- **Time window**: last 30 days (or `sources.github.pr_lookback_days`)
- **PR cap**: 50 (or `sources.github.pr_lookback_count`)
- **Depth**: shallow Wiki — overview + one page per top-level module dir

Goal is to be **runnable in a few minutes** and produce a solid baseline
that the user can review.

## Backfill mode (`--since 1y`, `--chunk month` semantics)

When the user asks for a deeper backfill ("過去 1 年遡って" / "2026年全部"),
process in **chunks** to keep individual LLM calls bounded and to support
resume-on-interrupt.

### Chunk strategies

| User intent | `since` | `chunk` |
|---|---|---|
| Recent quarter | 90d | week |
| Past 6 months | 6m | month |
| Past year | 1y | month |
| Full history | 0 (= no `since`) | month, but cap at 24 chunks for sanity |

For each chunk:

1. Compute the date range `[chunk_start, chunk_end)`.
2. List merged PRs in that range:
   ```bash
   gh pr list --state=merged --limit 100 \
      --json number,title,author,mergedAt,url,baseRefName,labels \
      --search 'merged:>=YYYY-MM-DD merged:<=YYYY-MM-DD'
   ```
3. For each PR, run the **same accumulation pipeline** as `accumulation.md`:
   filter (Stage 1) → worth-saving gate (Stage 2) → dedup (Stage 3) →
   write (Stage 4).
4. Update `.lorgen/state.json`:
   ```json
   {
     "onboard": {
       "last_completed_until": "2026-03-31T23:59:59Z",
       "last_chunk_completed": "2026-03",
       "in_flight": false
     }
   }
   ```
5. Move to the next chunk. **Save state between chunks** so a Ctrl-C or
   token-budget exhaustion is survivable.

### Resume

When the user re-invokes onboard with `--resume`-equivalent, read
`.lorgen/state.json`:

```python
state = json.load(open(".lorgen/state.json"))
resume_from = state["onboard"]["last_completed_until"]
# Continue from resume_from
```

If `state.json` is absent or has no `onboard` block, treat it as a fresh
run from the requested `since`.

### Logging

For each chunk, after completion, emit:
- One `onboard.chunk_completed` log event per `logging.md` with chunk
  range, PR count, and added/updated/skipped Knowledge counts.
- The accumulation pipeline's normal events (`gate.decision`,
  `dedup.decision`, `knowledge.created` etc.) for each PR processed.

Logs accumulate in `.lorgen/logs/<YYYYMMDD>.jsonl` as the run progresses,
so a Ctrl-C mid-chunk leaves a partial but readable audit trail.

### Progress reporting

Emit one line of stdout **per chunk completed**, not per PR:

```
Chunk 2026-04: 12 PRs scanned, 4 Knowledge items added, 1 updated.
Chunk 2026-03: 18 PRs scanned, 6 Knowledge items added, 2 updated.
...
```

Detail (which Knowledge files, which PRs) goes to `.lorgen/logs/`.

## What gets generated on a fresh `onboard`

Always:

1. **`.lorgen/wiki/overview.md`** — top-level summary derived from README,
   `pyproject.toml` / `package.json` / `Cargo.toml` / etc., and the
   directory layout. One section: "what this repo is", "main components",
   "entry points".

2. **`.lorgen/wiki/modules/<name>.md`** — one shallow page per top-level
   module / directory under `src/` (or the project's equivalent). Each
   page: purpose, main files, key types/functions, links to relevant
   Knowledge.

3. **Knowledge from convention files** — read these and convert each into
   `.lorgen/knowledge/conventions/<slug>.md` if not already present:
   - `CLAUDE.md`
   - `AGENTS.md`
   - `.cursorrules`, `.windsurfrules`
   - `CONTRIBUTING.md` (sections about code style / conventions)
   - `README.md` (sections like "design principles", "trade-offs")

4. **Knowledge from existing ADRs** — for each ADR file under
   `sources.adr_dirs` (default `.lorgen/adr/`, `docs/adr/`,
   `docs/decisions/`), create a `decision`-kind Knowledge item with
   `sources: [{type: adr, path: ...}]`. Skip if already covered.

5. **Knowledge from recent PRs** — apply the accumulation pipeline to
   each PR in the time window.

6. **Knowledge from intent-rich code comments** — apply the accumulation
   pipeline to TODOs / FIXMEs / "why:" comments in the codebase.

Optional (off by default — opt-in via `wiki.pages` in config):

7. **Steered Wiki pages** — if `.lorgen/config.yaml` → `wiki.pages` lists
   specific pages, generate exactly those.

8. **ADR scaffolding** — if `.lorgen/config.yaml` → `outputs.write_adr: true`,
   any new `decision`-kind Knowledge produced by onboard is mirrored
   into `<outputs.adr_dir>/NNNN-<slug>.md` (default `.lorgen/adr/`) per
   `accumulation.md` → "Stage 5". If the directory doesn't exist yet,
   create it. When `outputs.adr_dir` points outside `.lorgen/` (e.g.
   `docs/adr/`), inform the user — adding a top-level `adr/` directory
   to a repo that didn't have one is a meaningful structural change
   worth surfacing.

## Quick mode (`--quick` semantics)

For a fast onboard (`--quick` or just "do a light scan"):

- Skip step 5 (recent PRs)
- Skip step 6 (code comments)
- Just generate steps 1, 2, 3, 4 — those are local file reads, no
  network, fast.

This is the right default when the user just wants to see the
directory layout and convention files normalised into `.lorgen/`.

## After onboard

End the run with a one-paragraph summary in stdout:

```
Onboard complete.
- Wiki pages created: 4 (overview, modules/auth, modules/billing, modules/api)
- Knowledge added: 12 (conventions: 4, decisions: 6, runbooks: 2)
- Knowledge skipped (already present or not worth saving): 8
- PRs scanned: 47

Review with: git status / git diff .lorgen/
Commit when ready.
```

The summary is for stdout. Detailed per-event records are already in
`.lorgen/logs/` from the chunk loop above; do not re-log the summary.

## Things `onboard` does NOT do

- Does not commit. The user does that.
- Does not modify the user's source code (no comment additions, no
  rewrites). The `.lorgen/` directory is the only write target.
- Does not delete previously-generated Knowledge or Wiki content. If you
  detect that an existing file is now redundant or wrong, surface the
  observation in the summary; let the user decide.
- Does not run `git push` / `gh pr create` / anything network-write.
