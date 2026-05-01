---
trigger: "example, schema demo, what a Knowledge item looks like"
kind: convention
scope: repo
tags: [meta, example]
sources:
  - {type: user-record, ref: "initial scaffold"}
created: 2026-04-29
updated: 2026-04-29
---

# Example Knowledge item — schema demo

This file exists as a live example of the Knowledge item schema. Real Knowledge
items in a user's repo would replace files like this.

## Anatomy

The front-matter at the top of this file is structured per
`.claude/skills/lorgen/schema.md`:

- `trigger:` — the keywords/phrases Lorgen will match against to decide whether
  this item is relevant to a given user question. Make it specific.
- `kind:` — one of `convention`, `decision`, `runbook`, `fact`, `lesson`.
  Determines the default folder under `.lorgen/knowledge/`.
- `scope:` — `repo` (default) or `global`.
- `tags:` — optional free-form labels.
- `sources:` — required. Each source has a `type` and at least one of
  `ref`, `path`, or `url`. Examples:
  - `{type: pr, ref: "#42", url: "..."}`
  - `{type: commit, ref: "abc1234"}`
  - `{type: adr, path: ".lorgen/adr/0003-decimal.md"}`
  - `{type: code, path: "src/auth/session.py", line: 42}`
  - `{type: user-record, ref: "initial scaffold"}`
- `created:` and `updated:` — ISO dates.

## Body

The body is just Markdown. Keep it tight — a few sentences to a couple of
paragraphs. If you want to write more, it probably belongs in a Wiki page
(`.lorgen/wiki/`) instead.

## When to delete this file

This is scaffolding. In your own project repo, delete this file once you
have at least one real Knowledge item.
