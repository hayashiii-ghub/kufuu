# kufuu 設計書

**バージョン**: 3.0
**確定日**: 2026-05-12
**ベース**: tw93/Waza skill 群
**対象**: Claude Code (`~/.claude/skills/`)

実装本体は `../skills/*/SKILL.md` を SoT とする。本書は「設計の意思決定ログ」と「全 skill 共通の原則」を扱う。

---

## 1. 目的とスコープ

日本語圏チーム開発において、Claude Code が「コードを書く」作業を一貫した discipline で実行するための skill 群を定義する。waza skill 群を起点とし、SP (anthropic/superpowers) から選択的に取り込み、日本語圏 team-dev の現実に最適化する。

### 対象作業
- 機能実装 (TDD discipline 込み)
- バグ調査 / root cause investigation
- code review / PR 説明文 / レビュー対応
- 設計判断 / 評価 / 計画策定 / 計画実行

### 対象外
- UI design (既存 design system 厳守前提、必要時は追加 skill)
- 散文 (blog, SNS, white paper) の推敲
- 多 source research / domain learning (頻度低、`kenkyu` skill は drop)
- Claude Code config audit (個人運用領域)

---

## 2. core 4 skill

| skill | 漢字 | 動詞 | 担当 | waza 起源 | version |
|---|---|---|---|---|---|
| `sadoku` | 査読 | 見る・書く | code review / PR 説明文 | check + write 部分吸収 | 3.0.0 |
| `kouchiku` | 構築 | 考える・作る | 設計判断 / 評価 / 計画策定 / 計画実行 | think | 2.0.0 |
| `tansaku` | 探索 | 追う | バグ調査 / root cause investigation | hunt | 2.0.0 |
| `shiken` | 試験 | 試す | TDD discipline / PRUNE | (新規) | 3.0.0 |

**役割境界の原則**: 動詞単位で 4 分割。`sadoku` は実装行為を含まない、`kouchiku` が設計から実装まで一気通貫、`tansaku` はバグ専用、`shiken` は TDD discipline 専用。

仕様の詳述は各 `../skills/*/SKILL.md` 参照。本書では設計原則と決定ログのみ扱う。

### 抜いた skill

| skill | 抜いた理由 |
|---|---|
| `design` | 既存 design system 厳守前提、新規 UI 設計が頻発する場合のみ後付け追加 |
| `write` (`suikou`) | 散文 specialized、PR 文関連だけ sadoku に吸収 |
| `note-thumbnail` | note 記事用、team-dev スコープ外 |
| `kenkyu` (研究) | 6-phase research は team-dev で頻度低い、WebFetch + 会話で代替 |
| `shutoku` (取得) | GitHub + 公開 web 中心なら WebFetch + gh で代替、X / Feishu / WeChat は読まない |
| `kenshin` (健診) | 個人 sysadmin tool 寄り、team-dev pack ではなく個人 skill 置き場に別途配置 |

### 別途

- `wt` — git worktree shell tool (draft、実装後回し、同階層 `WT-SPEC.md` 参照)

---

## 3. 設計原則

### 3.1 waza 哲学の継承

**1 skill 多モード**: 発話トリガーと状態トリガーで mode 分岐。skill 数を抑え、チーム学習コストを下げる。

**references/ ファイル分離**: SKILL.md は controller として軽量に保ち、詳細 rule / persona / chart は `references/*.md` に分離。

**scripts/ で決定論的処理**: LLM 判断に頼らない作業 (URL fetch, env collection, test 実行) は shell script に外出し (今回は採用なし)。

**環境 agnostic**: subagent facility / native tool 検出 + fallback を必須化。Claude Code / Codex / その他で同じ skill が動く。

### 3.2 Controller Owns Information

情報取得目的の subagent は default で使わない。subagent は **judgment 委譲装置**であって execution / 情報取得装置ではない。

裏付け:
- `tansaku` は `Never state from memory. Run grep first.` を Hard Rule 化
- `sadoku` は「diff 内の未知の識別子: 承認前に grep 必須」を停止条件化

### 3.3 Inline Default + Subagent 明示 gate

default は TodoWrite ベースの inline 実行。subagent は次の 3 条件のみ起動可:

