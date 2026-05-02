---
trigger: "rename, Mimir, Lorgen, Lordgenome, naming, rebrand, brand identity"
kind: decision
scope: repo
tags: [meta, naming]
sources:
  - {type: commit, ref: "47bc509"}
  - {type: user-record, ref: "命名を変更しよう lorgen 天元突破紅蓮らがんに登場する生体コンピュータからとっている"}
---

# Decision — rename Mimir → Lorgen

## What

Renamed the entire plugin from **Mimir** (Norse oracle) to **Lorgen**
(short for **Lordgenome** / ロージェノム — the biological supercomputer
in *Tengen Toppa Gurren Lagann*). All file/directory names, identifiers,
env vars, paths, prose, commit messages, and the GitHub repository
itself (`kawazy666/mimir` → `kawazy666/lorgen`).

## Why

- **Identity over allusion** — Mimir is a popular knowledge-related name
  (Apache Mimir, Grafana Mimir, multiple ML projects). Lorgen has
  effectively zero name-collision in software tooling and gives the
  project a unique brand handle.
- **Domain narrative fit** — Lordgenome is a *biological* compute
  archive of the Spiral race's accumulated knowledge — read by
  successors to inform decisions. Lorgen plays an analogous role for
  a code repository.
- **One short, sayable handle** — `lorgen`, `@lorgen`, `/lorgen:review`,
  `.lorgen/`, `bin/lorgen-*`, `LORGEN_*` all work as prefixes without
  feeling forced.

## Trade-off accepted

- Anyone with the previous `kawazy666/mimir` install is broken until
  they re-add the new marketplace. We accept this for a v0.x project
  with no known external users.
- The Norse-oracle metaphor in the old README intro is gone; the new
  intro leans on the Gurren Lagann lore (Spiral Logic AI). Slightly
  more niche, but the product is named after a fictional bio-computer
  either way — niche is the point.

## Mechanics

- Three-pass case-sensitive sed: `MIMIR_` → `LORGEN_`, `Mimir` →
  `Lorgen`, `mimir` → `lorgen` (order matters — wider-net first would
  break the env-var case).
- File renames preserved git history via `mv` followed by `git add -A`
  (rename detection picked up most files at >50% similarity threshold).
- `gh repo rename lorgen` for the remote, `git remote set-url` for the
  local. GitHub auto-redirects the old URL for a grace period.

## Where to read more

- Commit `47bc509` — the rename + the new review skill in one shot.
- `README.md` intro — the new lore paragraph.
- `assets/logo.png` — the spiral / eye logo.
