# Mimir — リポジトリ知識キュレーター

> ソースコードだけでは語れない**「なぜそうなっているか」**(設計判断、
> 経緯、トレードオフ、教訓) を捕捉・蓄積する。すべてをリポジトリ内の
> Markdown として保存するので、人間と AI エージェントの双方が同じ知識
> ベースを読み、育てていける。

名前は北欧神話の **Mimir** (ミミル) ── オーディンが知恵を求めて頭部を
保管・相談したという賢者の名から。本プラグインがリポジトリに対して
果たす役割と同じ。

Mimir は **Claude Code プラグイン**として配布される。一度インストール
すれば、「なぜ」を捕捉・取得したい全リポジトリで使える。

---

## アーキテクチャ

### このリポジトリ (`mimir`) ── プラグインソース + 自身のデータ

```
mimir/
├── .claude-plugin/
│   └── plugin.json                          # プラグインマニフェスト (name, version, ...)
├── agents/
│   └── mimir.md                             # subagent 定義
├── skills/
│   └── mimir/                               # 操作マニュアル (multi-file skill)
│       ├── SKILL.md
│       ├── schema.md
│       ├── retrieval.md
│       ├── accumulation.md
│       ├── sources.md
│       ├── onboard.md
│       ├── logging.md
│       └── metrics.md                       # logs → metrics の compile contract
├── hooks/
│   └── hooks.json                           # PreToolUse: Write|Edit redaction guard
├── bin/
│   ├── mimir-pretool-guard                  # hooks/hooks.json から呼ばれる
│   ├── mimir-redact-check                   # secret パターンスキャナ (exit 1 = ブロック)
│   ├── mimir-redact                         # redact ラッパー (stdout)
│   └── mimir-compile-metrics                # jq のみで動く決定論的 compile
├── .mimir/                                  # データストア (このリポ自身。dogfood)
│   ├── config.yaml                          (tracked)
│   ├── .gitignore                           (tracked) cache/ + state.json + logs/ を除外
│   ├── knowledge/                           (tracked)
│   │   ├── conventions/
│   │   └── decisions/
│   ├── wiki/                                (tracked)
│   │   ├── overview.md
│   │   └── architecture.md
│   ├── adr/                                 (tracked) outputs.write_adr=true 時にのみ生成
│   │   └── 0001-<slug>.md
│   ├── metrics/                             (tracked) per-session 集計 JSON
│   │   └── 20260429-102345-a3f9c2.json
│   ├── logs/                                (gitignored) raw per-session JSONL
│   ├── cache/                               (gitignored)
│   └── state.json                           (gitignored)
├── docs/
│   └── architecture.md                      # 長文アーキ概要 (living doc)
└── README.md
```

### あなたのアプリリポジトリ (`/plugin install mimir` 後)

```
your-app-repo/
├── .claude/
│   └── plugins/
│       └── mimir/                           # インストール済みプラグイン (project scope)
│           ├── .claude-plugin/plugin.json
│           ├── agents/mimir.md
│           ├── skills/mimir/...
│           ├── hooks/hooks.json
│           └── bin/mimir-*                  # redact-check, compile-metrics ほか
├── .mimir/                                  # Mimir のデータストア (初回利用時に生成)
│   ├── config.yaml                          (tracked) チーム共有設定
│   ├── .gitignore                           (tracked) cache/ + state.json + logs/ を除外
│   ├── knowledge/                           (tracked) Knowledge アイテム
│   │   ├── conventions/
│   │   ├── decisions/
│   │   ├── runbooks/
│   │   ├── facts/
│   │   └── lessons/
│   ├── wiki/                                (tracked) Wiki ページ
│   ├── adr/                                 (tracked) outputs.write_adr=true 時に生成
│   │   ├── 0001-decimal-money.md            #   Mimir が生成した ADR
│   │   └── 0002-aurora-migration.md
│   ├── metrics/                             (tracked) per-session compile 済み集計
│   │   ├── 20260429-102345-a3f9c2.json     #   path / count / timestamp のみ
│   │   ├── 20260429-110512-c8f2d4.json     #   集約による privacy 担保
│   │   └── 20260430-091205-b1e3a7.json
│   ├── logs/                                (gitignored) raw per-session JSONL
│   │   └── 20260429-102345-a3f9c2.jsonl    #   セッション末で metrics/ に compile
│   ├── cache/                               (gitignored) PR/issue/LLM の生キャッシュ
│   └── state.json                           (gitignored) onboard カーソル, indexing state
└── (アプリケーションコード)
```

3 つの構成要素:
- **プラグインソース** (`agents/`, `skills/mimir/`, `hooks/`, `bin/`,
  `.claude-plugin/plugin.json`) はこのリポジトリにのみ存在する。
  `/plugin install` 時に `.claude/plugins/mimir/` (プロジェクト) または
  `~/.claude/plugins/mimir/` (ユーザー) へコピーされる。
- **データストア** (`.mimir/`) は各ユーザーリポジトリで初回利用時に
  作成される。Knowledge / Wiki / Metrics は tracked、raw logs は
  gitignored。
