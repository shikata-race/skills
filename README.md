# Claude Code Skills Repository

Claude Code のスラッシュコマンド（スキル）を管理するリポジトリ。
使うたびに学習ログが追記され、スキルが進化していく。

## 仕組み

```
GitHub (shikata-race/skills)
        ↕  /skill-sync
~/.claude/commands/   ← Claude Code が読む場所
```

1. `/skill-sync` — GitHub から最新スキルを pull して `~/.claude/commands/` に反映
2. `/skill-update` — スキルに学習内容を追記して GitHub に push
3. 個別スキル（例: `/gas-parallel`）— 実際の作業を実行

## クイックスタート

```bash
# 初回セットアップ
git clone https://github.com/shikata-race/skills.git ~/.claude/skills-repo
cp ~/.claude/skills-repo/*.md ~/.claude/commands/
# ただし README.md / REGISTRY.md はコピー不要
```

以降は `/skill-sync` で自動同期。

## スキル一覧

→ [REGISTRY.md](./REGISTRY.md) を参照

## スキルファイルのフォーマット

各スキルファイルは以下の構造を持つ：

```markdown
# skill-name — 説明

概要文

## 使い方
## 実行手順
## 注意事項

---

## 学習ログ

<!-- skill-update によって自動追記される -->
### YYYY-MM-DD
- 学んだこと
- うまくいったパターン
- 失敗と改善案
```
