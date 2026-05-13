# kufuu 設計書

**バージョン**: 3.4
**更新日**: 2026-05-13
**ベース**: tw93/Waza skill 群
**対象**: Agent Skills 対応 agent / skill pack 配布

実装本体は `../skills/*/SKILL.md` を SoT とする。本書は「設計の意思決定ログ」と「全 skill 共通の原則」を扱う。

---

## 1. 目的とスコープ

日本語圏チーム開発において、AI coding agent が「コードを書く」作業を一貫した discipline で実行するための skill 群を定義する。waza skill 群を起点とし、SP (anthropic/superpowers) から選択的に取り込み、日本語圏 team-dev の現実に最適化する。

### 対象作業
- 機能実装 (TDD discipline 込み)
- バグ調査 / root cause investigation
- code review / PR 説明文
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

**役割境界の原則**: 動詞単位で 4 分割。`kouchiku` は controller として設計から計画実行までを持つが、原因調査は `tansaku`、TDD discipline は `shiken`、レビュー / PR 文は `sadoku` に handoff block で渡す。

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

- `wt` — git worktree shell tool (実装済 v0.1.0、`bin/wt help` で詳細、判断 24 で WT-SPEC.md は廃止)

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
| gate (a) 重い情報取得 | 5+ ファイル横断 (parent context が 50k+ token を消費する見込み) | "Where is Foo used across the codebase?" |
| gate (b) Specialist review | 大規模 diff (500+ 行 ≈ 5-15k token) の security / architecture 分析 | sadoku の専門家レビュー |
| gate (c) 機械 fan-out | 5+ 独立機械タスク、並列化で wall clock が 1/N に縮む (spec watertight、視覚判断不要) | mass rename, doc generation |

しきい値の根拠は §3.9 (token ベース工数評価) を参照。

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
| frontmatter `description` | 英語中心 + 主要な日本語 trigger。Agent Skills の discovery SoT |
| `when_to_use:` keywords | 非標準の補助メモ。日英短語のみ、発火の SoT にしない |
| 本文 Mode 名 / 節タイトル / ラベル | 日本語 |
| Hard Rules 宣言文 | 日本語 (圧力下の自己参照テキスト、母語が効く) |
| 固有名詞 (TDD, mock, RED/GREEN/REFACTOR/PRUNE 等) | 英語残し |
| agent の応答 (自然文) | ユーザの問い合わせ言語に合わせる (日本語問い合わせなら、説明 / 要約 / 提案理由 / 質問は日本語、skill 内 label と固有名詞は英語残し) |

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

### 3.6.1 skill frontmatter / trigger 方針

Agent Skills の発火は `name` と `description` が主戦場。`when_to_use` はクライアントによって読まれない可能性があるため、**発火の SoT にしない**。

**方針**

1. `description` は `What it does. Use when ...` の 2 文構成を基本にする
2. 日本語 trigger は主要 phrase だけ入れる (各 skill 4-8 個まで)
3. 同義語の網羅はしない (`落ちる` / `クラッシュ` など実運用で強い語だけ)
4. `when_to_use` は後方互換 / 人間可読の補助メモとして短く残す
5. body のモード表にない trigger を frontmatter に足さない

**理由**: description を薄くしすぎると discovery が落ちる。一方で keyword stuffing は誤発火と保守コストを増やす。kufuu は「明示発話 + 状態/意図」の最小セットで発火面を作る。

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

1. **選択肢を提示する**: open question (「どうしますか?」) を投げず、案 + 推奨を出す。ただし**明らかな場合は 1 案でよい** (`kouchiku` の「3 案以上は出さない」と整合、迷うときのみ代替案 1 つ)
2. **推奨度を数値化**: 各案に **N/10 の推奨度 + 1 行根拠** を付ける。数値だけでは検証不能なので根拠必須
3. **簡潔 ≠ 短い**: 結論先出し + 必要な根拠は残す。儀礼や前置きは削るが、判断の根拠は削らない
4. **構造化できる情報は図**: 表 / mermaid / box diagram で構造を見せる。判断のニュアンスは散文で書く (「図 vs 文」の使い分け)
   - **コンテキスト可視化は AI が初稿**: 従来「面倒で書けなかった依存関係や前提」を、AI が初稿、人間が添削する分業で外化する
   - **未完成が通常状態**: 完璧な図を狙わず「質問が出る図」を作る。更新は質問発生時のみで十分 (図の保守地獄を避ける)
   - **形式は mermaid に統一**: テキストで diff が取れる、git 親和性高い、code review の流れに乗る
