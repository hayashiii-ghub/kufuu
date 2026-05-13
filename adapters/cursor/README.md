# Cursor adapter (未対応 — 土台のみ)

**ステータス**: スケルトン、実装は需要が出てから (YAGNI)

Cursor は `.cursor/rules/*.mdc` 形式で project-level rules を読み込む。kufuu の `skills/*/SKILL.md` をこの形式に変換する adapter を将来ここに置く。

## 想定する変換マッピング

| kufuu (skills/) | Cursor (.cursor/rules/) |
|---|---|
| `skills/sadoku/SKILL.md` | `.cursor/rules/sadoku.mdc` (frontmatter で globs / alwaysApply 指定) |
| `skills/sadoku/references/*.md` | `.cursor/rules/sadoku-<ref>.mdc` または rule 本文に inline |
| trigger (発話) | Cursor では明示 trigger 機構が薄い、`description` で吸収 |
| 状態 trigger (diff 検出等) | Cursor 側の自動 hook で検出可能 |
| subagent gate (b) (sadoku 専門家レビュー) | Cursor の Composer / agents 機構にマッピング (要調査) |
| TodoWrite ベース inline 実行 | Cursor の plan 機能 / Tasks (要調査) |

## 既知の差分 (要調査)

- frontmatter スキーマ (`globs`, `alwaysApply`, `description` 等)
- subagent / sub-task 機構の有無 (Composer の挙動)
- mermaid render の有無
- mcp / tool 呼び出しの portability

## 実装に着手するときの手順 (案)

1. `generate.sh` 作成 — `skills/*/SKILL.md` → `.cursor/rules/*.mdc` に変換
2. SKILL.md 内の Claude Code 固有用語 (Task tool, TodoWrite) を抽象化、または cursor 用の置換表を持つ
3. `references/` を inline するか別 rule にするか決定
4. `agents/` (sadoku の専門家レビュー) を Cursor Composer に移植可能か検証
5. 動作確認 (kufuu repo を Cursor で開いて trigger を試す)

## 着手の判断条件

実利用者から「Cursor で kufuu を使いたい」要望が出た時点で着手。それまで土台のまま。

参考リンク:
- Cursor docs: <https://docs.cursor.com/context/rules-for-ai>
