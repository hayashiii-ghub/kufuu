# Codex adapter (未対応 — 土台のみ)

**ステータス**: スケルトン、実装は需要が出てから (YAGNI)

Codex (OpenAI Codex CLI / 後継) は `~/.codex/` または repo 内の `AGENTS.md` を読み込む。kufuu の skill pack をこの方式に橋渡しする adapter を将来ここに置く。

## 想定する変換マッピング

| kufuu (skills/) | Codex |
|---|---|
| `skills/*/SKILL.md` | `AGENTS.md` に概要 + skill 個別は inline / 参照 |
| trigger (発話) | Codex では明示 trigger 機構薄い、prompt の中で skill 名を呼ぶ運用 |
| subagent gate (b) | Codex の sub-task 機構 (要調査) |
| TodoWrite | Codex の plan 機能 (要調査) |
| 評価=環境変化 (検証ログ verbatim) | shell 出力を引用する運用、概ね portable |

## 既知の差分 (要調査)

- skill 切り分けの mechanism (Codex は AGENTS.md 1 ファイル前提か、複数 file 対応か)
- subagent / sub-task の有無
- tool 呼び出しの粒度 (shell / file 操作 etc.)
- mermaid / 図表示の対応

## 実装に着手するときの手順 (案)

1. `AGENTS.md` (root の補助 instruction) で kufuu 全体を概観
2. 各 skill の trigger / mode を Codex prompt 形式で再表現するか、AGENTS.md に列挙
3. `references/` の参照方式 (Codex から file を直接読ませるか、prompt に inline するか) 決定
4. 動作確認 (kufuu repo を Codex CLI で開いて trigger を試す)

## 着手の判断条件

実利用者から「Codex で kufuu を使いたい」要望が出た時点で着手。それまで土台のまま。

参考リンク:
- Codex CLI: <https://github.com/openai/codex>
- AGENTS.md 標準: 本 repo の `AGENTS.md` を Codex は標準で読む可能性が高い
