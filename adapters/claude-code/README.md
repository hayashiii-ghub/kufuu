# Claude Code adapter

Claude Code は `~/.claude/skills/` 配下を skill として読み込む。kufuu の `skills/` をその下に配置する。

## 方法 1: symlink (推奨、編集が即反映される)

```bash
ln -s "$(pwd)/skills/sadoku"   ~/.claude/skills/sadoku
ln -s "$(pwd)/skills/kouchiku" ~/.claude/skills/kouchiku
ln -s "$(pwd)/skills/tansaku"  ~/.claude/skills/tansaku
ln -s "$(pwd)/skills/shiken"   ~/.claude/skills/shiken
```

## 方法 2: copy (固定 snapshot)

```bash
cp -r skills/sadoku skills/kouchiku skills/tansaku skills/shiken ~/.claude/skills/
```

## 方法 3: 別 namespace (waza 等と共存させたい場合)

```bash
mkdir -p ~/.claude/skills-kufuu
ln -s "$(pwd)/skills/"* ~/.claude/skills-kufuu/
# Claude Code 側で skills-kufuu/ を skill ディレクトリとして認識させる設定が別途必要
```

## 既存 waza skill との衝突

waza の `kakunin` / `kentou` / `tsuiseki` 等とは skill 名が違うので衝突しない (kufuu は sadoku / kouchiku / tansaku / shiken)。共存可能。

## 動作確認

```
"レビューして"           → sadoku 通常レビュー が起動
"設計どうする"           → kouchiku 通常検討 が起動
"エラー"                 → tansaku 通常追跡 が起動
"TDDで"                  → shiken が起動
```

trigger 一覧は `docs/workflow.md` (markdown 版) または `docs/workflow.html` (visual 版) を参照。