5. **読み手の負荷を最優先**: agent 側の手間より、user / reviewer の認知負荷を下げる側に立つ
6. **表記の統一**: 選択肢ラベルは用途で固定する (混在は読み手にノイズ)
   - 選択肢 (alternative): 大文字 `A` / `B` / `C` / `D`
   - 分類 (category, gate, mode 等): 小文字 `(a)` / `(b)` / `(c)`
   - 手順 (sequence): 数字 `1.` / `2.` / `3.`
   - ギリシャ文字 (α / β) や混在は不採用、判断ログでも統一

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
- 「明らかなら 1 案、迷うときのみ代替案 1 つ」(無用な選択肢水増しを防ぐ、`kouchiku` paralysis 防止と整合)

**他原則との関係**

- 散文 4 チェック (§3.7) は引き算の言い換え。引き算原則が上位概念
- `kouchiku` の「3 案以上は出さない」(判断 14) と同根、選択肢の上限を共有
- 評価=環境変化 (§3.6) と独立: 引き算は「読み手の負荷」、§3.6 は「検証可能性」を扱う

### 3.9 工数はトークンベースで評価

実行者は AI agent (Claude / Codex / Cursor 等) であり、人間ではない。判断軸を **「人間が読む行数 / かかる時間」から「token 消費 / context window 占有 / API コスト」に切り替える**。

**評価軸の置き換え**

| 旧 (人間ベース) | 新 (AI ベース) |
|---|---|
| 「3+ ファイル横断は重い」 | input token (3 ファイル ≈ 3-15k token、context の 1-7%) |
| 「500+ 行 diff は Standard レビュー」 | 500 行 ≈ 5-15k token、subagent gate (b) の妥当範囲 |
| 「人間が 10 分かかる作業」 | API 1 回 = 数秒 + 数千〜数万 token、コスト = $0.01-0.1 |
| 「面倒で図を書かない」 | AI なら数秒で書ける (引き算原則 §3.8 図優先と整合) |

**subagent gate のしきい値再定義**

| gate | 旧条件 | 新条件 (token + cost ベース) |
|---|---|---|
| (a) 重い情報取得 | 5+ ファイル横断 | parent context が 50k+ token を消費する見込み = 別 context window で隔離 |
| (b) Specialist review | 500+ 行 diff | review 対象が 10k+ token、専門性で response 品質が変わる |
| (c) 機械 fan-out | 5+ 独立タスク | 並列化で wall clock が 1/N に縮む、token cost は N 倍 |

**判断時に意識すること**

- inline 実行 = parent context が膨らむが、結果が直接 main に残る (検証ログ verbatim 引用が容易)
- subagent 実行 = parent context は守られるが、結果は要約されて戻る (情報損失あり、裏取り必要)
- token cost と response 品質のトレードオフを意識 (10k token の subagent 1 回 vs inline 5k token × 2 回)

**人間時間が意味を持つ場面**

- 視覚確認 / UI 動作テスト (人間の眼が必要)
- レビュアーへの返信文 (受け取り手が人間)
- 完了記録の読み返し (ベテランがレビュー)

これらは引き算原則 §3.8「読み手の負荷を最優先」と接続。**AI が出力構造化に時間 (token) を使い、人間が読む時間を最小化する** のが kufuu の基本姿勢。

### 3.10 ファクトチェック原則

**自分の知識で裏取りできない事実は、外部ソースで確認するまで断定しない**。AI agent の知識カットオフ後の事実は、内部記憶で答えると fabrication になる。

**裏取りトリガー (いずれか 1 つでも該当)**

