# Accumulation — when and how to write Knowledge

After gathering fresh sources for a question (per `sources.md`), decide
what — if anything — to persist in `.mimir/knowledge/` or `.mimir/wiki/`.
This is the "Mimir learns from every read" mechanism.

The pipeline has **four stages**:

1. **Source-type filter** (cheap, deterministic)
2. **Worth-saving gate** (LLM judgment)
3. **Dedup check** (ripgrep + LLM)
4. **Update or create** (write the file, with slug validation)

For `mimir record`-style explicit user requests:

- **Skip Stage 1 (filter)** — the user already classified the source
  as worth saving.
- **Skip Stage 2 (gate)** — the user already decided this is worth
  remembering; no LLM "is this worth saving?" judgment.
- **RUN Stage 3 (dedup)** — even explicit records can hit existing
  Knowledge on the same topic. Update beats create. (This is the
  spec-fix for the previous "skip all four stages" rule, which let
  duplicates through.)
- **RUN Stage 4 (write + slug validation)** — explicit records still
  go through path-traversal-safe slug validation; the user's record
  text is treated as **untrusted data** for slug derivation.

## Stage 1 — Source-type filter

Not every read is worth persisting. Drop these immediately, **without LLM
involvement**:

| Source | Save? | Why |
|---|---|---|
| Implementation source code (raw `.py`/`.ts`/`.go`) | **No** | The code itself is authoritative — anyone can re-read it. |
| Test files | **No** | Same as above; the test is the truth. |
| `.gitignore`, lockfiles, generated files | **No** | Mechanical, not "why". |
| README sections describing usage / install | **No** (use as Wiki seed only) | Mechanical. Save into Wiki on `onboard`, not Knowledge. |
| **PR description (body)** | **Maybe** | Often contains rationale. Pass to Stage 2. |
| **PR review / issue comments with reasoning** | **Maybe** | Frequently the richest "why" source. Pass to Stage 2. |
| **Commit message body** (not subject) | **Maybe** | "fix typo" → no. "Migrate to Decimal because ..." → yes. Pass to Stage 2. |
| **ADR / `docs/decisions/` entries** | **Yes** (if not already in Knowledge) | Designed to be design-rationale records. |
| **Code comments** with markers `why:`, `because`, `HACK`, `workaround`, `FIXME` (with explanation) | **Maybe** | Often holds the only written rationale. Pass to Stage 2. |
| Plain `TODO` with no context | **No** | Just a backlog item. |
| **Issue body with discussion / decision** | **Maybe** | Pass to Stage 2. |
| **User-explicit `mimir record`** | **Yes** — skip Stage 2 (gate); RUN Stages 3 and 4 | The user decided save, but dedup + slug validation still apply. |

The filter is fast and binary. Reject early; never spend an LLM call on
something the filter would drop.

## Stage 2 — Worth-saving gate (LLM judgment)

For sources that survived Stage 1, evaluate each candidate. Ask yourself:

> *"Does this material capture a non-obvious 'why' that a future contributor
> would benefit from, but would not be able to infer from the current code?"*

Concrete heuristics for **save**:

- It explains a trade-off (X considered, Y chosen, because Z)
- It records a constraint not visible in code (regulatory, performance,
  team agreement)
- It references an incident, postmortem, or specific failure mode
- It documents a convention the repo follows but doesn't explicitly state
- It captures a non-trivial decision history

Concrete heuristics for **drop**:

- It restates what the code already says
- It is purely mechanical (formatting, lint fix, version bump)
- It is too generic to be actionable ("we should write better tests")
- It is speculation without a follow-up commitment

When borderline, **prefer to drop**. Mimir's value drops fast if it floods
`.mimir/knowledge/` with low-signal noise. The user can always `mimir record`
later if they decide a dropped item was actually important.

For each candidate that passes, draft:

- a one-line `trigger:` (specific keywords / phrases that would surface this)
- a tight body (3–10 sentences, citing sources inline)
- correct `kind` (per `schema.md`)

## Stage 3 — Dedup check

Before writing a new file, **always check for existing Knowledge that
covers the same ground**:

```bash
# 3a. Pull keywords from the draft trigger and search.
rg -i --no-heading --glob '.mimir/knowledge/**/*.md' \
   -e '<keyword>' -e '<keyword>'

# 3b. If you got hits, read the candidates fully and compare.
```