| カテゴリ | 条件 | 例 |
|---|---|---|
| gate (a) 重い情報取得 | 5+ ファイル横断 + 複数命名規則探索 / Web 3+ ソース集約 | "Where is Foo used across the codebase?" |
| gate (b) Specialist review | 大規模 diff (500+ 行) の security / architecture 分析 | sadoku の専門家レビュー |
| gate (c) 機械 fan-out | 5+ 独立機械タスク (codemod 等)、spec watertight、視覚判断不要 | mass rename, doc generation |

**禁止事項**: TDD RED-GREEN サイクル / 視覚確認絡み / 1-2 ファイル単位の通常実装 / 計画実行モード本体。これらは inline 強制。

### 3.4 SP からの選択的取り込み

| 取り込み | 適用先 | 効果 |
|---|---|---|
| Announce-at-start (🌲 Using /X ...) | 全 skill 共通 | チームメンバー / ログでの可視性 |
| worktree Step 0 検出 | 全 skill 共通の冒頭判定 | worktree 内 / normal repo 判定、`wt` 連携準備 |
| Hard Rules 冒頭の 1 文ガード | `shiken` / `tansaku` | 圧力下に出る言い訳を 1 行で封じる |

**取り込まなかった SP パターン**: Iron Law 一行宣言 / Red Flags list / Rationalization 表。チーム開発前提なら PR レビューが外部目線で同等の役割を果たし、Hard Rules + 1 文ガードで代替可能。dramatic framing は cargo-cult と判断。

### 3.5 日本語圏 team-dev 最適化

| 要素 | 言語 |
|---|---|
| skill name | 英語短語 (`sadoku`, `tansaku`...) — slash command で打つ前提 |
| frontmatter `description` | 英語 — Skill 一覧 UI 標準 |
| `when_to_use:` keywords | 日英並記 (中文除外) |
| 本文 Mode 名 / 節タイトル / ラベル | 日本語 |
| Hard Rules 宣言文 | 日本語 (圧力下の自己参照テキスト、母語が効く) |
| 固有名詞 (TDD, mock, RED/GREEN/REFACTOR/PRUNE 等) | 英語残し |

### 3.6 評価は「環境変化」で見る

「agent が何を言ったか」ではなく「環境がどう変わったか」を判定軸にする。完了記録の機械検証可能項目は、コマンド出力をそのまま引用する。要約・自己申告は禁止。

**機械検証 evidence の最低要件**

- command 名 + 末尾 3-5 行を引用
- 失敗時は full error
- 0 件結果も明示 (例: `grep -E '...' diff → 0 matches`)

**適用先**

| skill | 機械検証項目 |
|---|---|
| `sadoku` | tests pass / 停止条件 scan / verification command / PII scan |
| `tansaku` | root cause confirm (fix 前後の挙動 diff)、regression test pass |
| `shiken` | RED→GREEN→PRUNE 各段階の test 実行ログ |
| `kouchiku` | (計画実行モードのみ) verification |

例外: 散文評価 (PR 文の伝わりやすさ) は機械判定困難、自己申告で運用 (4 チェック)。

### 3.7 散文 (PR 文等) の扱い

唯一の評価軸: **伝わりやすさ** (読み手が必要情報に最短で辿り着けるか)。

**4 チェック (これだけ)**

1. **結論先出し** — 1 文目で「何が変わったか」が分かる
2. **1 段落 1 主張** — 読み手に並列処理させない
3. **語彙が読み手のもの** — チーム / 外部レビュアーが知ってる言葉か
4. **儀礼削除** — 「まず最初に」「以上、よろしくお願いします」は技術文に不要

**意図的にしないこと**

- AI 臭リストの機械適用 (単語狩りは表面的、model 世代で症状が変わる、固定リストはすぐ古びる)
- 「自然な文章」を抽象的に求めない (判定は読み手任せで指針にならない)

### 3.8 引き算 (認知負荷削減)

skill / agent は読み手 (user / reviewer) の認知負荷を下げる側に立つ。agent 側の出力構造化の手間は厭わない。kufuu の他原則 (PR 粒度・テスト最小化・散文 4 チェック) と同じ「引き算」哲学を全 skill に貫通させる。