- agent の知識カットオフ後 (例: 「最新の X バージョン」「2026 年の Y 標準」)
- 外部 OSS / SaaS の現行仕様 (npm パッケージ、SaaS 機能、IDE 拡張など)
- 標準 / specification の存在確認 (「そういう標準あるの?」)
- リアルタイム性が要る事実 (株価、API 稼働状況、最新ニュース)

**裏取り手段 (優先順)**

| 手段 | 用途 | 引用形式 |
|---|---|---|
| 1. 一次ソース直接 fetch (公式 docs / GitHub README / 公式 changelog) | 最も信頼度高 | URL + 該当行引用 |
| 2. 検索特化 tool (引用付き) | 複数ソース統合、cross-check | 引用 link 必須 |
| 3. リアルタイム検索 tool | 最新動向、リアルタイム情報 | 引用 link 必須 |
| 4. WebSearch / ハーネス標準検索 | 一次ソース URL 発掘の入口 | 検索結果から 1 へ |

**裏取りせずに答えてよい場面**

- 自分の出力した内容の参照 (この conversation 内)
- 言語仕様 / アルゴリズムの基礎 (カットオフ前から不変)
- repo 内の事実 (grep / read で直接確認可能)

**やってはいけないこと**

- 「おそらくこうだと思います」で断定的に答える (信頼性を損なう)
- 確認していない URL を fabricate する
- 過去の knowledge cutoff 時点の状況を最新と誤認する

**実例 (このリポジトリで実際に起きた失敗)**: 2026/05 の kufuu 設計時、私 (Claude) は Agent Skills 標準を「fabricated 疑い」と誤判断した。WebSearch / WebFetch で先に裏取りすれば防げた失敗。判断 23 はこの実例から立てた。

### 3.11 Skill handoff 原則

**kouchiku は controller、専門 skill は discipline owner** として扱う。controller が全工程を抱え込むのではなく、次の skill が迷わず再開できる固定成果物で渡す。

| 条件 | handoff 先 | 渡すもの |
|---|---|---|
| 原因未確定 / 再現不明 / 予期しない test failure | `tansaku` | 症状、期待値、実際値、evidence、expected return |
| root cause 確定済み bugfix / 純ロジック / API / ビジネスルール | `shiken` | spec、root cause、failing behavior、test target |
| 実装完了 / PR open 前 | `sadoku` | change intent、files changed、verification、scope notes |

共通 handoff block:

```text
handoff: [skill]
reason: [なぜ今渡すか]
context: [症状 / 仕様 / 設計判断]
evidence:
  - [file:line / command output / logs]
expected return:
  - [戻してほしい成果物]
```

**判断軸**:
- kouchiku が「たぶんこう」で原因調査を代替しない
- tansaku が fix 方針を広げすぎず、regression guard は shiken に渡す
- shiken は GREEN 後に設計判断を広げず、検証ログ付きで戻す
- sadoku は diff だけでなく前段の判断 / 検証ログを review evidence として読む

---

## 4. skill 別 要約

仕様の本体は `../skills/*/SKILL.md`。本節は俯瞰用の要約のみ。

### 4.1 `sadoku` (査読) — 見る・書く

通常レビューと PR 説明文の 2 モード。実装行為 (計画実行) は `kouchiku` に移管、TDD は `shiken`、バグ調査は `tansaku` に分離。reviewer コメントへの返信文ドラフトは skill mode 化せず通常会話で対応。

`references/pr-template.md` に PR 説明文モードの本体 (5 セクション template / 8 ステップの書き方 / 4 チェック / PII scan / 1 issue = 1 PR ルール) を集約。`references/persona-catalog.md` で 3 persona (security / architecture / adversarial) の起動条件を定義、agent prompt は `references/agents/reviewer-{security,architecture}.md`。

### 4.2 `kouchiku` (構築) — 考える・作る

軽量検討 / 通常検討 / 評価 / 計画実行の 4 モード。設計から計画実行までを controller として担う。