- **ADR ミラー** (`.mimir/adr/`) は `outputs.write_adr` が true、または
  per-record で opt-in したときにのみ書き出される。デフォルトはオフ。
  リポジトリ慣習に合わせて出力先を `outputs.adr_dir` で `docs/adr/`
  などへ上書き可能。

**Logs と metrics**: raw per-session ログは `.mimir/logs/` 配下
(gitignored / マシンごと ── query 文字列、ソース本文、LLM 要約など
secret を含み得るため)。各 invocation の終了時に Mimir はログを
`.mimir/metrics/<session>.json` (tracked / チーム共有) に compile する
── path / count / timestamp のみで、集約による privacy 担保。改善
ループ (stale 検出、hot-item 昇格、trigger health) は metrics のみを
読み、raw logs は決して読まない。詳細は
[`skills/mimir/metrics.md`](skills/mimir/metrics.md)。

**機械的 secret ガード**: PreToolUse hook
(`hooks/hooks.json` + `bin/mimir-pretool-guard`) が `.mimir/**`
(`.mimir/logs/**` と `.mimir/cache/**` を除く) と設定済み ADR
ディレクトリへの全 Write/Edit を既知の secret パターンでスキャン
し、マッチした場合は hook 層で書き込みをブロックする。skill 命令
側も多重防御として `bin/mimir-redact` でコンテンツをパイプする。
パターン一覧は [`bin/mimir-redact-check`](bin/mimir-redact-check)。

---

## Mimir ができること

| あなたが頼むと | Mimir はこうする |
|---|---|
| `なぜ Decimal を money に使ってる?` | まず `.mimir/knowledge/` を検索。なければ `git log` / PR 説明 / ADR / コードコメントを読み、引用付きで回答を合成。新しい知見を `.mimir/knowledge/conventions/decimal-money.md` に保存し、次の質問は即座にヒットする。 |
| `auth モジュールの設計判断履歴を教えて` | `.mimir/wiki/modules/auth.md` と関連 Knowledge を読む。欠落があれば git/PR/issue を辿り、Wiki を更新。 |
| `この PR の判断を残しておいて` | PR 説明とレビューコメントを蒸留し、引用付きで `decision` 種別の Knowledge を新規作成。 |
| `この repo を onboard して` | 一括スキャン: README / CLAUDE.md / AGENTS.md / ADR / 直近マージ済 PR (デフォルト 30 日 / 50 件) / コードコメント → Wiki overview、モジュール別ページ、シード Knowledge を生成。 |
| `この決定を ADR にして` / `@mimir record --adr "..."` | Knowledge を `.mimir/adr/NNNN-<slug>.md` (または `outputs.adr_dir` 指定先) に MADR-lite フォーマットでミラー。連番、不変、Knowledge と双方向リンク。デフォルトはオフ ── per-record か `outputs.write_adr: true` で opt-in。 |
| `過去 1 年遡って onboard して` | 同じ処理を月単位のチャンクで。state-file による再開対応。 |

出力は引用付きの Markdown を stdout へ
(`(PR #42)`, `(commit abc1234)`, `(.mimir/adr/0003-decimal.md)` のような
インライン引用)。ファイルは `.mimir/` に書き込まれる。Mimir は
**絶対に** commit / push しない ── 人間が `git diff .mimir/` をレビュー
してコミットする。

---

## なぜ Mimir なのか (素の Claude Code との比較)

| Mimir なし | Mimir あり |
|---|---|
| 新セッションのたびに README を読み直し / 慣習を grep し直す | 一度蓄積した Knowledge を以後即座に取得 |
| 「なぜここがこうなっているか」は現在のコードからしか答えられない | git 履歴 / PR の根拠 / ADR / 過去の教訓から蒸留して回答 |
| Knowledge は会話と共に消える | Knowledge は `.mimir/` で生き続け、セッション・コントリビュータ・バージョン管理を超えてチーム共有される |
| 同じ質問は毎回同じトークンを消費 | 反復質問は `.mimir/knowledge/` のキャッシュにヒットし、ほぼ無料 |

Mimir は Devin の **Knowledge + DeepWiki** 機能に着想を得つつ、
- リポジトリ内に常駐
- 追加コストなし (Claude Code セッション内で動く)
- 完全に透明 (データモデルはすべて読める / 編集できる / 書き直せる
  プレーンな Markdown)

---

## インストール

```bash
# プロジェクトローカル (プラグインは .claude/plugins/mimir/ に配置)
/plugin install mimir@kawazy666/mimir

# ユーザーグローバル (~/.claude/plugins/mimir/ に配置)
/plugin install mimir@kawazy666/mimir --user
```

バージョン pin は `@v0.1.0` のように付与。最新追従はバージョンを
省略 (現在のコミット SHA を使用)。fork / 自己公開する場合は
`kawazy666/mimir` を自分の GitHub `owner/repo` に置換。

インストール後、Claude Code を再起動 (または `/reload-plugins`) して
新しい agent と skill を読み込ませる。次に任意のプロジェクトで:

