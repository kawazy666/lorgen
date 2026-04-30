# Retrieval — looking up existing Knowledge

When the user asks Mimir a "why" / "history" / "convention" question, the first
move is to check whether the answer already lives in `.mimir/knowledge/`.
Only if it doesn't, fall through to fresh source-gathering (`sources.md`).

The retrieval method is **ripgrep + LLM selection**, *not* vector search.
Same approach Cursor / Claude Code / Devin use for code: deterministic, fast,
fails loudly with no false positives. (Reference: cognition.ai's argument
against RAG for code-heavy workloads.)

## Step 1 — Search existing Knowledge with ripgrep

Build a query from the user's question — extract the most distinctive keywords
(domain terms, identifiers, file/module names). Then:

```bash
rg --no-heading --line-number \
   --glob '.mimir/knowledge/**/*.md' \
   --max-count 3 \
   -e '<keyword1>' -e '<keyword2>' -e '<keyword3>'
```

For a question like "なぜ Decimal を使うのか?", run:

```bash
rg -i --no-heading --line-number --glob '.mimir/knowledge/**/*.md' \
   -e 'Decimal' -e 'money' -e '通貨'
```

Tactics:

- Run case-insensitive (`-i`) by default.
- Search **trigger lines first** with a tighter pattern, then expand to body
  if no hits:
  `rg -i --glob '.mimir/knowledge/**/*.md' '^trigger:.*Decimal'`
- Cap hits with `--max-count 3` per file — you don't need every line, just
  enough to identify the candidate file.
- Don't grep the whole repo. Scope to `.mimir/knowledge/`.

**When you read a candidate fully** (Step 2), emit a
`knowledge.referenced` (or `wiki.referenced`) log line per
`logging.md`. The `helpful` field is **load-bearing** for trigger
health analysis (`metrics.md`):

- `helpful: true` — this Knowledge / Wiki file's content materially
  contributed to the final answer you returned to the user.
- `helpful: false` — you read it (so it counts as `refs`) but its
  content did not appear in the final answer (= retrieved-but-unused).
  Repeated `helpful: false` on the same file is the signal that its
  `trigger` is over-broad and needs refinement.

Set the field at the **end of answer synthesis**, when you know which
candidates actually informed the answer. The metrics compile step
splits these into `helpful_true` / `helpful_false` per file — if you
collapse them by emitting only `true`, the trigger-health signal is
lost.

## Step 2 — Read full candidates

If ripgrep returned 1–5 candidate files, `Read` each one fully (they're
small — Knowledge items are intentionally a few sentences to a couple
paragraphs of body).

If ripgrep returned >5 candidates, you likely matched on a generic keyword.
Either:

- Narrow the keywords and re-run, or
- Read just the `trigger:` and `tags:` lines of each candidate first
  (`rg -A2 '^trigger:' .mimir/knowledge/...`) and pick the top 3 to read fully.

## Step 3 — Decide if Knowledge answers the question

For each candidate, ask yourself:

- Does this item's body actually answer the user's question, or is it just
  topically related?
- Is its `updated:` date recent enough that the cited sources are still
  authoritative? (If the item references PR #42 and it was updated 6 months
  ago, that's usually fine — PR #42 doesn't change. If it references "the
  current production database is Postgres" and was updated 2 years ago,
  re-verify against the current state of the repo.)

### If Knowledge answers fully

Return the answer in stdout, citing the Knowledge file path inline:

```markdown
## なぜ Decimal を使うのか

monetary 値は `decimal.Decimal` 必須。

2026-01 の incident — float の累積誤差で請求額が1円ズレた (PR #42)。
以後 `src/billing/` 配下は Decimal、test では `Decimal('1.00') == Decimal('1.0')`
で比較する。

(.mimir/knowledge/conventions/decimal-money.md)
```

**Do NOT trigger accumulation** in this path — there's no new information
to persist.

### If Knowledge partially answers

Return what Knowledge has, then gather fresh sources to fill the gap
(see `sources.md`). After answering, run accumulation to **update** the
existing Knowledge with the new detail (see `accumulation.md` → update vs
create).

### If no relevant Knowledge

Skip to `sources.md` to gather fresh material, then `accumulation.md` to
synthesise + persist + answer.

## Step 4 — When you should *also* check Wiki

Wiki pages are coarser than Knowledge items, but for **broad questions**
(e.g. "auth モジュール全体について教えて") Wiki is the right entry point:

```bash
rg -i --no-heading --line-number --glob '.mimir/wiki/**/*.md' \
   -e 'auth'
ls .mimir/wiki/modules/  # quick overview
```

Read the matching Wiki page first; it likely points to several Knowledge
items via internal links. Follow them.

## Failure modes — be loud

If ripgrep returns nothing and source-gathering also turns up nothing
substantive, **say so explicitly**:

> ".mimir/knowledge/ を検索したが Decimal に関する記録なし。
> git log と PR も確認したが、float→Decimal 移行は履歴に明示なし。
> `src/billing/` の現在のコードは Decimal を使っているが、移行の経緯
> は記録されていない。recordしますか?"

This is the correct behavior. Inventing a rationale to look helpful is the
wrong move — it pollutes future Knowledge with hallucinated history.