通常検討モードの出力は前提崩し / 攻撃検証 / owner skill 付き Plan steps を必ず含む。承認後に「計画実行」「進めて」で同 skill 内の計画実行モードへ移行。計画実行中に原因未確定の挙動に遭遇したら `tansaku`、TDD 必要層に触れたら `shiken` に handoff する。実装完了後は `sadoku` に PR レビューを渡す。

### 4.3 `tansaku` (探索) — 追う

通常追跡 / 二分探索 / 再発追跡の 3 モード。**hypothesis を 1 文書けるまで実装を変更しない**。grep / read / log 収集 / test 再現は調査行為として許可。1 hypothesis = 1 instrument の原則で、`references/logging-techniques.md` から 1 パターンだけ仕込む。fix 確定後は前後の挙動 diff をそのまま引用 (要約禁止)。

### 4.4 `shiken` (試験) — 試す

`TDDで` / `テストから書いて` の発話 trigger、または層分け判定で起動。RED → GREEN → REFACTOR → **PRUNE** のサイクル。

層分けトリガー: 純ロジック / API / バグ修正は強制、インタラクションは推奨、純スタイル / アニメ / 文言はスキップ可。PRUNE 後に残った各 test は対象実装を一時的に壊して fail を目視する (失敗しない test は削除対象)。unrelated dirty file がある worktree で作業全体を `git stash` しない。

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

検出結果は完了記録の `worktree` 行に出力。worktree の作成・削除は `wt` shell tool の責務であり、skill 側は行わない。

### 5.3 subagent ガイドライン

- default は inline TodoWrite
- subagent 起動条件 = 3.3 の (a)(b)(c) のみ
- subagent prompt の必須項目: scope / context / constraints / expected output format
- subagent 成果物の受け取り: git diff / file 読み / test 再実行で**必ず main 側で裏取り**
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

**根拠**: 日本語圏 team-dev では中英 bilingual rule 不要、SNS は scope 外、design doc 一貫性は sadoku scope 外、tw93 個人 OSS 固有は移植不可。残るのは PR description / commit message / release notes 関連のみ。

### 判断 08: worktree 管理は skill ではなく shell tool 側に

**結論**: skill には Step 0 検出 (`GIT_DIR != GIT_COMMON`) のみ。worktree 作成・削除・list は `wt` shell tool に集約。

**根拠**: SP/using-git-worktrees の 215 行を skill に持たせるのは過剰。並列実行頻発時、shell 側で list/status を持つほうがチームの可視性が高い。waza の環境 agnostic 思想を維持 (shell 未導入環境でも skill は動く)。ただし `wt` 自体の実装は並列頻度測定後に判断、現時点では spec のみ確保。

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

**結論**: 旧版 `sadoku` は「review (見る)」「Plan Execution (作る)」「Reviewer Feedback (返す)」が併存し性格が混ざっていた。v2.0 で **Plan Execution を `kouchiku` に移管**、v3.0 で **レビュー咀嚼モードも廃止**。`sadoku` は「見る・書く」に純化 (2 モードのみ)、`kouchiku` は「考える・作る」を一気通貫で担う。

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
- **選択肢の水増し**: 「明らかなら 1 案、迷うときのみ代替案 1 つ」、`kouchiku` の paralysis 防止 (判断 14) と整合

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

### 判断 20: ハーネス agnostic 化を adapters/ 構造で実現 (※ 判断 21 で見直し)

> **更新**: 判断 21 で Agent Skills 標準 (agentskills.io) への準拠が決まったため、`adapters/` 階層は最小化された。本判断は「標準を知らない時点での独自設計」として記録のみ残す。

**結論**: skill 本体 (`skills/`) はハーネス (Claude Code / Cursor / Codex 等) agnostic な SoT として保ち、各ハーネス固有の配置 / 変換は `adapters/<harness>/` に隔離する。`AGENTS.md` を root に置いて補助 instruction 標準にも対応。

**理由**:
- kufuu は **multi file pack** (skill 4 + references + agents)。single file 標準 (AGENTS.md だけ / `.cursorrules` だけ) では skill 切り分けができない
- 各ハーネス用 dir (`.claude/` `.cursor/` `.codex/`) を root に並べる構造はメンテ地獄 (アンチパターン)
- 「SoT + 変換 adapter」は新ハーネス対応時に `adapters/<name>/` を 1 つ作るだけで済む

