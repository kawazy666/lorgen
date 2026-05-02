---
trigger: "review architecture, multi-agent, Coordinator, Searcher, Investigator, Comparator, role separation"
kind: decision
scope: repo
tags: [meta, architecture, review]
sources:
  - {type: commit, ref: "47bc509"}
  - {type: code, path: "skills/review/SKILL.md"}
  - {type: user-record, ref: "ユーザー要望: ちゃんとした多役割レビュー / search → 並列調査 → 照合"}
---

# Decision — review uses a 4-role multi-agent team (Coordinator + Searcher + parallel Investigators + Comparator)

## What

`/lorgen:review` orchestrates four distinct roles, three of which run
as `Task`-spawned sub-agents in isolated contexts:

| Role | Spawn count | Job |
|---|---|---|
| **Coordinator** | 1 (the main agent itself) | Parse CLI, collect diff, orchestrate, render report, accumulate |
| **Knowledge Searcher** | 1 (Task) | Grep `.lorgen/{knowledge,wiki,adr}` for relevance candidates |
| **Knowledge Investigator** | N (Task, parallel in 1 message) | Deep-read ONE candidate, extract rules + outdated_signals |
| **Code Comparator** | 1 (Task; 2 if rules>50) | Walk each rule against diff hunks, emit findings + summary |

## Why

Two failure modes drove the split:

1. **Context bloat.** Stuffing the diff + every Knowledge file into
   one prompt blows the budget and produces shallow reasoning across
   too much material. The Investigator-per-candidate pattern keeps
   each sub-agent's window holding ONE file — bodies summarise into
   small `rules` arrays, only summaries return.
2. **Task interleaving.** Triaging relevance, deep-reading, and
   judging code conformance are three different cognitive modes that
   interfere. Splitting them into discrete passes — each in its own
   sub-context — produced visibly better findings during dogfood.

## Trade-offs accepted

- **Latency**: 4 sequential phases (Searcher → Investigators →
  Comparator → render) is slower than one big call. The parallel
  Investigator phase amortises this — N candidates take ≈ one
  candidate's wall time. We optimise for finding quality, not
  minimum latency.
- **Token cost**: more agents = more prompt overhead. Mitigated by
  (a) `--depth quick|standard|deep` capping candidate count
  (5 / 12 / 25), (b) Investigators receive a hunks-summary not the
  full diff (Comparator alone gets the full diff, capped at 50 KB).
- **Sub-agent capability surface**: `general-purpose` Task spawns
  have full tool surface. The "Do NOT write/run" lines in the
  prompts are soft defences. Mitigations documented in
  `skills/review/SKILL.md` § "Residual risks the primer alone cannot
  close" — tool-restricted spawn (preferred), render-time redact-
  check, fenced rendering of all sub-agent text in stdout.

## Why this specific split (not 2 or 5)

- **2 roles** (Search+Investigate combined → Comparator) — loses
  parallelism because one agent reads all candidates serially in its
  own context, defeating the bloat-mitigation purpose.
- **5+ roles** — adding a separate "Outdated Detector" or "Severity
  Calibrator" role was considered. Rejected for MVP because
  Investigators already produce `outdated_signals` cheaply and
  Comparator already calibrates severity from `severity_hint`. Added
  roles would be ceremonial without unique value.

## Where to read more

- `skills/review/SKILL.md` — full role prompts and Coordinator handling.
- `docs/architecture.md` § "Why a 4-role multi-agent team for review".
- [decision: review-as-main-agent-skill](review-as-main-agent-skill.md)
  — why the Coordinator must be the main agent, not the Lorgen
  subagent.
