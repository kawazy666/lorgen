---
trigger: "ADR location, outputs.adr_dir, .lorgen/adr, docs/adr, where ADRs live"
kind: decision
scope: repo
tags: [meta, schema, adr]
sources:
  - {type: commit, ref: "f873d47"}
  - {type: user-record, ref: "docの中のadrを.mimir配下にしよう"}
  - {type: code, path: ".lorgen/config.yaml"}
  - {type: code, path: "bin/lorgen-pretool-guard"}
---

# Decision — ADR default output location is `.lorgen/adr/`

## What

`outputs.adr_dir` defaults to `.lorgen/adr/` (was `docs/adr/`). All
Lorgen-curated artefacts now live under a single root: `.lorgen/`.
ADR scanning still includes `docs/adr/` and `docs/decisions/` in
`sources.adr_dirs` so existing repo conventions are not orphaned.

## Why

- **Single root for Lorgen-curated artefacts** — Knowledge, Wiki,
  metrics, logs, ADRs all under `.lorgen/`. The user's source tree
  stays untouched by default.
- **Hook scope simplifies** — `bin/lorgen-pretool-guard` already
  guards `.lorgen/**` (excluding `logs/`/`cache/`). The default ADR
  path is now automatically covered without an extra hardcode for
  `docs/decisions/` (removed).
- **Convention compatibility** — repos that already use `docs/adr/`
  override `outputs.adr_dir`. The hook re-extends scope to that path
  via the config value.

## Trade-off accepted

- Industry-convention readers (code review tools, ADR explorers)
  expect `docs/adr/`. Users who care can set
  `outputs.adr_dir: docs/adr` in `.lorgen/config.yaml`. The override
  is documented in `skills/lorgen/schema.md` and `accumulation.md`.
- The removed `docs/decisions/` hardcode in the pretool hook means
  Lorgen no longer guards that path *unless* the user sets
  `outputs.adr_dir` to it. This is symmetric with how all override
  paths work.

## Mechanics

- `.lorgen/config.yaml` `outputs.adr_dir: .lorgen/adr`,
  `sources.adr_dirs: [.lorgen/adr, docs/adr, docs/decisions]`.
- `bin/lorgen-pretool-guard` default `adr_dir=".lorgen/adr"`.
- `accumulation.md` Stage 5 numbering function takes adr_dir as arg;
  no path baked in.

## Where to read more

- `docs/architecture.md` — "Why `.lorgen/` lives inside the user repo".
- `skills/lorgen/schema.md` — `outputs.adr_dir` reference.