**実装フェーズ**:

| Phase | 内容 | きっかけ |
|---|---|---|
| 0 | `skills/` SoT + Claude Code symlink 案内 (旧 README) | 初版 |
| **1** | `adapters/claude-code/` 作成、root README から deploy 手順を移動 | 対称性確保 |
| **2** | `AGENTS.md` 追加 (repo 入口、ツール agnostic) | 他ツール対応の宣言 |
| **3 (土台)** | `adapters/cursor/` `adapters/codex/` スケルトン作成 (TODO + 変換マッピング案のみ) | 着手見込みの可視化 |
| 4 (将来) | SKILL.md 内の Claude Code 固有用語 (Task, TodoWrite) を抽象化 | 2 個目の adapter 実装と同時 |
| 5 (将来) | 各 adapter の `generate.sh` / 動作確認 | 実利用者からの要望で個別着手 |

**採用しない選択肢**:
- ❌ `.cursor/` `.codex/` `.claude/` を root に並列 (重複メンテ地獄)
- ❌ 需要が無いうちから空の adapter を作りすぎる (YAGNI 違反、土台のみで止める)
- ❌ AGENTS.md だけで済ませる (multi file pack の skill 切り分けができない)

**形式の判断**:
- `AGENTS.md` は OpenAI 提唱の標準、Cursor 等が対応進行中。kufuu でも入口として置く価値あり (補助、SoT は skills/)
- adapter 内の変換は需要が出てから (引き算原則の YAGNI と整合)

### 判断 21: Agent Skills 標準 (agentskills.io) への準拠 (判断 20 を上書き)

**結論**: kufuu は **Agent Skills 標準** (agentskills.io) に沿った skill pack とする。判断 20 (adapters/ 階層) は廃止、`skills/` を SoT として保持しつつ `npx skills add` 経由で skills-compatible agent に配置できる構造にする。Codex plugin / MCP plugin としては扱わない。

**経緯**:
- 判断 20 を立てた時点 (2026/05) では「ハーネス独自仕様の matrix」を前提に adapters/ で変換する設計を考えていた
- しかし実際は Agent Skills 標準が公開され、複数の agent / CLI が対応済
- kufuu の `skills/<name>/SKILL.md` 構造は **既に標準準拠**だった (frontmatter `name` / `description` あり、検出順序「skills/」は標準の探索パス第 2 位)

**検証**: `npx skills add github:hayashiii-ghub/kufuu --list` で 4 skill (sadoku / kouchiku / tansaku / shiken) が正しく認識されることを確認 (2026/05/13)。

**変更点 (判断 20 → 判断 21)**:
- `adapters/<harness>/` 階層 (Claude Code / Cursor / Codex のスケルトン) を削除、`adapters/README.md` 1 枚に集約
- root `README.md` の install 手順を `npx skills add` 主役に書き換え
- `skills/sadoku/agents/` を `skills/sadoku/references/agents/` に移動 (標準は `references/` のみ、`agents/` は kufuu 独自命名だった)

**標準準拠で得られたもの**:
- skills-compatible agent に同じ skill pack を配置できる
- `npx skills add github:hayashiii-ghub/kufuu` の 1 コマンドで配置可能
- adapter メンテコストゼロ
- 配布性向上 (Agent Marketplace 等への公開も技術的に可能)

**学び**: 「標準が無い」と推定して独自仕組みを作るより、**先に標準を確認するべき** (この経験は判断 23 ファクトチェック原則の根拠にも繋がる)。

### 判断 22: 工数評価を AI トークンベースに切り替え

**結論**: kufuu の判断軸を「人間が読む行数 / かかる時間」から「AI agent の token 消費 / context window 占有 / API コスト」に切り替える (§3.9)。subagent gate のしきい値も token + cost ベースで再定義。