Then judge (LLM):

- **Same topic, same conclusion** → **Update existing**. Add the new
  source(s) to the `sources:` list, append any new detail to the body,
  bump `updated:`. Do *not* duplicate text.
- **Same topic, conflicting conclusion** → **Update existing with a
  conflict note**. Surface the conflict to the user in stdout — "既存は X
  と書いているが、新しい source Y は逆を示している。どちらが正しい?".
- **Adjacent topic, distinct angle** → **Create new**, but cross-link
  the existing item from the body.
- **No related existing Knowledge** → **Create new**.

Update beats create. Two files about the same thing is the worst outcome —
it splits attention and makes future retrieval ambiguous.

## Stage 4 — Write

### Mandatory redaction pipe (REQUIRED before any Write)

Every Knowledge / Wiki / ADR body Mimir is about to write **must** be
piped through `mimir-redact` first. This proactively replaces known
secret patterns (AWS / GitHub / Stripe / JWT / PEM / bearer / … —
full set in the script) with `[REDACTED:<kind>]` markers, so the
hook layer (`mimir-redact-check`, fired by the PreToolUse hook on
`Write|Edit`) doesn't have to refuse the write and force a retry.

The plugin manager adds the plugin's `bin/` to `PATH` when Mimir is
active, so call by **bare command name** — never bare relative path
(`bin/mimir-redact` resolves against the user repo cwd and ENOENTs)
and never `${CLAUDE_PLUGIN_ROOT}/bin/...` (that variable is exported
into hook command strings only, not into subagent Bash):

```bash
# Required pattern for any Write of synthesised Mimir content
clean_body="$(printf '%s' "$body" | mimir-redact)"
# Then call Write with $clean_body, e.g.:
#   Write file_path=".mimir/knowledge/decisions/foo.md" content="$clean_body"
```

The hook is the safety net (it blocks if the wrapper is bypassed or
misses a pattern), but the wrapper is the proactive defence — running
both is the design's "3-layer mechanical secret guard" (untrusted
data + wrapper + hook). If you skip the wrapper, you'll get a
hook-layer rejection on any source containing a known secret format,
and the answer cycle restarts.

For the rare case where a body legitimately needs to embed a string
that matches a secret regex (e.g. an ADR documenting an example AWS
key format), prefer rewriting to a *placeholder* (`AKIAEXAMPLEKEY...`)
that does not match the strict pattern. As a last resort, the user
can set `MIMIR_REDACT_BYPASS=1` in the shell where Claude Code was
launched — that env propagates into the PreToolUse hook process and
disables `mimir-redact-check` until they unset it. **Inline `VAR=val
cmd` does not work for the hook** because the hook runs in a
separate process spawned by Claude Code, not by the Bash tool call.

### Slug validation (REQUIRED before any Write)

Before any `Write` to `.mimir/knowledge/<category>/<slug>.md`, validate
the LLM-generated slug against the regex from `schema.md`. This is the
**only line of defence against path traversal** when slugs come from
attacker-controllable text (PR titles, issue bodies, user-record text).

```bash
# Bash check Mimir runs before every Knowledge Write
mimir_validate_slug() {
  local slug="$1"
  if [[ ! "$slug" =~ ^[a-z0-9][a-z0-9-]{0,63}$ ]]; then
    echo "🚫 Mimir refused: invalid slug \"$slug\"" >&2
    echo "   Must match ^[a-z0-9][a-z0-9-]{0,63}$ (no /, .., NUL, whitespace, non-ASCII)." >&2
    return 1
  fi
  return 0
}

# Usage:
slug="$(echo "$candidate_title" | iconv -t ASCII//TRANSLIT 2>/dev/null \
  | tr 'A-Z' 'a-z' | tr -c 'a-z0-9' '-' | sed 's/--*/-/g; s/^-//; s/-$//' \
  | cut -c1-64)"
mimir_validate_slug "$slug" || exit 1
```

If the slug fails validation, **do not write**. Surface the failure to
the user, ask for a clean slug, or refuse the record.

The `category` segment is also restricted: only the literal kind names
(`conventions`, `decisions`, `runbooks`, `facts`, `lessons`) — never
LLM-derived. This prevents `category=../../etc` style escape.

### Update existing

