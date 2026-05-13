# AGENTS.md

このリポジトリは **kufuu** — 日本語圏チーム開発向けの AI コーディング skill pack です。

## 構成

- **skill 本体** は `skills/` 配下 (ハーネス agnostic な SoT、multi file)
  - `sadoku` (査読) — code review / PR 説明文
  - `kouchiku` (構築) — 設計判断 / 計画策定 / 計画実行
  - `tansaku` (探索) — バグ調査 / root cause investigation
  - `shiken` (試験) — TDD discipline / PRUNE
- **設計原則と判断ログ** は `docs/DESIGN.md`
- **使い方ガイド** は `docs/workflow.md` / `docs/workflow.html`
- **ハーネス別配置** は `adapters/<harness>/README.md`

## このリポジトリでの作業ルール

1. skill 本体 (`skills/`) はハーネス agnostic に書く。Claude Code / Cursor / Codex 等の固有 API 名は本文に出さない (出す場合は `adapters/<harness>/` に隔離)
2. 設計判断は `docs/DESIGN.md` の判断 N に追加 (時系列順)
3. PR 粒度: 1 issue = 1 PR (kufuu 自身の `sadoku` の停止条件と同じ)
4. ドキュメントは引き算原則 (§3.8) に従う: 選択肢提示 + 推奨度 N/10 + 1 行根拠 / 図優先

## このリポジトリを使う AI agent への指示

- README.md と docs/DESIGN.md を最初に読む
- `bin/wt` は git worktree CLI、kufuu pack 同梱の補助ツール
- skill の起動 trigger は `docs/workflow.md` の「mode 切替」表を参照

## ライセンス

MIT (`LICENSE` 参照)
