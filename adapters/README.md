# adapters/

このディレクトリは将来 Agent Skills 標準でカバーできない特殊ケースが出たときに備えた予約領域です。**現状は空**で問題ありません。

## なぜ空か

kufuu は [Agent Skills 標準](https://agentskills.io) に準拠しており、以下の方法で 30+ AI tool に配置できます:

```bash
npx skills add github:hayashiii-ghub/kufuu
```

ハーネス独自の変換 / 配置スクリプトは不要です (詳細は root の README を参照)。

## 何が入る予定か

実需要が出た場合のみ追加:

- 標準仕様外の独自 frontmatter 拡張 (Cursor `paths` 等) の生成スクリプト
- 特定ツール固有の install hook
- 標準対応していない自社内 AI tool 向けの変換

「将来必要になるかも」だけで先行追加はしません (引き算原則 / YAGNI)。