**判断軸**:
- kufuu の実行者は AI agent (Claude / Codex / Cursor) であり、人間ではない
- 「3+ ファイル横断」「500+ 行 diff」のような行数ベース判定は人間の認知負荷を前提にしていた、AI には別の単位が要る
- token cost (`$0.01-0.1` / 呼び出し) と context 占有率がボトルネック

**残す人間軸**:
- 視覚確認 / UI テスト
- レビュアーへの返信文 (受け取り手が人間)
- 完了記録の読み返し

**他原則との関係**:
- §3.8 引き算原則「読み手の負荷を最優先」を AI と人間の両軸で再解釈
- §3.6 評価=環境変化と独立 (こちらは検証可能性、§3.9 は判断軸)

### 判断 23: ファクトチェック原則 (自分の知識で裏取り不可な事実は外部ソース確認)

**結論**: AI agent の知識カットオフ後 / 不確実な事実は、利用可能な検索・fetch・一次ソースで**裏取りしてから断定する**。kufuu の skill / DESIGN 全体に Hard Rule として組み込む (§3.10)。

**直接のきっかけ (実害例)**:
- 2026/05、Cursor の skills 機能 / Agent Skills 標準 / skill marketplace を私 (Claude) は「fabricated 疑い」と推定で否定した
- 実際は存在していたが、当時の私の知識だけでは判断できない事実だった
- WebSearch を最初にしていれば防げた誤判断、kufuu 設計が一時的に誤った方向 (判断 20 の adapters/ 構造) に進んだ

**裏取り手段の優先順** (§3.10):
1. 一次ソース直接 fetch (公式 docs / GitHub / changelog)
2. 検索特化 tool (引用付き)
3. リアルタイム検索 tool
4. WebSearch / ハーネス標準検索 (一次ソース URL の入口)

**他原則との関係**:
- §3.2 Controller Owns Information の「Never state from memory. Run grep first」を、外部事実にも拡張
- 判断 16 (評価=環境変化) の「自己申告禁止」と同根、不確実な事実を断定しない discipline

**緩和策 (機械適用の形骸化を防ぐ)**:
- 「裏取り不要」場面 (repo 内事実、言語仕様基礎、conversation 内参照) を明文化、毎回 web 検索しない
- 不確実なら「不確実」と言う、断定しないことを許容 (= 「分かりません」が悪い回答ではない)

### 判断 24: docs を 2 ファイルに集約 + 選択肢表記の統一

**結論**: `docs/` の 4 ファイル (DESIGN.md / workflow.md / workflow.html / WT-SPEC.md) を **2 ファイル (DESIGN.md + workflow.md) に集約**。`workflow.html` と `WT-SPEC.md` を削除。同時に §3.8 引き算原則に「選択肢表記の統一」を追加 (大文字 `A/B/C` / 小文字 `(a)/(b)/(c)` / 数字 `1./2./3.` の使い分け)。

**削除した 2 ファイルと理由**:

| ファイル | 削除理由 |
|---|---|
| `docs/workflow.html` | `workflow.md` と内容重複、GitHub 上で mermaid render される、二重メンテのコスト > 視覚効果のメリット |
| `docs/WT-SPEC.md` | wt v0.1.0 実装済、`bin/wt help` で網羅可能、draft 時代の仕様書は役目を終えた |

**残した 2 ファイルの役割分担**:
- `DESIGN.md`: 設計判断ログ、開発者 (kufuu を改良したい人) 向け
- `workflow.md`: 使い方 + mermaid 図、利用者 (kufuu を使う人) 向け

**選択肢表記統一 (§3.8 原則 6)**:
- 選択肢 (alternative): `A` / `B` / `C` / `D`
- 分類 (gate / mode / category): `(a)` / `(b)` / `(c)`
- 手順 (sequence): `1.` / `2.` / `3.`
- ギリシャ文字 (α / β) や混在は不採用

**判断軸**:
- 引き算原則 §3.8 と整合 (二重メンテ削減、表記の認知ノイズ削減)
- 「設計判断は捨てない」原則は維持 (= workflow.md だけに統合する案 A は採らなかった)
- 利用者と開発者の concerns を分離 (1 ファイル化で両者に無関係な情報を読ませない)

