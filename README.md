# kufuu

日本語圏チーム開発向けの Agent Skills 対応 skill pack + git worktree CLI。tw93/Waza を起点に、SP (anthropic/superpowers) から選択的に取り込み、日本語圏 team-dev に最適化した **core 4 skill** と、並列開発を支える **`wt`** (worktree manager) を含む。

「waza (技)」に対して「kufuu (工夫)」— 動詞単位で責務を分けた skill 集。

- repo: <https://github.com/hayashiii-ghub/kufuu>
- license: MIT

## core 4 skill

| skill | 漢字 | 動詞 | 担当 |
|---|---|---|---|
| `sadoku` | 査読 | 見る・書く | code review / PR 説明文 |
| `kouchiku` | 構築 | 考える・作る | 設計判断 / 評価 / 計画策定 / 計画実行 |
| `tansaku` | 探索 | 追う | バグ調査 / root cause investigation |
| `shiken` | 試験 | 試す | TDD discipline / PRUNE |

動詞で 4 分割した役割境界が原則。`sadoku` は実装行為を含まない、`kouchiku` が設計から実装までを一気通貫で担う、`tansaku` はバグ専用、`shiken` は TDD discipline 専用。

## install

kufuu は [Agent Skills 標準](https://agentskills.io) に沿った **skill pack** です。skills-compatible agent へ `skills` CLI で配置できます。

### 推奨: `npx skills add` (1 コマンドで現在のハーネスに自動配置)

```bash
npx skills add github:hayashiii-ghub/kufuu
```

`-g` で global、`-a <agent>` で対象指定可能。詳細は [vercel-labs/skills](https://github.com/vercel-labs/skills) 参照。

### 動作確認済の検出例

```
◇  Found 4 skills
   sadoku    — PR code review and PR description authoring
   kouchiku  — Design decisions, plan drafting, kill/keep/pivot evaluation, and plan execution
   tansaku   — Bug debugging and root cause investigation
   shiken    — TDD discipline: write failing test first, witness it fail, ...
```

### 手動配置 (npx を使わない場合)

`skills/<name>/` を各ツールの skills dir にコピー or symlink:

| ツール | path |
|---|---|
| Claude Code | `~/.claude/skills/` (global) または `./.claude/skills/` (project) |
| Cursor / Codex / OpenCode / Cline 等 | `~/.agents/skills/` または `./.agents/skills/` (共通標準) |
| Cursor 個別 | `.cursor/skills/` も対応 |

### bin/wt (worktree CLI)

kufuu 同梱の bash CLI、`npx skills add` 対象外。手動 install:

```bash
ln -s "$(pwd)/bin/wt" ~/.local/bin/wt
wt help
```

### 既存 waza skill との衝突

waza の `kakunin` / `kentou` / `tsuiseki` 等とは skill 名が違うので衝突しません (kufuu は sadoku / kouchiku / tansaku / shiken)。共存可能。

## install (wt — git worktree CLI)

並列開発で worktree を扱うための CLI。skill とは独立、bash 単体。

```bash
# symlink (推奨)
mkdir -p ~/.local/bin
ln -s "$(pwd)/bin/wt" ~/.local/bin/wt

# PATH に ~/.local/bin が無ければ追加
export PATH="$HOME/.local/bin:$PATH"

wt help
```

### 使い方の例

```bash
wt new feat-A             # .worktrees/feat-A/ 作成 + branch 切り
wt ls                     # 全 worktree 一覧
wt status feat-A          # 詳細
wt rm feat-A              # 削除 (未 commit / 未 push があると拒否)
wt cleanup --dry-run      # merged 済 worktree の削除候補
cd "$(wt enter feat-A)"   # worktree に cd
wt new feat-B --launch claude  # 作成して Claude を起動
```

`.worktrees/` は repo 内に作られるので、 `.gitignore` に追加するのを推奨 (kufuu 自身は既に追加済)。

## quick start

各 skill は発話トリガーで起動する。

```
"設計どうする"           → kouchiku 通常検討
"計画実行" / "進めて"     → kouchiku 計画実行
"レビューして"           → sadoku 通常レビュー
"PR文書いて"            → sadoku PR 説明文
"エラー" / "動かない"     → tansaku 通常追跡
"TDDで" / "テストから書いて" → shiken
```

詳しい trigger 一覧と mode 切替は `docs/workflow.md` (mermaid 図入り) を参照。

## 設計原則

1. **waza 哲学の継承**: 1 skill 多 mode、references 分離、scripts で決定論的処理
2. **Controller Owns Information**: 情報取得目的の subagent は default で使わない
3. **Inline default + subagent 明示 gate**: subagent は (a) 重い情報取得 / (b) Specialist review / (c) 機械 fan-out の 3 条件のみ
4. **SP から選択的取り込み**: Announce-at-start / worktree Step 0 検出 / Hard Rules 冒頭の 1 文ガード。Iron Law / Red Flags / Rationalization 表は不採用
5. **日本語圏最適化**: skill name は英語短語、本文は日本語、固有名詞 (TDD, mock, RED/GREEN/REFACTOR/PRUNE 等) は英語残し
6. **評価は「環境変化」で見る**: 完了記録の機械検証可能項目は command 出力をそのまま引用、自己申告は禁止
7. **散文は「伝わりやすさ」のみ**: 4 チェック (結論先出し / 1 段落 1 主張 / 読み手語彙 / 儀礼削除)
8. **引き算 (認知負荷削減)**: 選択肢提示 + 推奨度 N/10 + 1 行根拠 / 図優先 / 読み手の負荷を最優先。kufuu の他原則 (PR 粒度・テスト最小化) と同じ「引き算」哲学を全 skill に貫通させる
9. **工数はトークンベース**: 判断軸を「行数 / 人間時間」から「token 消費 / context 占有 / API コスト」に切り替え。実行者は AI agent 前提 (§3.9 / 判断 22)
10. **ファクトチェック**: 知識カットオフ後 / 不確実な事実は利用可能な検索・fetch・一次ソースで裏取りしてから断定 (§3.10 / 判断 23)

詳細と 25 件の設計判断ログは `docs/DESIGN.md` を参照。

## ディレクトリ構成

```
kufuu/
├── README.md                    ← この入口 (人間中心、GitHub で最初に表示)
├── AGENTS.md                    ← AI agent 入口 + Working Agreements + skill trigger 表
├── LICENSE                      ← MIT
├── .gitignore
├── bin/
│   └── wt                       ← git worktree CLI (bash)
├── skills/                      ← skill 実装本体 (SoT、npx skills add で配置)
│   ├── sadoku/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── pr-template.md
│   │       ├── project-context.md
│   │       ├── persona-catalog.md
│   │       └── agents/
│   │           ├── reviewer-security.md
│   │           └── reviewer-architecture.md
│   ├── kouchiku/SKILL.md
│   ├── tansaku/
│   │   ├── SKILL.md
│   │   └── references/logging-techniques.md
│   └── shiken/
│       ├── SKILL.md
│       └── references/testing-anti-patterns.md
├── adapters/                    ← Agent Skills 標準でカバーできない特殊ケース用 (現状は予約領域)
│   └── README.md
└── docs/                        ← 設計ドキュメント
    ├── DESIGN.md                ← 設計書 + 設計判断ログ 25 件 (開発者向け)
    └── workflow.md              ← 使い方ガイド (利用者向け、mermaid 図入り)
```

## version

各 skill 個別に semver:
- sadoku: 3.0.0 (レビュー咀嚼モード廃止で major)
- kouchiku: 2.0.0 (計画実行モード追加)
- tansaku: 2.0.0 (改名)
- shiken: 3.0.0 (改名 + 漢字ラベル変更)
- wt: 0.1.0 (MVP)

## ライセンス / 出典

- License: MIT (`LICENSE` 参照)
- ベース: [tw93/Waza](https://github.com/tw93/Waza)
- 参考: [anthropic/superpowers](https://github.com/anthropic/superpowers)

## contributing

issues / PRs を歓迎します。新しい skill 追加よりも、既存 skill の磨き込みを優先する pack 哲学です (詳細は `docs/DESIGN.md` の判断 13 を参照)。
