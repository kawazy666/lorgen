---
trigger: "marketplace, marketplace.json, install, /plugin marketplace add, single repo"
kind: decision
scope: repo
tags: [meta, distribution]
sources:
  - {type: commit, ref: "bd11516"}
  - {type: code, path: ".claude-plugin/marketplace.json"}
  - {type: code, path: ".claude-plugin/plugin.json"}
  - {type: user-record, ref: "/plugin marketplace add → Marketplace file not found エラー"}
---

# Decision — single repo serves as both marketplace and plugin

## What

`kawazy666/lorgen` is **simultaneously a Claude Code marketplace and
the single plugin it lists**. Two manifests coexist under
`.claude-plugin/`:

- `plugin.json` — this plugin's identity (name, version, description).
- `marketplace.json` — marketplace identity, with one `plugins[]`
  entry pointing at `source: "./"` (= the repo root).

End-user install is two steps:

```
/plugin marketplace add kawazy666/lorgen     # registers the marketplace
/plugin install lorgen@lorgen                # installs the plugin from it
```

The `@lorgen` suffix on install refers to the **marketplace name**
(declared in `marketplace.json`), not the GitHub `owner/repo`. This
was the source of the initial install failure — the natural-looking
`/plugin install lorgen@kawazy666/lorgen` is invalid syntax.

## Why this shape (not a separate marketplace repo)

- **Discovery & install flow** — `/plugin marketplace add <repo>`
  expects a `marketplace.json`. Without it, the add succeeds (per
  the v2.x message) but install resolution fails because no plugin
  source is mapped.
- **Single source of truth** — for a one-plugin project, a separate
  marketplace repo would be ceremonial: every plugin update would
  need a paired marketplace bump. Colocating cuts coordination
  overhead.
- **`source: "./"` works for git-hosted marketplaces** — relative
  paths resolve relative to the marketplace root. As long as the
  repo is added via Git (GitHub URL), `./` correctly points at the
  same repo for plugin source resolution.

## Trade-off accepted

- If we ever ship a second plugin from the same author, we either
  (a) add it to this marketplace's `plugins[]` (this repo becomes
  multi-plugin), or (b) split into a dedicated marketplace repo.
  Option (a) is the lower-friction path; we cross the bridge if the
  scope demands.
- The install command's `@lorgen` suffix is not intuitive for users
  who think in `owner/repo` terms. We mitigate via README — the
  install snippet is documented as the canonical 2-step flow.

## Where to read more

- `README.md` § "インストール" — the 2-step canonical flow.
- `.claude-plugin/marketplace.json` — current marketplace manifest.
- `docs/architecture.md` § "Distribution".