**5 原則**

1. **選択肢を提示する**: open question (「どうしますか?」) を投げず、案 + 推奨を出す。ただし**明らかな場合は 1 案でよい** (kentou の「3 案以上は出さない」と整合、迷い時のみ 2-3 案)
2. **推奨度を数値化**: 各案に **N/10 の推奨度 + 1 行根拠** を付ける。数値だけでは検証不能なので根拠必須
3. **簡潔 ≠ 短い**: 結論先出し + 必要な根拠は残す。儀礼や前置きは削るが、判断の根拠は削らない
4. **構造化できる情報は図**: 表 / mermaid / box diagram で構造を見せる。判断のニュアンスは散文で書く (「図 vs 文」の使い分け)
   - **コンテキスト可視化は AI が初稿**: 従来「面倒で書けなかった依存関係や前提」を、AI が初稿、人間が添削する分業で外化する
   - **未完成が通常状態**: 完璧な図を狙わず「質問が出る図」を作る。更新は質問発生時のみで十分 (図の保守地獄を避ける)
   - **形式は mermaid に統一**: テキストで diff が取れる、git 親和性高い、code review の流れに乗る
5. **読み手の負荷を最優先**: agent 側の手間より、user / reviewer の認知負荷を下げる側に立つ

**適用先 (既存原則の強化)**

| 項目 | 既存 | 引き算で強化 |
|---|---|---|
| 全 skill の応答スタイル | 各 SKILL.md | 選択肢提示 + 推奨度数値化 |
| sadoku PR 説明文 | `references/pr-template.md` | 構造化情報は表 / mermaid 優先 |
| kouchiku 通常検討 | 推奨案 + 代替案 | 各案に推奨度 N/10 + 1 行根拠 |
| docs 全般 | DESIGN.md / workflow.md | 表・mermaid・box diagram を散文より先に置く |

**緩和策 (機械適用の形骸化を防ぐ)**

- 数値推奨度の根拠を 1 行必須化 (検証可能性の確保、agent 内部判断のブラックボックス化を防ぐ)
- 「簡潔」=「結論先出し + 必要な根拠は残す」を明文化 (根拠を削る agent にならないように)
- 「明らかなら 1 案、迷い時のみ 2-3 案」(無用な選択肢水増しを防ぐ、kentou paralysis 防止と整合)

**他原則との関係**

- 散文 4 チェック (§3.7) は引き算の言い換え。引き算原則が上位概念
- kentou の「3 案以上は出さない」(判断 14) と同根、選択肢の上限を共有
- 評価=環境変化 (§3.6) と独立: 引き算は「読み手の負荷」、§3.6 は「検証可能性」を扱う

---

## 4. skill 別 要約

仕様の本体は `../skills/*/SKILL.md`。本節は俯瞰用の要約のみ。

### 4.1 `sadoku` (査読) — 見る・書く

通常レビューと PR 説明文の 2 モード。実装行為 (計画実行) は `kouchiku` に移管、TDD は `shiken`、バグ調査は `tansaku` に分離。reviewer コメントへの返信文ドラフトは skill mode 化せず通常会話で対応。

`references/pr-template.md` に PR 説明文モードの本体 (5 セクション template / 8 ステップの書き方 / 4 チェック / PII scan / 1 issue = 1 PR ルール) を集約。`references/persona-catalog.md` で 3 persona (security / architecture / adversarial) の起動条件を定義、agent prompt は `agents/reviewer-{security,architecture}.md`。

### 4.2 `kouchiku` (構築) — 考える・作る

軽量検討 / 通常検討 / 評価 / 計画実行の 4 モード。設計から実装まで一気通貫。

通常検討モードの出力は前提崩し / 攻撃検証 / Plan steps を必ず含む。承認後に「計画実行」「進めて」で同 skill 内の計画実行モードへ移行。計画実行中に TDD 必要層に触れたら `shiken` のサイクルに入る。実装完了後は `sadoku` に PR レビューを渡す。

### 4.3 `tansaku` (探索) — 追う

