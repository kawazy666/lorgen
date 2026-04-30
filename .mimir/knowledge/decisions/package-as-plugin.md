---
trigger: "plugin, claude code plugin, distribution, install, /plugin install, marketplace"
kind: decision
scope: repo
tags: [meta, architecture, distribution]
sources:
  - {type: user-record, ref: "design hearing 2026-04-29"}
  - {type: code, path: "docs/architecture.md"}
  - {type: code, path: ".claude-plugin/plugin.json"}
---

# Decision — package Mimir as a Claude Code plugin

## What

We moved from "standalone subagent + `install.sh` symlinking into
`.claude/`" to a proper Claude Code plugin (`.claude-plugin/plugin.json`,
`agents/`, `skills/mimir/` at repo root). Distribution is now via
`/plugin install mimir@<github-url>`.

## Why

1. **One-command install** — users run `/plugin install` and Claude Code
   handles file placement under `.claude/plugins/mimir/`. No manual
   `cp -r` + `bash install.sh` ritual.
2. **Auto-update** — Claude Code 2.0.70+ auto-updates plugins on
   startup, so improvements reach users without manual re-copy.
3. **Versioning** — plugin manifest carries semver; users can pin or
   follow HEAD per their tolerance for change.
4. **Marketplace discoverability** — a plugin can be listed in
   community marketplaces (`anthropics/claude-plugins-official` etc.)
   and reach users we don't know about.
5. **Cleaner separation** — plugin source (`agents/`, `skills/`) is
   distinct from data store (`.mimir/`). Plugin lives in
   `~/.claude/plugins/mimir/` (global) or `<repo>/.claude/plugins/mimir/`
   (project install); user repos only need `.mimir/` for accumulated
   knowledge.

## Trade-off accepted

- **Hard dependency on Claude Code 2.0.70+**. Older Claude Code
  installs cannot use the plugin form. Acceptable: plugin support
  shipped widely before this pivot.
- **Standalone subagent install path is gone** — repos that previously
  used `cp -r .tim/` + `install.sh` (the prior structure) need to
  migrate to `/plugin install`. Their accumulated `.mimir/knowledge/`
  and `.mimir/wiki/` data are unaffected, only the install workflow
  changes.

## What changed in this repo

- `.tim/agent/tim.md` → `agents/mimir.md`
- `.tim/skill/*.md` → `skills/mimir/*.md`
- `.tim/install.sh`, `.tim/uninstall.sh` → deleted (`/plugin install` /
  `/plugin remove` replace them)
- New: `.claude-plugin/plugin.json`
- `.tim/` → `.mimir/` (data store rename, content preserved)

## Where to read more

- `docs/architecture.md` → "Why a Claude Code plugin"
- `.claude-plugin/plugin.json` — manifest
- [Anthropic — Create plugins](https://code.claude.com/docs/en/plugins)
- [Anthropic — Plugins reference](https://code.claude.com/docs/en/plugins-reference)