```
@mimir 初期化して
```

Mimir が `.mimir/config.yaml`、`.mimir/.gitignore`、空の `knowledge/`
/ `wiki/` ディレクトリを作成。準備ができたらコミット。

---

## 使い方

Claude Code 上で subagent として呼び出す:

```
@mimir なぜ Decimal を money に使ってるの?

@mimir この repo を onboard して

@mimir 過去 1 年の決定を遡って蓄積して、月単位で

@mimir record "release process: tags push を main の merge 後に実行する"

@mimir この決定を ADR にして
```

意図を素直に書くだけでもよい ── intent が `description` に合致すれば
Claude Code が Mimir にルーティングする。

---

## アンインストール

```bash
/plugin remove mimir            # プラグイン (skill + agent) を削除
rm -rf .mimir                   # データストアを削除 (任意)
```

`/plugin remove` はプラグイン本体のみを削除する。`.mimir/knowledge/`
と `.mimir/wiki/` は `rm -rf .mimir` を別途実行しない限り残る。

---

## ファイルスキーマ (短縮版)

### Knowledge アイテム — `.mimir/knowledge/<category>/<slug>.md`

```markdown
---
trigger: "Decimal, money, monetary calculation"
kind: convention            # convention | decision | runbook | fact | lesson
scope: repo
tags: [money, billing]
sources:
  - {type: pr, ref: "#42", url: "..."}
  - {type: commit, ref: "abc1234"}
  - {type: adr, path: ".mimir/adr/0003-decimal.md"}
created: 2026-04-29
updated: 2026-04-29
---

[Markdown 本文 ── 数文〜数段落]
```

### Wiki ページ — `.mimir/wiki/<area>.md`

```markdown
---
title: Auth module
parent: architecture
summary: Session issuance, storage, refresh, and revocation.
updated: 2026-04-29
---

[Markdown 本文 ── モジュール概要、Knowledge アイテムへのリンク]
```

完全リファレンス: [`skills/mimir/schema.md`](skills/mimir/schema.md)

---

## 設定

`.mimir/config.yaml` (tracked / チーム共有):

```yaml
model: claude-opus-4-7
effort: high

sources:
  git: true
  code_comments: true
  github:
    enabled: true
    pr_lookback_days: 30
    pr_lookback_count: 50
  adr_dirs:
    - .mimir/adr
    - docs/adr
    - docs/decisions

outputs:
  write_adr: false
  adr_dir: .mimir/adr

wiki:
  repo_notes: []
  pages: []
```

完全リファレンス: [`skills/mimir/schema.md`](skills/mimir/schema.md)

---

## Mimir がやらないこと

- **絶対に commit / push / PR 作成をしない** ── あなたが
  `git diff .mimir/` をレビューした上で行う。
- **デフォルトでは `.mimir/` の外にコードを書かない**。ADR 生成を
  opt-in した上で `outputs.adr_dir` を `.mimir/` の外 (例: `docs/adr/`)
  へ上書きした場合に限り、その指定先にも書き込む ── これだけが唯一
  の外向き書き出しで、ユーザー設定によるもの。
- **歴史を捏造しない** ── 全主張に実在のソース引用を付与。確信を
  もって答えられない場合は素直にそう言う。
- **Secret を保存しない** ── API キー / トークンは保存前に redact。

---

## アーキテクチャ詳細

設計判断・検討した代替案・現在の形を選んだ理由 (プラグイン配布、
単一 `.mimir/` データストア、ベクター検索ではなく ripgrep + LLM の
retrieval、"read = accumulate" 等) は
[`docs/architecture.md`](docs/architecture.md) を参照。

---

## ステータス

| コンポーネント | 状態 |
|---|---|
| Subagent (`agents/mimir.md`) | **MVP 完了** |
| Skill bundle (`skills/mimir/*`) | **MVP 完了** ── SKILL.md + schema/retrieval/accumulation/sources/onboard/logging/metrics |
| `.mimir/` データストアスキーマ | **MVP 完了** |
| プラグインマニフェスト (`.claude-plugin/plugin.json`) | **MVP 完了** |
| `mimir onboard` (recent + 1y backfill) | **MVP** (skill に記述、agent が実行) |
| ADR 生成 (`outputs.write_adr` + per-record opt-in) | **MVP 完了** |
| 監査ログスキーマ (`.mimir/logs/<session>.jsonl` / gitignored) | **MVP 完了** |
| Compile 済み metrics (`.mimir/metrics/<session>.json` / tracked) + `bin/mimir-compile-metrics` + smoke test | **MVP 完了** |
| 機械的 secret ガード (PreToolUse hook + `bin/mimir-redact{,-check}`) | **MVP 完了** |
| `mimir consolidate` (Knowledge dedup pass) | **Roadmap** |
| Stale 検出 + hot-item 昇格 (metrics 消費) | **Roadmap** |
| Notion ソースコネクタ | **Roadmap** |
| 定期自動 reindex | **Roadmap** |
| Claude Code 以外から呼べる Standalone CLI | **Roadmap** |

---

## ライセンス

MIT.