通常追跡 / 二分探索 / 再発追跡の 3 モード。**hypothesis を 1 文書けるまで code を触らない**。1 hypothesis = 1 instrument の原則で、`references/logging-techniques.md` から 1 パターンだけ仕込む。fix 確定後は前後の挙動 diff をそのまま引用 (要約禁止)。

### 4.4 `shiken` (試験) — 試す

`TDDで` / `テストから書いて` の発話 trigger、または層分け判定で起動。RED → GREEN → REFACTOR → **PRUNE** のサイクル。

層分けトリガー: 純ロジック / API / バグ修正は強制、インタラクションは推奨、純スタイル / アニメ / 文言はスキップ可。PRUNE 後に残った各 test は実装を revert して fail を目視する (失敗しない test は削除対象)。

`references/testing-anti-patterns.md` で 6 種の anti-pattern を定義 (mock の存在 assert 禁止、production class への test-only method 禁止、部分 mock、snapshot 濫用、`.skip` 理由なし放置、同 assertion 重複)。

---

## 5. 全 skill 共通仕様

### 5.1 Announce-at-start

各 skill 冒頭で:
```
🌲 Using /<skill> for [purpose taken from trigger context].
```

### 5.2 Step 0: worktree 検出

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
[ "$GIT_DIR" != "$GIT_COMMON" ] && echo "(worktree内: $(git branch --show-current))"
```

検出結果は完了記録の `worktree` 行に出力。worktree の作成・削除は `wt` shell tool の責務、skill 側は行わない。

### 5.3 subagent ガイドライン

- default は inline TodoWrite
- subagent 起動条件 = 3.3 の (a)(b)(c) のみ
- subagent prompt の必須項目: scope / context / constraints / expected output format
- subagent 成果物の受け取り: git diff / file 読 / test 再実行で**必ず main 側で裏取り**
- 並列 subagent 最大 3

---

## 6. 設計判断ログ

各判断は「結論 + 根拠」の対で記録。後で振り返って「なぜそうしたか」を辿れるようにする。

### 判断 01: なぜ waza ベース? SP ではなく

**結論**: waza は実戦経験を圧縮した「動くツールベルト」、SP は逸脱を防ぐ「規律フレームワーク」。team-dev には waza の出力テンプレ・mode 統合・project context 抽出が効く。SP からは worktree Step 0 検出と Hard Rules 冒頭の 1 文ガードのみ移植、Rationalization 表 / Iron Law / Red Flags は不採用。

**根拠**: waza の 1 skill 多モードはチーム導入の学習コストが低い、Sign-off テンプレは他人が読む前提でスキャナブル、version 管理付きで配布更新しやすい。SP の 1 skill 1 鉄則の規律性は強いが、scope が小さい team-dev タスクではオーバーキル、coordination tax が重い。

### 判断 02: subagent 抑制 (controller owns information)

**結論**: 情報取得目的の subagent は default で使わない。subagent は analysis 分離装置であって execution 装置ではない。

**根拠**: waza 全 skill 中、subagent を使うのは `sadoku` の専門家レビューのみ。「controller が情報を持った状態で persona judgment を委譲」のパターン。決定論的取得 (grep/read) のほうが再現可能、subagent summary は情報損失が不可逆。

### 判断 03: inline default + subagent 明示 gate (3 条件のみ)

**結論**: TodoWrite ベースの inline 実行が default。subagent は gate (a) 重い情報取得 / gate (b) Specialist review / gate (c) 機械 fan-out の 3 条件のみ起動可。

**禁止事項**: TDD RED-GREEN / レビュー咀嚼 / 視覚確認絡み / 1-2 ファイル単位の通常実装 / 計画実行モード本体。これらは inline 強制。

### 判断 04: なぜ「最終実行フェーズで subagent fan-out」を default にしないか

**結論**: SP の subagent-driven-development パターンは team-dev の 80% のタスクで赤字。default にしない、明示 mode opt-in。

**根拠 (体感値)**:

| タスク種別 | 効果 |
|---|---|
| 適合タスク (codemod / mass-rename / mechanical CRUD) | 速度 +20〜30% |
| 不適合タスク (普通の feature 実装) | 速度 -30〜50% (re-review loop + verification 負荷) |
| フロント team-dev のタスク分布 | 後者が約 80% |

**実害**: TDD red-green 希釈 / "Done" lies の頻発 / Style/judgment drift / Coordination tax。

### 判断 05: TDD トリガー層分け (なぜ全コード強制ではないか)

**結論**: SP 流の「全コードに失敗テスト先行」をフロントに機械適用すると brittle なテストが量産される。**レイヤーで振る舞いを変える**。

| レイヤー | 扱い |
|---|---|
| 純ロジック / ビジネスルール / API 層 | 強制 |
| バグ修正 (どのレイヤーでも) | 強制 regression guard |
| インタラクション | 推奨 |
| 純スタイル / アニメ / 文言 / asset | スキップ可 (理由必須) |
| 設定 / build / CI | 強制 (動作確認) |

**スキップガード**: 完了記録に 1 行必須 (`tdd: skip — CSS-only, no logic touched`)、ロジックに 1 行でも触れたら強制復帰、PR 集計で `skip × N / enforced × M`。

### 判断 06: なぜ AI 臭リストを作らないか

**結論**: 散文の評価軸は伝わりやすさのみ。AI 臭の固定リストは作らない、§3.7 の 4 チェックだけ運用。

**根拠**: model 進化で症状が変わる (現行 model は "delve / leverage / It's worth noting" 系をもう落ちにくい) / 単語狩りは perverse incentive (「delve 禁止」→「explore を 50 回」など不自然回避を生む) / リストの maintain コスト >> 効果 / 表面パターン除去 ≠ 伝わりやすさ。

### 判断 07: `suikou` (write) 抜き、sadoku に部分吸収

**結論**: waza/write は散文 specialized で、機能の 8 割は team-dev で使わない。**3 つだけ sadoku に移植**: PR Description Mode、Release Notes Drafting、PII Hard Stop。

**根拠**: 日本語圏 team-dev では中英 bilingual rule 不要、SNS は scope 外、design doc 連貫性は sadoku scope 外、tw93 個人 OSS 固有は移植不可。残るのは PR description / commit message / release notes 関連のみ。

### 判断 08: worktree 管理は skill ではなく shell tool 側に

**結論**: skill には Step 0 検出 (`GIT_DIR != GIT_COMMON`) のみ。worktree 作成・削除・list は `wt` shell tool に集約。

**根拠**: SP/using-git-worktrees の 215 行を skill に持たせるのは過剰。並列実行頻発時、shell 側で list/status を持つほうがチーム可視性高い。waza の環境 agnostic 思想を維持 (shell 未導入環境でも skill は動く)。ただし `wt` 自体の実装は並列頻度測定後に判断、現時点では spec のみ確保。

### 判断 09: 日本語圏最適化の粒度

**結論**: 全部日本語化はせず、要素ごとに使い分け (§3.5 の表)。skill name / description / 固有名詞は英語、本文 / Mode 名 / Hard Rules は日本語。

### 判断 10: SP からどれを取り、どれを捨てるか

**取り込み 3 点**: Announce-at-start / worktree Step 0 検出 / Hard Rules 冒頭の 1 文ガード (shiken/tansaku)。

**不採用 (waza の Hard Rules で代替)**: Iron Law 一行宣言 / Red Flags list / Rationalization 表。チーム開発前提なら `sadoku` の PR レビューが外部目線で同等の役割を果たし、表は形骸化リスクが高い。

**不採用 skill**: subagent-driven-development / executing-plans / brainstorming / writing-skills / finishing-a-development-branch / requesting-code-review。SP の skill は waza の既存 skill に代替がある (`kouchiku` 計画実行が executing-plans 相当、`kouchiku` が brainstorming 相当)、または waza 哲学と矛盾する (subagent-driven-development) ため。

### 判断 11: PR 粒度・PR 文 format の指定取り込み

**結論**: ユーザ指定の **5 セクション固定 template + 1 issue = 1 PR** 粒度ルールを `sadoku` PR 説明文モードに正式取り込み。本体はすべて `references/pr-template.md` に集約、SKILL.md からは呼び出しのみ (フォーマット変更は references の更新だけで完結する設計)。

**5 セクション**: 課題 / DoD / 実装の流れとレビュー順 / 実装中に分かったこと / 検証。

**粒度ルール**: 実装中に発見した別問題は scope に含めず「実装中に分かったこと」に記録 + 別 issue / 別 PR。diff が複数 issue 跨ぎなら停止条件発動。

### 判断 12: test pruning と worktree affordance (shiken に PRUNE phase 追加)

**結論**: SP の TDD は「書く discipline」に寄っているが、**「捨てる / 最小化する discipline」も同等に必要**。shiken の cycle を RED → GREEN → REFACTOR → **PRUNE** に拡張。worktree (wt) は「捨てるコストゼロ」の affordance を提供。

**3 層防御**:

| 層 | 担当 | 役割 |
|---|---|---|
| 予防 | shiken (TDD discipline + 層分けトリガー) | 不要な test を書かせない |
| 削減 | shiken の PRUNE phase + 実装 revert verification | 書いた test を最小化、機能性を実装 revert で検証 |
| 検出 | sadoku の停止条件 (テスト最小性違反) | review 時に anti-pattern を hard stop |

### 判断 13: Phase 1 slim 化: 7 skill → core 4 に集約

**結論**: 「事前に書き溜めない、運用しながら追加する」原則を適用。Phase 1 は core 4 skill (sadoku / kouchiku / tansaku / shiken) のみ実装、他は drop。

**drop した skill**: kenkyu (research workflow 頻度低) / shutoku (WebFetch + gh で代替) / kenshin (個人 sysadmin tool、別配置)。

**drop した mode**: sadoku の Triage / Release Worthiness / Release Notes / Ship、tansaku の Rendering Bug / IME / Unicode。

**採用条件**: Mode は 3 回以上の実発火、references は週 1 以上の参照、agents persona は 5 回以上の実起動、scripts は 3+ シナリオ再利用、Gotcha 行は実事故 1 件 = 1 行。pre-emptive な記述は YAGNI で全削除。

### 判断 14: 役割境界の純化 (v2.0 → v3.0)

**結論**: 旧版 `sadoku` は「review (見る)」「Plan Execution (作る)」「Reviewer Feedback (返す)」を併存し性格が混ざっていた。v2.0 で **Plan Execution を `kouchiku` に移管**、v3.0 で **レビュー咀嚼モードも廃止**。`sadoku` は「見る・書く」に純化 (2 モードのみ)、`kouchiku` は「考える・作る」を一気通貫で担う。

**レビュー咀嚼モード廃止の判断軸**:
- 「咀嚼 = 分類」工程は人間判断の方が速い (ユーザが頭で振り分ける方が、Claude に分類させて再判断するより早い)
- 1 件対応が実運用の大半 (mode を切る価値が低い)
- 返信文ドラフトは通常会話で十分
- 実装が必要な指摘は `kouchiku` に直接

**日英混合の整理**: モード名・節タイトル・ラベルを日本語化 (Plan Execution → 計画実行 / Hard Stops → 停止条件 / Sign-off → 完了記録 / phrase trigger → 発話トリガー / evidence → 検証ログ / verbatim → そのまま / self-report → 自己申告)。「起草」を「書き方」「書き出し」に置換。固有名詞は英語残し。

### 判断 15: skill 命名を漢字熟語セットに

**結論**: kakunin / kentou / tsuiseki / shiken のローマ字書きでは意味が直感的に伝わらなかったため、責務に合わせた漢字熟語に変更: **sadoku (査読) / kouchiku (構築) / tansaku (探索) / shiken (試験)**。

`shikou` (試行) も検討したが、思考 (shikou) と打ち分けにくいため `shiken` (試験) を採用。

### 判断 16: 評価=環境変化の原則を完了記録に組み込み

**結論**: 完了記録の機械検証可能項目は、コマンド出力をそのまま引用する。「pass しました」「直りました」だけの自己申告は禁止 (§3.6)。

**根拠**: agent の自己申告は嘘をつき得る (出力を要約・装飾する誘惑がある)。環境がどう変わったか (test runner の最終 summary 行 / grep の 0 matches / fix 前後の挙動 diff) を verbatim で残すことで、後追い検証が可能になる。

### 判断 17: pr-template.md への完全集約

**結論**: PR 説明文モードは SKILL.md からは呼び出しのみ。template / 書き方 8 ステップ / 4 チェック / PII scan / 粒度ルールはすべて `sadoku/references/pr-template.md` に集約。

**根拠**: waza の「SKILL.md 軽量・要約 / references に詳細」哲学の徹底適用。PR フォーマット変更が頻発する局面で、SKILL.md を編集せず references の更新だけで完結する。

### 判断 18: 引き算 (hikizan) を skill 化せず全体原則として組み込み

**結論**: 「認知負荷削減」を志向する hikizan 思想を、新規 skill ではなく **kufuu の全体原則 (§3.8)** として組み込む。

**判断軸**:
- hikizan の中身を分解すると、半分は既に kufuu に組み込まれている (PR 粒度 → 判断 11、テスト最小化 → 判断 12、散文 4 チェック → §3.7)
- 新規要素 (選択肢提示 + 推奨度数値化、図優先) は agent の対話振る舞い全般に効く **meta 規律**、特定タスク起動より全 skill 共通の原則に向く
- skill 数を増やすのは Phase 1 slim 化 (判断 13) と逆行

**緩和策 (機械適用の形骸化を防ぐ)**:
- **数値推奨度**: 数値だけだと検証不能なので「N/10 + 1 行根拠」を必須化
- **「簡潔」の罠**: 結論だけ書く agent にならないよう「簡潔 = 結論先出し + 必要な根拠は残す」を明文化
- **選択肢の水増し**: 「明らかなら 1 案、迷い時のみ 2-3 案」、kentou の paralysis 防止 (判断 14) と整合

**適用先**:
- 全 skill の応答スタイル: 選択肢提示 + 推奨度
- `kouchiku` 通常検討モード: 推奨度 N/10 + 1 行根拠を出力に必須
- `sadoku` PR 説明文モード: 構造化情報は表 / mermaid 優先
- docs 全般: 表・mermaid・box diagram を散文より先に置く

### 判断 19: コンテキスト可視化を AI に委ねる (ふんわり取り込み)

**結論**: 「業務認知地図 / ベテラン暗黙知の外化 / ノービス支援」という強い概念を、§3.8 図優先の中にふんわり取り込み。新規 skill (例: `chizu`) は作らず、kouchiku / sadoku に新 mode も足さない。

**取り込んだ強い概念**:
- AI が「面倒で書けなかった図」の初稿を出し、人間が添削する分業
- 完璧な図ではなく「質問が出る図」を狙う (未完成が通常状態)
- ベテラン暗黙知の外化 (副次効果として、ベテラン自身の盲点発見にも繋がる)

**取り込まなかったもの**:
- 新 skill `chizu` (kufuu の射程外、別 pack で温める)
- 新 mode (kouchiku / sadoku への追加)
- ノービス判定機構 (誰がノービスかを skill が判定するのは困難)

**判断軸**:
- kufuu は code 中心の pack、業務プロセス全般は射程外 (= repo から読める範囲に絞る)
- 既存原則 §3.8 図優先と方向が一致するため、追記が最小コスト
- 強制力ある skill 化は kufuu の slim 化方針 (判断 13) と逆行

**形式**: mermaid に統一 (テキスト diff、git 親和性、code review の流れに乗る)

---

## 7. 残タスク

| 項目 | 優先度 | 備考 |
|---|---|---|
| 既存 waza skill との共存方針 | 中 | 同居 / 上書き / 別 namespace を決定 |
| `~/.claude/skills/` への deploy | 中 | 共存方針決定後 |
| `wt` shell tool 実装 | 低 | 並列実行頻度判明後に着手判断、現状は spec のみ |
| HTML 配布物の整理 | 低 | 旧 4 枚は削除、workflow.html だけ残す |

---

## 8. バージョニング方針

各 skill 個別に semver (waza 準拠):

- **major**: breaking change (mode 削除、output format 変更、skill rename)
- **minor**: 新 mode、新 reference
- **patch**: Gotcha 追加、wording 改善

skill 間で version を揃える必要はなく、独立 evolve。