### 判断 25: frontmatter trigger は description 主、when_to_use 補助

**結論**: Agent Skills の discovery SoT は `description` とし、`when_to_use` は非標準の補助メモとして短く残す。各 skill の `description` には主要な日本語 trigger と状態/意図 trigger を入れるが、同義語の網羅はしない。

**判断軸**:
- `description` が薄いと skill discovery が落ちる
- keyword stuffing は誤発火と保守コストを増やす
- `when_to_use` はクライアントによって読まれない可能性があるため、発火条件の SoT にできない

**運用ルール**:
- `description` は `What it does. Use when ...` の 2 文構成
- 日本語 trigger は各 skill 4-8 個まで
- body のモード表に存在しない trigger を frontmatter に追加しない
- `when_to_use` は後方互換 / 人間可読メモとして残す

### 判断 26: skill 間 handoff を固定 block 化

**結論**: `kouchiku` は controller として計画と判断を持つが、原因調査 / TDD / レビューの discipline は `tansaku` / `shiken` / `sadoku` に固定 handoff block で渡す (§3.11)。

**変更した境界**:
- 旧: `kouchiku` が「設計から実装まで一気通貫」として広く抱える
- 新: `kouchiku` は controller、専門 skill は discipline owner。実装を進める途中でも原因未確定なら `tansaku`、TDD 必要層なら `shiken`、実装完了後は `sadoku`

**固定 block の理由**:
- 自然文の「次は shiken で」だけでは、受け手が何を根拠に再開するか曖昧になる
- `reason / context / evidence / expected return` を揃えると、次 skill が推測で補完しない
- `sadoku` が diff だけでなく前段の判断と検証ログを review evidence として読める

**採らなかった案**:
- skill をさらに細分化する案: core 4 の学習コストが上がるため不採用
- kouchiku から実装権限を完全に外す案: 小さい修正の摩擦が増えるため不採用
- handoff を docs だけに置く案: 実際に発動する SKILL.md 側で見えないため不採用

### 判断 27: agent 応答言語をユーザ問い合わせ言語に合わせる

**結論**: agent の自然文応答 (説明 / 要約 / 提案理由 / 質問) は、ユーザの問い合わせ言語に合わせる。日本語の問い合わせには日本語ベースで返し、skill 内の英語 label (RED / GREEN / handoff key / frontmatter 用語) と固有名詞 (TDD, mock, controller 等) のみ英語残し (§3.5)。

**動機**:
- 日本語会話中に英語句が不必要に混入 (例: 「implementation touches API behavior」「proceed しますか?」) が認知ノイズになる
- §3.5 は **skill の内容** (SKILL.md 本体) の言語方針を定義していたが、**agent の応答そのもの** の言語方針は未定義だった
- 判断 09 (日本語圏最適化の粒度) を skill 内容だけでなく agent 応答にも拡張

**運用ルール**:
- 自然文 (説明 / 要約 / 推論 / 提案理由 / 質問) は問い合わせ言語に合わせる
- skill 内 label / frontmatter 用語 / 固有名詞は英語残し (一貫性のため)
- 例文・サンプル値も日本語ベース (例: `reason:` の value は日本語で書く)

**他原則との関係**:
- §3.5 (日本語圏最適化) の延長、適用範囲を「skill 内容」から「agent 応答」まで広げた
- §3.7 (散文の 4 チェック) の「語彙が読み手のもの」と同根

### 判断 28: simplify 相当を sadoku の新 mode として取り込み

**結論**: claude-code の `simplify` 相当 (重複削除 / 命名統一 / 不要な抽象化除去 / dead code / efficiency) を、新 skill ではなく `sadoku` の新モード **simplify findings** として取り込む。発見は sadoku、実装は kouchiku に handoff 委譲し、sadoku の「見る・書く」純化 (判断 14) と整合させる。`コードレビュー` / `コードレビューして` は compound trigger として通常レビュー → simplify findings を順に実行。

