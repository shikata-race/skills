# skill-sync — スキル同期

GitHub の `shikata-race/skills` リポジトリから最新スキルを pull して
`~/.claude/commands/` に反映するスキル。

## 使い方

```
/skill-sync
```

---

## 実行手順

以下を順番に実行してください。

### STEP 1: リポジトリの状態確認

```bash
ls ~/.claude/skills-repo 2>/dev/null && echo "exists" || echo "not cloned"
```

- 存在しない場合 → STEP 2a（初回クローン）
- 存在する場合 → STEP 2b（pull更新）

### STEP 2a: 初回クローン

```bash
git clone https://github.com/shikata-race/skills.git ~/.claude/skills-repo
```

### STEP 2b: 最新をpull

```bash
cd ~/.claude/skills-repo && git pull origin main
```

### STEP 3: スキルファイルをコマンドディレクトリにコピー

```bash
mkdir -p ~/.claude/commands
```

`~/.claude/skills-repo/` 内の `.md` ファイルのうち、
`README.md` と `REGISTRY.md` を除いた全ファイルを `~/.claude/commands/` にコピーする。

```bash
for f in ~/.claude/skills-repo/*.md; do
  name=$(basename "$f")
  if [ "$name" != "README.md" ] && [ "$name" != "REGISTRY.md" ]; then
    cp "$f" ~/.claude/commands/
  fi
done
```

### STEP 4: 同期結果を報告

コピーしたスキルファイルの一覧と、REGISTRY.md の内容を表示してユーザーに報告する。

---

## 注意事項

- `~/.claude/commands/` のファイルはこのコマンド実行後すぐ有効になる
- ローカルで手動編集したスキルは上書きされるので注意
- 編集内容を保持したい場合は先に `/skill-update` でpushすること

---

## 学習ログ

<!-- skill-update によって自動追記される -->
