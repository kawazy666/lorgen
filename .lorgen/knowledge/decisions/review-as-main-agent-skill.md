---
trigger: "/lorgen:review, review entry point, why slash command, subagent Task restriction, main agent skill"
kind: decision
scope: repo
tags: [meta, architecture, review]
sources:
  - {type: commit, ref: "47bc509"}
  - {type: code, path: "skills/review/SKILL.md"}
  - {type: code, path: "commands/review.md"}
  - {type: user-record, ref: "claude-code-guide確認: subagentはTaskを呼べない (Claude Code 2.x)"}
---

# Decision — review skill runs in the main agent context, invoked via `/lorgen:review` only

## What

The Knowledge-grounded code review feature lives at
`skills/review/SKILL.md` (top-level plugin skill, NOT inside
`skills/lorgen/`). The canonical entry point is the slash command
`/lorgen:review` (file `commands/review.md`, plugin-namespaced as
`<plugin>:<command>`). `@lorgen review` is intentionally **not
supported** — the Lorgen subagent redirects review requests back to
`/lorgen:review` and exits.

## Why

A hard platform constraint forces this shape: **subagents on Claude
Code 2.x cannot call the `Task` tool**. Documentation explicitly
states `Subagents cannot spawn other subagents`. The review feature
is built as a 4-role multi-agent team — Coordinator + Knowledge
Searcher + parallel Knowledge Investigators + Code Comparator — and
spawning the worker subagents requires `Task`. That capability is
only available in the main Claude Code agent.

So the Coordinator MUST be the main agent. The slash command
mechanism is the natural way to hand control from the user to the
main agent in a structured way (vs. `@subagent` which would invoke
the Lorgen subagent and lose the ability to spawn workers).

## Why NOT a degraded `@lorgen review` fallback

We considered shipping a single-context, no-parallelism degraded
review path on `@lorgen review`. Rejected because:

- Same name, two very different behaviours = high cognitive load.
- The 4-role split is the entire point — degrading it removes the
  context-isolation that prevents file-body bloat in the
  Coordinator's window.
- One canonical entry point is easier to document, test, and
  protect (the untrusted-data primer + `commands/review.md` opaque
  flag fence + Step-0 regex validation all assume one path).

## Trade-off accepted

- Users who naturally type `@lorgen review` are bounced — Lorgen's
  subagent has explicit redirect copy in `agents/lorgen.md` and
  `skills/lorgen/SKILL.md` operating loop.
- If Anthropic ever lifts the subagent-Task restriction, we could
  add `@lorgen review` as a true alias. Forward-compat is fine
  because the entry point split is documentation-only — the skill
  bundle does not assume an `@lorgen` caller.

## Where to read more

- `skills/review/SKILL.md` § "When this skill activates" + § "What
  review does NOT do".
- `docs/architecture.md` § "Why `/lorgen:review` and not `@lorgen
  review`".
- [decision: four-role-multi-agent-review-team](four-role-multi-agent-review-team.md).
