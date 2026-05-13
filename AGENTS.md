# AGENTS.md

このリポジトリは **kufuu** — 日本語圏チーム開発向けの AI コーディング skill pack です。

## 構成

- **skill 本体** は `skills/` 配下 (ハーネス agnostic な SoT、multi file)
  - `sadoku` (査読) — code review / PR 説明文
  - `kouchiku` (構築) — 設計判断 / 計画策定 / 計画実行
  - `tansaku` (探索) — バグ調査 / root cause investigation
  - `shiken` (試験) — TDD discipline / PRUNE
- **設計原則と判断ログ** は `docs/DESIGN.md`
- **使い方ガイド** は `docs/workflow.md` (mermaid 図入り)
- **ハーネス別配置** は `adapters/<harness>/README.md`

## このリポジトリでの作業ルール

1. skill 本体 (`skills/`) はハーネス agnostic に書く。Claude Code / Cursor / Codex 等の固有 API 名は本文に出さない (出す場合は `adapters/<harness>/` に隔離)
2. 設計判断は `docs/DESIGN.md` の判断 N に追加 (時系列順)
3. PR 粒度: 1 issue = 1 PR (kufuu 自身の `sadoku` の停止条件と同じ)
4. ドキュメントは引き算原則 (§3.8) に従う: 選択肢提示 + 推奨度 N/10 + 1 行根拠 / 図優先

## このリポジトリを使う AI agent への指示

- README.md と docs/DESIGN.md を最初に読む
- `bin/wt` は git worktree CLI、kufuu pack 同梱の補助ツール
- skill の発動 trigger は下記「skill の発動 trigger 一覧」を参照、詳細仕様は各 SKILL.md

## skill の発動 trigger 一覧

ユーザの発話に下記キーワードが含まれたら、対応する skill / mode を起動する。詳細 (停止条件 / 完了記録 / 状態 trigger / 専門家レビューなど) は各 `skills/<name>/SKILL.md` を参照。

| skill | mode | 発話 trigger |
|---|---|---|
| `sadoku` (査読) | 通常レビュー | `レビューして` / `コードレビュー` |
| `sadoku` | PR 説明文 | `PR文書いて` / `PR description` |
| `kouchiku` (構築) | 軽量検討 | `どうやって直す` / `やり方どっち` |
| `kouchiku` | 通常検討 | `設計どうする` / `方針決めたい` / `アーキテクチャ判断` |
| `kouchiku` | 評価 | `やる価値ある` / `採用すべきか` / `kill か keep か` / `やめる?` |
| `kouchiku` | 計画実行 | `計画実行` / `進めて` / `着手` / `実装開始` |
| `tansaku` (探索) | 通常追跡 | `エラー` / `動かない` / `落ちる` / `クラッシュ` |
| `tansaku` | 二分探索 | `前は動いてた` / `アップデート後から` / `更新後動かない` |
| `tansaku` | 再発追跡 | `同じ問題が再発` |
| `shiken` (試験) | 起動 | `TDDで` / `テストから書いて` |

**注**: reviewer コメントへの返信文ドラフトは skill mode 化していない (通常会話で `返信書いて` / `押し戻したい` のように直接依頼)。実装が必要な指摘は `kouchiku` の計画実行モードに振る。

## ライセンス

MIT (`LICENSE` 参照)
