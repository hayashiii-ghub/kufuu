# wt — Worktree Manager Shell Tool

**ステータス**: v0.1.0 released (`bin/wt`)
**作成日**: 2026-05-11
**実装日**: 2026-05-12

---

## 1. 目的

Claude Code (および他 agent) で git worktree を並列管理するための CLI ツール。
skill 側からは worktree 作成・削除責任を切り離し、shell tool に集約。

---

## 2. 設計原則

- 真実の出処は `.git/worktrees/`、独自 state を持たない
- 各 skill は Step 0 検出 (worktree内 / normal repo) のみ実装、shell tool が不在でも動く
- 環境 agnostic (bash / zsh 互換)
- メタデータが必要なら `.wt/meta.json` (optional)

---

## 3. コマンド API

```bash
wt new <task-name> [--from <base-branch>] [--launch <agent>]
  # 新規 worktree 作成 + branch 切る + path 出力
  # --from 省略時は main / master を自動検出
  # --launch claude で Claude を当該 worktree で起動

wt ls
  # 一覧: path / branch / last commit / 未push数 / test status / idle時間

wt status <task-name>
  # 個別詳細: branch state / 未commit差分 / 直近 test 結果 / 未push commits

wt rm <task-name> [--force]
  # 削除前ガード: test pass確認 + 未push commit警告
  # --force でガード skip

wt cleanup [--dry-run]
  # merged branch の worktree 自動検出・削除
  # --dry-run で削除候補のみ表示

wt enter <task-name>
  # cd 用: eval "$(wt enter feat-A)"
  # または `--launch claude` で起動まで一気に

wt sync [<task-name>]
  # 指定 (or 全) worktree で base branch 最新化
```

---

## 4. 活用ケース 8つ

### Case 1: 別機能の並列実装
2人 (or 2 Claude) が同時に別機能を実装。各 worktree で独立進行。
```bash
wt new feat-user-profile --from main --launch claude
# 別terminal
wt new feat-billing --from main --launch claude
```

### Case 2: 長時間 test 中の別 task
e2e test 走行中に別 worktree で hotfix を進める。
```bash
wt status main         # main worktree のtest進行確認
wt new hotfix-auth     # 別worktreeで hotfix開始
```

### Case 3: 危険変更の実験
ライブラリアップグレード等を独立 worktree で試験。
```bash
wt new experiment-react19
# 試験 → ダメなら
wt rm experiment-react19 --force
```

### Case 4: PRレビュー対応
出した PR の修正依頼を、現作業を中断せず別 worktree で対応。
```bash
wt ls
wt enter feat-user-profile --launch claude
# 修正 → push → 元worktreeへ
```

### Case 5: 並列 debugging (tansaku 連携)
3+ の独立した test failure を別々の worktree で hypothesis 検証。
ただし `/tansaku` 内で「3+独立失敗 + 共有state無し」確認必須。
```bash
wt new debug-auth-race
wt new debug-batch-completion
wt new debug-abort-flow
```

### Case 6: チーム内状況共有
`wt ls` で「誰が何をやってるか」可視化、stale worktree 検出。
```
feat-user-profile  abe@hayashi   2h idle    tests: pass
feat-billing       claude        active    tests: 2 failing
hotfix-auth        miyamoto      30m idle  tests: pass, 3 unpushed
```

### Case 7: PR merge 後の後始末
`wt cleanup` で merged worktree 自動削除、orphan 累積防止。
```bash
wt cleanup --dry-run
# Would remove: feat-user-profile (merged into main 2d ago)
wt cleanup
```

### Case 8: skill 連携 (next-task offer)
`/sadoku` Sign-off 末尾に `wt new <task-name>` 提案、ただし skill が wt を直接呼ばない。
```
[/sadoku Sign-off末尾]
next-task offer: `wt new <task-name>` で次のworktreeを切れます
```

### Case 9: shiken の PRUNE & "test the test" 検証
TDD experiment を隔離 worktree で書き、PRUNE phase で最小化、Iron Rule 検証で test 機能性を確認。
```bash
wt new shiken-payment-validation
# RED → GREEN → REFACTOR → PRUNE で必要 test に絞る
# Iron Rule: 各 test が実装破壊で fail することを目視
git stash
npm test src/payment.test.ts    # → fail を確認
git stash pop
npm test src/payment.test.ts    # → pass を再確認
# 失敗しなかった test は削除して再 commit
```
worktree 内なので、実装ロックな test を書いても `wt rm` で全捨て可能。本流 branch には PRUNE 通過後の test だけ commit される。

---

## 5. 実装言語

**bash** で MVP 実装 (`bin/wt`、約 390 行)。依存は git のみ。機能拡張で複雑度が上がったら go へ移行検討。

---

## 6. ファイル配置

```
~/.local/bin/wt           # 実行ファイル (PATH に配置)
~/.config/wt/             # config (optional)
.wt/                      # repo 内メタデータ (optional, gitignore)
```

---

## 7. skill との連携契約

**skill 側 (Step 0 検出 snippet)**:
```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
[ "$GIT_DIR" != "$GIT_COMMON" ] && echo "(worktree内: $(git branch --show-current))"
```

**shell tool 側 (skill から呼ばれない)**:
- ユーザ or CI が直接コマンド実行
- skill は Sign-off で「次タスク用に `wt new <name>` を打てます」と案内するのみ
- skill が `wt` を spawn することは禁止 (skill哲学維持)

---

## 8. v0.1.0 で確定した設計

- **worktree path**: `<repo>/.worktrees/<task>/` (repo 内、`.gitignore` 推奨)
- **base branch 検出**: `origin/HEAD` → `main` → `master` の順で fallback
- **`--launch <agent>`**: bash 内ロジック (`(cd $path && exec $agent)`)、別 hook script は導入せず
- **meta.json**: 不採用 (`.git/worktrees/` が SoT)
- **削除ガード**: 未 commit (`git status --porcelain`) と未 push (`git log @{u}..HEAD`) を警告、`--force` で skip
- **test pass ガード**: v0.1 では不採用 (環境依存大)、必要なら呼び出し側で wrapper

## 9. 残課題 (v0.2 以降)

- 並列実行の単位 (機能 / PR / task) のチーム規約化
- test pass ガード (`wt rm` 時、`package.json` 等から test command を読む)
- `wt ls` の idle 時間表示 (mtime / 直近 commit から推定)
- `wt sync` の rebase 失敗時の自動 abort
- bash → go 移行 (機能増えた場合)

---

## 10. 実装着手判断

並列実行頻度が週数回以上に達した場面で MVP 着手済 (2026-05-12)。v0.1.0 で 7 subcommand を bash 実装、smoke test 通過。今後の改良は実運用で観測された pain point から逆算する。
