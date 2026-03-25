# skill-sync — GitHubからスキルを同期する

`shikata-race/skills` リポジトリから最新のスキルをpullして `~/.claude/commands/` に反映する。
複数マシン間でスキルを共有するときにも使う。

## 使い方

```
/skill-sync
```

---

## 実行手順

### STEP 1: ローカルリポジトリの確認

```bash
ls ~/.claude/skills-repo 2>/dev/null && echo "exists" || echo "not cloned"
```

- **存在しない** → STEP 2a（初回クローン）
- **存在する** → STEP 2b（pull更新）

### STEP 2a: 初回クローン

```bash
git clone https://github.com/shikata-race/skills.git ~/.claude/skills-repo
```

### STEP 2b: 最新をpull

```bash
cd ~/.claude/skills-repo && git pull origin main
```

差分があった場合は更新されたスキルファイル名を表示する。

### STEP 3: スキルファイルをコピー

`~/.claude/skills-repo/` 内の `.md` ファイルのうち、`README.md` と `REGISTRY.md` を除いて `~/.claude/commands/` にコピーする：

```bash
mkdir -p ~/.claude/commands
for f in ~/.claude/skills-repo/*.md; do
  name=$(basename "$f")
  if [ "$name" != "README.md" ] && [ "$name" != "REGISTRY.md" ]; then
    cp "$f" ~/.claude/commands/
  fi
done
```

### STEP 4: 結果を報告

同期したスキルの一覧と各スキルの説明（REGISTRY.mdから）を表示する。

---

## 注意

- ローカルで直接 `~/.claude/commands/` を編集した内容は上書きされる
- 編集内容を保持したい場合は先に `/skill-update` でpushしてから同期する

---

## 学習ログ

<!-- /skill-update で追記される -->