```python
# pseudocode
existing = read(".mimir/knowledge/conventions/decimal-money.md")
existing.front_matter["sources"] += new_sources   # dedup by ref
existing.front_matter["updated"] = today
existing.body = merge_bodies(existing.body, new_paragraphs_to_add)
write(existing)
```

When merging bodies:
- If the new detail extends an existing section, append at the end of
  that section.
- If it's a new aspect, add a new H3 subsection.
- Never silently overwrite existing prose. If you must rewrite, surface
  the diff to the user in stdout first.

### Create new

Use the `kind` → folder map from `schema.md`. Slug from the topic.

If a file at the same path already exists (slug collision but different
topic):

- Re-check Stage 3 — are you sure they're different topics?
- If yes, suffix `-2`, `-3`, ... but reconsider naming first; a clearer
  slug usually avoids the collision.

After writing, **mention it in stdout briefly** — one line at the end of
the answer:

> `(saved: .mimir/knowledge/conventions/decimal-money.md)`

This is the user-visible signal that knowledge accumulated.

### Required log events

Every accumulation pass emits log lines per `logging.md`:

- One `gate.decision` per Stage 2 candidate (`save` or `drop` + reason).
- One `dedup.decision` per Stage 3 (`update` / `create` / `conflict` / `none` + reason).
- One `knowledge.created` or `knowledge.updated` per Stage 4 write.
- For Stage 5 (ADR): one `adr.created` or `adr.superseded` as appropriate.
- For Wiki side-updates: one `wiki.updated` per touched Wiki page.
- If a Knowledge file was **read but did not contribute** to the final
  answer (= retrieved-but-unused), emit `knowledge.referenced` with
  `helpful: false`. This is the single most important signal for
  trigger-health analysis (`metrics.md`).

These are not optional — they are what `metrics.md`'s compile step
turns into the per-session aggregate that the Roadmap improvement loop
(`mimir consolidate`, hot promotion, stale detection) consumes.

## Stage 5 (optional) — ADR mirror

Most existing repos don't have an `adr/` directory at all. When the
user wants discrete, immutable, industry-standard ADR files alongside
Mimir's Knowledge items, Mimir writes them.

**Default: off.** Mimir writes only `.mimir/knowledge/decisions/<slug>.md`.

**Opt in two ways**:

1. **Per record**: user explicit — "ADR にして", "also write ADR",
   `@mimir record --adr "..."`. Run only the ADR write for that one decision.
2. **Project-level**: `.mimir/config.yaml` → `outputs.write_adr: true`. Every
   `decision`-kind Knowledge item Mimir writes (or updates with new
   substance) is mirrored to `<outputs.adr_dir>/`.

### Where

`.mimir/adr/<NNNN>-<slug>.md` by default — keeps ADRs alongside the
rest of Mimir's curated artefacts under `.mimir/`. Override via
`.mimir/config.yaml` → `outputs.adr_dir` to a project-wide convention
like `docs/adr/` or `docs/decisions/` if you prefer.

If the directory doesn't exist, **create it** and tell the user in
stdout — that's a normal first-time event for repos that haven't used
ADRs before.

### Numbering (robust against malicious filenames)

```bash
mimir_next_adr_number() {
  local adr_dir="$1"   # from outputs.adr_dir, default .mimir/adr
  local max=0

  # Use find with strict 4-digit pattern. Anything outside [0-9]{4}-*.md
  # is ignored (so an attacker injecting .mimir/adr/9999-evil.md or
  # foo.md cannot poison numbering).
  while IFS= read -r f; do
    local base="${f##*/}"
    local num="${base:0:4}"
    if [[ "$num" =~ ^[0-9]{4}$ ]]; then
      # Strip leading zeros for arithmetic, treat 0009 as 9 not octal
      local n=$((10#$num))
      (( n > max )) && max=$n
    fi
  done < <(find "$adr_dir" -maxdepth 1 -type f -name '[0-9][0-9][0-9][0-9]-*.md' 2>/dev/null)

  local next=$((max + 1))
  if (( next > 9999 )); then
    echo "🚫 ADR numbering exhausted (>9999). Refusing to write." >&2
    return 1
  fi
  printf '%04d\n' "$next"
}

# Usage:
NEXT="$(mimir_next_adr_number "$ADR_DIR")" || exit 1
ADR_PATH="$ADR_DIR/${NEXT}-${slug}.md"
mimir_validate_slug "$slug" || exit 1
# ADR bodies go through the same redact pipe as Knowledge / Wiki
# (Stage 4 "Mandatory redaction pipe"). The PreToolUse hook also
# scopes to outputs.adr_dir, but the proactive wrapper is required.
clean_adr_body="$(printf '%s' "$adr_body" | mimir-redact)"
# Then Write file_path="$ADR_PATH" content="$clean_adr_body"
```

