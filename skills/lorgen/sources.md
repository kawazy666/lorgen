# Sources — gathering fresh material

When existing Knowledge doesn't answer the question (per `retrieval.md`),
gather material from the underlying sources. Five collectors, listed in
roughly increasing cost / depth:

1. **Code comments** (cheap, often surprisingly informative)
2. **Local ADR files** (`.lorgen/adr/`, `docs/adr/`, `docs/decisions/`) (cheap, high signal)
3. **Git log** (cheap, structural)
4. **Git blame** (targeted, useful when the question is about a specific
   line / function)
5. **GitHub PRs and issues** (more expensive — network calls; cache aggressively)

Always start with the cheapest applicable collector. Stop as soon as you
have enough to answer with confidence.

## 1. Code comments

The repo's own comments often answer the question directly when the
question is about a specific decision in code.

```bash
# Markers we treat as intent-rich (they almost always carry "why")
rg -i --no-heading --line-number \
   --glob '!.lorgen/' --glob '!node_modules/' --glob '!.venv/' \
   -e '\bwhy:' -e '\bbecause\b' -e '\bworkaround\b' \
   -e '\bHACK\b' -e '\bFIXME\b' -e '\bXXX\b'

# Scoped to a specific module
rg -i --no-heading --line-number \
   --glob 'src/auth/**' \
   -e 'why:' -e 'because' -e 'HACK'
```

Each hit is a `<file>:<line>: <text>` record. To get the surrounding
context, follow up with `Read` on a small window:

```
Read tool — file: src/auth/session.py, offset: <line - 5>, limit: 20
```

When citing, use `(src/auth/session.py:42)` in the answer.

## 2. ADR / decisions folder

Default scan paths (override via `.lorgen/config.yaml` → `sources.adr_dirs`):

```bash
ls .lorgen/adr/ docs/adr/ docs/decisions/ doc/adr/ adr/ 2>/dev/null
```

Each `*.md` is high-signal — these files exist *because* someone decided
to write down a "why". Read fully, cite by relative path:
`(.lorgen/adr/0003-decimal-money.md)` or `(docs/adr/0003-decimal-money.md)`
depending on where the repo keeps them.

## 3. Git log

For "since when" / "what changed" / "who decided" questions:

```bash
# Subject + body for the last N commits (default to bound)
git log --pretty=format:'%H%x00%h%x00%an%x00%aI%x00%s%x00%b' \
        --name-only -n 50

# Restrict to a path
git log --pretty=format:'%h %s%n%b' --name-only -n 30 -- src/billing/

# Restrict to a date range (matches the user's "since when" question)
git log --since='2026-01-01' --until='2026-03-01' \
        --pretty=format:'%h %s%n%b' --name-only

# Search commit messages for a keyword
git log --grep='Decimal' --pretty=format:'%h %s%n%b' --name-only
```

Pull the **body** (not just subject) — that's where rationale lives.

For very large histories, page through in chunks (`-n 30 --skip 0`,
`-n 30 --skip 30`, ...) instead of dumping everything.

## 4. Git blame

When the question is about a specific line / function / file ("なぜここ
こうなってる?"):

```bash
# Blame a single line
git blame -L 42,42 --porcelain src/billing/calc.py

# Blame a range
git blame -L 40,60 src/billing/calc.py
```

Blame gives you the commit hash; follow up with `git show` for the full
commit (message + diff):

```bash
git show <sha>
```

The commit message body usually carries the "why". The diff gives you
the "what changed alongside". Cite as `(commit abc1234)`.

## 5. GitHub PRs and issues — `gh` CLI

If `gh` is not installed or not authenticated, **degrade gracefully** —
note in the answer that GitHub data was not available, and proceed with
just git data.

```bash
# Detect availability
which gh && gh auth status >/dev/null 2>&1 && echo OK
```

### List recent merged PRs

```bash
gh pr list --state=merged --limit 50 \
   --json number,title,author,mergedAt,url,baseRefName,labels \
   --search 'merged:>=2026-04-01 merged:<=2026-04-30'
```

### Read a specific PR (with bodies, reviews, comments)

```bash
gh pr view 42 \
   --json number,title,author,mergedAt,url,body,baseRefName,headRefName,labels,reviews,comments
```

The PR body and review threads are gold for "why". Cite as `(PR #42)`.

### Read an issue

```bash
gh issue view 37 \
   --json number,title,author,createdAt,closedAt,url,body,labels,comments
```

Issue body + comments often contain the original requirement and
discussion. Cite as `(issue #37)`.

### Cache PR / issue fetches

The `gh` calls are network-bound and slow. Cache results into
`.lorgen/cache/sources/`:

```
.lorgen/cache/sources/pr/42.json      # raw `gh pr view --json` output
.lorgen/cache/sources/issue/37.json
```

Before fetching, check the cache:

```bash
test -f .lorgen/cache/sources/pr/42.json && cat .lorgen/cache/sources/pr/42.json
```

The cache is gitignored (per `schema.md`). Stale entries are fine for
historical PRs — PR #42 from 6 months ago doesn't change. For recent /
open PRs, refetch.

### Repo slug detection

If `.lorgen/config.yaml` doesn't pin `sources.github.repo`, derive it from
git:

```bash
git remote get-url origin
# https://github.com/owner/repo.git  →  owner/repo
# git@github.com:owner/repo.git      →  owner/repo
```

If the URL doesn't look like GitHub, skip the GitHub collector entirely.

## Cross-references

When a PR description references "see issue #37" or "follow-up to PR #15",
**follow the chain**. The original issue often has the requirement; the
PR has the implementation; an earlier PR may have the rejected
alternative. The "why" is usually distributed across the chain.

Bound the chain depth — don't recurse infinitely. Two hops is usually
enough.

## Logging

After a source-gathering pass (regardless of which collectors were
used), emit one `sources.gathered` log line aggregating what was
fetched — see `logging.md` for the exact shape. Aggregate by source
type so the log stays compact (one line per gathering pass, not one
per fetched item).

If a collector failed (e.g. `gh` not authenticated), also emit an
`error` log line with `fatal: false` and continue with the remaining
collectors.

## Stop when you have enough

You're not writing a Wikipedia article. Stop gathering when:

- You have a clear, citable answer to the user's specific question, or
- The sources you're consulting are diminishing-returns (mostly
  duplicating earlier sources or unrelated to the question).

It's better to deliver a tight, well-cited answer in 30 seconds than a
sprawling, exhaustive one in 5 minutes.
