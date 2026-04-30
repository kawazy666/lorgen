---
trigger: "pivot, why subagent, why not Python, Claude Code skill, design history"
kind: decision
scope: repo
tags: [meta, architecture]
sources:
  - {type: user-record, ref: "design hearing 2026-04-28 / 2026-04-29"}
  - {type: code, path: "docs/architecture.md"}
---

# Decision — pivot from Python CLI to Claude Code subagent + skill

## What

Mimir was originally implemented as a standalone Python CLI calling the
Anthropic API directly. We replaced that with a Claude Code subagent
plus a multi-file skill (later packaged as a plugin — see
`package-as-plugin.md` for the next step).

## Why

Three forces drove the pivot:

1. **Cost duplication** — every CLI invocation made the calling agent
   (Claude Code) pay for one LLM round-trip and Mimir itself pay for
   another with a second API key. The subagent runs inside the caller's
   session, single LLM, single cost.
2. **Setup friction** — Python CLI required `pip install`,
   `ANTHROPIC_API_KEY`, a venv. Subagent setup is `cp -r` + an install
   script (later replaced by `/plugin install` — see follow-up decision).
3. **Iteration speed** — subagent + skill is Markdown prompts. Edit
   the skill files, restart Claude Code, done. No code, no tests, no
   release pipeline.

## Trade-off accepted

Subagent form is **Claude Code only**. Cursor / aider / CI / cron
callers cannot use it. We accept this for MVP because the primary user
is Claude Code, and a Python CLI on the same `.mimir/` data model can
be added later as a Roadmap item without changing any Knowledge file.

## What it ruled out

- Vector / RAG-based retrieval — Mimir leans on `ripgrep + LLM
  selection` over the structured Knowledge files, matching the
  Cursor / Claude Code / Devin approach.
- A separate Mimir daemon — the subagent is single-shot per
  invocation; "always-loaded state" is achieved by the skill loading
  `.mimir/` on demand.

## Where to read more

- `docs/architecture.md` — long-form record of every alternative
  considered.
- `.mimir/wiki/architecture.md` — pointer index.
- [decision: package as plugin](package-as-plugin.md) — the follow-up
  pivot from "subagent + standalone install.sh" to "Claude Code plugin".
