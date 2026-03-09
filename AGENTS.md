# スキル追加の規約

## ディレクトリ構造

```
my-skills/
├── .claude-plugin/
│   ├── plugin.json          # プラグインマニフェスト
│   └── marketplace.json     # マーケットプレース定義
├── skills/
│   ├── git-rewrite/
│   │   └── SKILL.md
│   └── <new-skill>/
│       ├── SKILL.md          # 必須: スキル定義（フロントマター + 本文）
│       └── ...               # 任意: 補助ファイル（スクリプト、リファレンス等）
└── AGENTS.md
```

## 新しいスキルの追加手順

1. skill-creatorスキルを使ってよしなにやる
2. `.claude-plugin/marketplace.json` の `plugins[0].skills` 配列にパスを追加する:

```json
"skills": ["./skills/git-rewrite", "./skills/<new-skill>"]
```
