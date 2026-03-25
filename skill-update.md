# skill-update — スキル学習・更新

使用後の学びをスキルファイルの `## 学習ログ` セクションに追記し、
GitHub にプッシュしてスキルを進化させるスキル。

## 使い方

```
/skill-update [skill-name]
```

例：`/skill-update gas-parallel`

引数省略時は直前に使ったスキルを対象とする（ユーザーに確認する）。

---

## 実行手順

### STEP 1: 対象スキルと学習内容の確認

ユーザーに以下を質問する：

1. **対象スキル**: どのスキルを更新するか（省略時は直前のスキル）
2. **学んだこと**: 今回うまくいったパターン・工夫・発見
3. **改善点**: 失敗したこと・次回こうすべき点
4. **バージョン更新**: パッチ（バグ修正）/ マイナー（機能追加）/ メジャー（大幅変更）

### STEP 2: スキルファイルを読む

```
~/.claude/skills-repo/<skill-name>.md
```

を Read ツールで読み、現在の内容と `## 学習ログ` セクションの位置を把握する。

### STEP 3: 学習ログを追記

`## 学習ログ` セクションの末尾（または `<!-- ... -->` コメントの直後）に追記：

```markdown
### YYYY-MM-DD
**コンテキスト**: [どんな作業で使ったか]
**うまくいったこと**:
- ...
**改善点・注意**:
- ...
**次回への提案**:
- ...
```

Edit ツールで `~/.claude/skills-repo/<skill-name>.md` を更新する。

### STEP 4: REGISTRY.md の最終更新日を更新

`~/.claude/skills-repo/REGISTRY.md` の対象スキル行の「最終更新」列を今日の日付に変更する。

### STEP 5: GitHub にプッシュ

```bash
cd ~/.claude/skills-repo
git add <skill-name>.md REGISTRY.md
git commit -m "feat(<skill-name>): 学習ログ追加 YYYY-MM-DD"
git push origin main
```

### STEP 6: `~/.claude/commands/` にも反映

```bash
cp ~/.claude/skills-repo/<skill-name>.md ~/.claude/commands/<skill-name>.md
```

### STEP 7: 完了報告

更新内容のサマリーをユーザーに表示する。

---

## スキル自体を修正する場合

学習ログの追記だけでなく、スキルの手順・ルール自体を改善する場合：

1. STEP 2でファイルを読む
2. 改善箇所を Edit ツールで直接修正
3. STEP 5でプッシュ（コミットメッセージは `fix:` または `feat:` を使う）

---

## 注意事項

- git push には GitHub の認証が必要（SSH鍵 or Personal Access Token）
- push 失敗時はローカルのみ更新してユーザーに手動pushを促す
- スキルファイルの構造（セクション名）は変えないこと

---

## 学習ログ

<!-- skill-update によって自動追記される -->