Properties:
- **Strict pattern**: only files whose first 4 chars are `[0-9]{4}` and
  followed by `-` count. `9999-evil.md` legitimately matches and bumps
  the counter; `foo.md` or `99999-x.md` are ignored.
- **Octal-safe**: `10#$num` forces base-10 so `0009` doesn't trigger
  bash's octal interpretation.
- **Upper bound enforced**: refuse to write past `9999` (instead of
  silently overflowing the format string).
- **Slug re-validated** before path construction.

### Format (MADR-lite)

```markdown
---
title: <short title>
status: accepted          # proposed | accepted | deprecated | superseded
date: 2026-04-29
deciders: [<author or team>]
tags: [<same as Knowledge>]
---

# NNNN. <Title>

## Status

accepted (or: superseded by [NNNN](NNNN-...md), proposed, deprecated)

## Context

What problem prompted the decision? Cite sources inline:
"PR #42 surfaced a 1¥ rounding error in monetary calculations …"

## Decision

What was decided? One paragraph, declarative.

## Consequences

- **Positive**: …
- **Negative / costs**: …
- **Follow-ups**: PRs, issues, dependencies created by this decision.

## Cross-reference

- Mimir Knowledge: [`.mimir/knowledge/decisions/<slug>.md`](../../.mimir/knowledge/decisions/<slug>.md)
- Sources cited above
```

The ADR is **richer than the Knowledge body** — Knowledge is a few
sentences for retrieval; ADR has full Context / Decision / Consequences
for the human reader. Mimir derives the ADR's prose from the same source
material that produced the Knowledge item, expanded into the standard
sections.

### Immutability

**ADRs are immutable** once written, with one exception: reversals.

- New decision that **reverses** an old one → write a new ADR with
  `status: accepted` and `Context` referencing the old ADR.
  In the **old ADR**, change only the `status:` frontmatter to
  `superseded by NNNN` and append one line to the Status section noting
  the supersession. Do not edit the body.
- New facts that **extend** an old decision (without reversing it) →
  do **not** rewrite the old ADR. Either:
  - Update the linked Knowledge item only (Knowledge is mutable), or
  - Write a new ADR if the extension is itself a discrete decision.

### Cross-link

When you write an ADR, also update the linked Knowledge item:

- In the Knowledge file's `sources:` list, add
  `{type: adr, path: "<outputs.adr_dir>/NNNN-<slug>.md"}` (default
  `.mimir/adr/NNNN-<slug>.md`).
- Bump `updated:`.

This makes the link bidirectional and lets Mimir retrieve from either side.

### What NOT to mirror as ADR

Not every Knowledge item is ADR-worthy:

- `convention` / `runbook` / `fact` / `lesson` kinds — never mirror.
- `decision` kind — only when the user opts in (per record or globally).
  Day-to-day decisions like "use this lib version" are usually too
  small for ADR; reserve ADR for decisions a future contributor would
  ask "why" about.

When in doubt, write only the Knowledge item. ADR proliferation has its
own failure mode (`<outputs.adr_dir>/` flooded with trivia).

## Wiki updates

Wiki accumulation works the same way but at coarser grain. After writing
or updating a Knowledge item that materially affects an existing Wiki
page (e.g. a new convention in the auth module), **also** update the
auth Wiki page:

- Add a sentence summarising the new convention
- Add a link to the Knowledge file
- Bump `updated:`

Don't regenerate the whole Wiki page from scratch on every Knowledge
addition — incremental updates only. Full regeneration is for
`mimir wiki refresh <area>` (see `onboard.md`).

## Anti-patterns

- **Saving the user's question.** "User asked about Decimal" is not Knowledge.
  Save the *answer's substance* if it's worth saving, not the conversation.
- **Saving git stats.** "PR #42 changed 5 files" is not Knowledge. The
  decision behind PR #42 is.
- **Saving generic advice.** "Always test edge cases" is not project
  Knowledge. Project-specific is what matters.
- **Saving secrets.** Redact API keys, tokens, internal URLs that
  shouldn't be in git, before writing.