**動機**:
- 「書いた後の production code 整理」工程が core 4 で明示的に持たれていなかった (shiken PRUNE は test 専用、§3.8 引き算原則は meta-principle のみ、sadoku 停止条件「テスト最小性違反」は test 寄り)
- claude-code に `simplify` コマンドが存在し、kufuu でも同等の発見能力が欲しい (運用実感)
- §3.8 引き算原則の operationalization (per-PR で actionable な判定基準に落とす)

**採用しなかった案**:

| 案 | 不採用理由 |
|---|---|
| A. 新 skill `seiri` (整理) を立てる | 判断 13 (Phase 1 slim 化、4 skill 限定) を覆す必要があり、現状の運用実績で覆すまでの根拠はない。学習コスト + 1 skill |
| B. 新 skill `hikizan` (引き算) | §3.8「引き算原則」と命名衝突、概念の二重化 |
| C. `kouchiku` 計画実行に「整理 phase」を追加 | controller の責務が膨らみ、判断 26 の dispatcher 原則 (kouchiku は専門 skill の責務を内包しない) と矛盾 |

**設計のポイント**:

1. **発見と実装を分離**: sadoku は発見と提案まで、実装は kouchiku に handoff block で委譲 (判断 14 と整合)
2. **明示 opt-in only**: simplify findings モードは発話 trigger のみ、state trigger を持たない (default 通常レビューを汚染しない、worst case 防御)
3. **severity 付き**: high / medium / low、kouchiku に振るのは high severity が default。medium / low は user 判断 (PR 説明文の「実装中に分かったこと」に記録 or 据え置き)
4. **compound trigger**: `コードレビュー` は通常レビュー + simplify findings の両方を順に実行 (語の重みで thoroughness を識別)、ただし出力は独立 section で混ざらない

**worst case 対策**:
- 些細な指摘ばかり出て critical 指摘 (停止条件) が埋もれるリスク → severity 付け + 独立 section 出力 + 「findings: 0 なら明示」ルールで対応
- claude-code simplify は「自分で fix する」が、kufuu は discipline 分離を保つために「propose + delegate」を採用

**他原則との関係**:
- §3.8 引き算原則 の operationalization (meta-principle → per-PR 判定基準)
- 判断 12 (shiken PRUNE) と相補 (PRUNE = test 専用、simplify findings = production code 専用)
- 判断 14 (sadoku 純化) を守る (sadoku は実装しない、発見と提案のみ)
- 判断 26 (skill 間 handoff 固定 block 化) を継承 (high severity は kouchiku に handoff block で渡す)

**運用観察項目** (3 ヶ月後に判断 04 と同じく実害ベースで再評価):
- simplify findings の採用率 (high severity → 実装に至る割合)
- compound trigger 経由の利用頻度 (= 「コードレビュー」と発話する頻度)
- 些細な指摘で critical が埋もれた事例の有無

---

## 7. 残タスク

Phase 1 設計フェーズは概ね完了。直近の運用観察待ち。

| 項目 | 状態 | 備考 |
|---|---|---|
| 既存 waza skill との共存方針 | 完 | skill 名が違う (sadoku/kouchiku/tansaku/shiken vs waza の kakunin/kentou/tsuiseki/shiken) ので衝突なし、README に明記 |
| `~/.claude/skills/` への deploy | 完 | 判断 21 で `npx skills add` 経由に統一、README に手順 |
| `wt` shell tool 実装 | 完 | v0.1.0 実装済 (`bin/wt`、391 行 bash)、smoke test 通過 |
| HTML 配布物の整理 | 完 | 判断 24 で workflow.html / WT-SPEC.md 削除、docs は DESIGN.md + workflow.md の 2 つに |
| 実 deploy で試運転 | 未 | `npx skills add github:hayashiii-ghub/kufuu` で実 task で動かす |
| factcheck 原則の運用検証 | 未 | 利用可能な検索 / fetch tool を実利用で詰める |

---

## 8. バージョニング方針

各 skill 個別に semver (waza 準拠):

- **major**: breaking change (mode 削除、output format 変更、skill rename)
- **minor**: 新 mode、新 reference
- **patch**: Gotcha 追加、wording 改善

skill 間で version を揃える必要はなく、独立 evolve。
