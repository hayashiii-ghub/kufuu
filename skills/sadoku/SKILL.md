---
name: sadoku
description: "PR code review and PR description authoring. Use when asked レビューして, コードレビュー, PR文書いて, or PR description; also when reviewing a git diff before opening a PR."
license: MIT
when_to_use: "PR確認, レビュー, code review, PR description"
metadata:
  version: "3.0.0"
---

# sadoku (査読)

```
🌲 Using /sadoku for [purpose taken from trigger context].
```

「diff を見る・書く」に純化した skill。通常レビューと PR 説明文の 2 モード。実装行為 (計画実行) は `kouchiku`、TDD は `shiken`、バグ調査は `tansaku` に分離。reviewer コメントへの返信文ドラフトや個別対応は skill mode 化せず通常会話で対応する (分類・咀嚼工程は人間判断のままにする)。

## Step 0: worktree 検出

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
[ "$GIT_DIR" != "$GIT_COMMON" ] && echo "(worktree内: $(git branch --show-current))"
```

検出結果は完了記録の `worktree` 行に出力。worktree の作成・削除は `wt` (別 shell tool) の責務であり、この skill では行わない。

## モード切替

| モード | 発話トリガー | 状態トリガー |
|---|---|---|
| 通常レビュー | `レビューして` / `コードレビュー` | diff 検出 |
| PR 説明文 | `PR文書いて` / `PR description` | PR open 直前 |

状態トリガーは誤発火回避のため、検出後に確認 prompt を 1 行挟む (`diff を検出しました。レビューしますか?`)。複数モードが成立しうる場合は発話トリガーを優先。

reviewer コメントへの返信文ドラフトは skill mode 化しない。ユーザが必要に応じて agent に直接「返信書いて」「押し戻したい」と頼む運用にする (= 通常会話)。実装が必要な指摘はユーザが `kouchiku` に振る (`設計どうする` / `計画実行`)。

## Handoff Intake

`kouchiku` / `tansaku` / `shiken` からの handoff block がある場合は、diff だけでなく前段の判断と検証ログも review evidence として読む。足りない場合は推測で補完せず、通常レビューの停止条件として扱う。

期待する入力:

```
handoff: sadoku
reason: 実装完了、PR 前のレビュー要求
change intent: [何を解決したか]
files changed:
  - [path]
verification:
  - [command] -> pass / fail
tdd:
  - RED: [...]
  - GREEN: [...]
  - PRUNE: [...]
root cause:
  - [tansaku があれば]
scope notes:
  - [やらなかったこと]
review focus:
  - [特に見てほしい観点]
```

## 通常レビューモード

深さ判定 → diff 読解 → 停止条件チェック → 専門家レビュー → 完了記録。

**深さ判定**

| 深さ | 条件 | 動作 |
|---|---|---|
| Quick | 50 行以内、テスト変更のみ等 | 停止条件チェックのみ |
| Standard | 50〜500 行 | 停止条件 + 専門家レビュー (security / architecture) |
| Deep | 500 行超、または security 接触 | Standard に加えて adversarial レビュー |

**専門家レビューの起動 — gate (b)**

Standard 以上で security / architecture 観点が必要な場合のみ subagent 起動。条件と persona は `references/persona-catalog.md`。subagent 成果物は必ず main 側で git diff / file 読み / test 再実行で裏取り。

## PR 説明文モード

template (5 セクション固定)、書き方の手順、4 チェック、PII scan、粒度ルールはすべて `references/pr-template.md` に集約してある。このモードに入ったら **まず `references/pr-template.md` を読み込み**、そこの手順に従って章ごとに提案する。

SKILL.md 側ではモード起動の判定と完了記録への evidence 引用のみ担う。停止条件や必須情報の欠落がなければ、まずレビュー可能な初稿を 1 回で出す。

## 停止条件

以下のいずれかに該当したら**作業を止めてユーザに確認**:

- 破壊的な自動実行 (例: `rm -rf`, `git reset --hard`, force push 等) を明示確認なしで走らせていないか
- diff 内の未知の識別子 (識別子が grep でヒットしないなら、勝手に補完せず質問)
- 依存追加の妥当性が不明 (lockfile 変更があれば理由を確認)
- **PR 説明文 / commit / release notes に PII / Secrets が混入している** (email, token, 個人名等)
- **テスト最小性違反** (いずれか):
  - mock の存在・呼び出し回数を assert している test
  - production class に test-only method が追加されている
  - 部分 mock (実 API の一部 field だけ) で structural assumption が隠れている
  - snapshot test の濫用 (small diff で全更新)
  - `.skip` / `xfail` が理由コメントなしで残っている
- **PR 粒度違反**: diff が複数 issue にまたがっている (= 1 issue = 1 PR ルール違反)
- **未確認の外部事実引用**: 「最新の X バージョン」「Y 標準」のような外部事実が裏取りなしで PR 説明文 / コメントに混入している (§3.10 ファクトチェック原則、URL 引用必須)

## 完了記録

機械検証可能項目は検証ログ (command + 出力末尾) を**そのまま引用**する。**要約・自己申告は禁止**。

```
worktree:         in-worktree / normal-repo
files changed:    N (+X -Y)
scope:            on target / drift: [何が漏れているか]
停止条件:         N found / N fixed
                    検証ログ: [scan command + 出力末尾 / 0件は "0 matches"]
tests:            N added, M essential
                    検証ログ: [test command 最終 summary 行]
verification:     [command] -> pass / fail
                    検証ログ: [出力末尾 3-5 行、失敗時は full error]
PII scan:         clean / found: [...]
                    検証ログ: [grep command + 出力 / 0件は "0 matches"]
文章:             伝わる / 伝わらない (伝わらない場合: 何が伝わらないか)    ※ 自己申告
```

## references/

- `pr-template.md` — PR 説明文モードの本体 (5 セクション template / 書き方手順 / 4 チェック / PII scan / 粒度ルール)
- `project-context.md` — diff 読解時の文脈抽出方針
- `persona-catalog.md` — 専門家レビュー (security / architecture / adversarial) の persona 起動条件
- `agents/reviewer-security.md` / `agents/reviewer-architecture.md` — 専門家レビュー subagent prompt
