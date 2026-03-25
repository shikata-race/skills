# gas-parallel — GAS並列実装スキル

Google Apps Script プロジェクトの改善タスクを、ファイル競合なしで4並列エージェントで実装し、clasp push → deploy まで自動実行するスキルです。

## 使い方

```
/gas-parallel [plan.mdのパス] [deploymentId]
```

引数省略時はカレントディレクトリの `plan.md` と `.clasp.json` から自動検出します。

---

## 実行手順

以下の手順を順番に実行してください。

### STEP 1: 前提確認

1. `plan.md` を読んで未完了タスクを一覧化する
2. `.clasp.json` を読んで deploymentId を確認する
3. プロジェクト内の全 `.gs` ファイルと `index.html` を glob で列挙する

### STEP 2: タスク選定とファイル割り当て

未完了タスクから最大4件を選び、**ファイル競合が起きないよう**以下のルールで割り当てる：

**競合防止ルール:**
- 同一ファイルを複数エージェントに割り当てない
- `index.html` は1エージェントのみ担当
- `Code.gs` は1エージェントのみ担当（新関数追記のみなら共存可能だが避ける）
- 新規ファイル作成タスクは必ず独立エージェントに割り当てる
- フロントエンド（index.html）とバックエンド（.gs）は別エージェントに分ける

**ファイル分類:**
| ファイル | 用途 |
|---|---|
| `Config.gs` | 定数・設定値 |
| `Auth.gs` | 認証・権限 |
| `Code.gs` | APIエンドポイント |
| `SheetService.gs` | スプレッドシート読み書き |
| `PdfService.gs` | PDF生成・Drive操作 |
| `NotificationService.gs` | メール通知 |
| `DashboardService.gs` | 集計データ |
| `TriggerSetup.gs` | トリガー設定 |
| `DateFilter.gs` | 日付フィルタ |
| `index.html` | フロントエンドSPA |
| `appsscript.json` | マニフェスト・スコープ |

### STEP 3: 4並列エージェント起動

選定したタスクをエージェントプロンプトに変換し、**1つのメッセージで4つのAgentツール呼び出しを同時実行**する。

各エージェントへの指示に必ず含めること：
- 担当ファイルのパスと「読んでから修正」の指示
- 具体的な実装仕様（曖昧にしない）
- 既存コードとの整合性（定数名・関数名を正確に指定）
- `Edit` ツールで既存ファイルを修正、`Write` ツールで新規作成

### STEP 4: 完了待機とマージ確認

全エージェント完了後：
1. 各エージェントの出力サマリーを確認
2. ファイルが正しく修正されているか Read ツールで抽出確認
3. 競合・矛盾がないか確認

### STEP 5: push → deploy

```bash
clasp push --force
clasp deploy --deploymentId <deploymentId>
```

### STEP 6: plan.md の完了マーク更新

実装したタスクを plan.md で完了済みとマークする（チェックマーク追加 or セクション移動）。

---

## エージェント割り当てパターン例

```
タスクA (新規.gs作成)  → Agent 1: 新規ファイル
タスクB (Code.gs追記)  → Agent 2: Code.gs末尾に新関数
タスクC (PdfService改修) → Agent 3: PdfService.gs
タスクD (UI追加)       → Agent 4: index.html
```

複数タスクを1エージェントに集約する場合は同一ファイル内のみ。

---

## 注意事項

- `appsscript.json` のスコープ変更は毎回確認（新機能追加時は必要なスコープを追加）
- Drive Advanced Service を使う場合は `dependencies` への追記も必要
- `clasp push` 失敗時は `appsscript.json` のフォーマットエラーを疑う
- GASの `const` はスクリプトスコープなので複数ファイルで同名定数を定義しない

---

## 学習ログ

<!-- skill-update によって自動追記される -->
### 2026-03-25
**コンテキスト**: 向井鉄鋼 現品票発行システム（GAS Web アプリ）の一括改善
**うまくいったこと**:
- ファイル割り当て: 新規.gs / Code.gs / PdfService.gs / index.html の4分割が最もコンフリクトしにくい
- `appsscript.json` は独立エージェントに任せると他と競合しない
- `autoGeneratePdf` と `_issueOne` でLockServiceを二重取得しないよう `_issueOneCore` に共通化
**改善点・注意**:
- Code.gs はエンドポイント追加が重なりやすい → 1エージェントに集約すること
- SheetService.gs も複数エージェントが触りやすい → キャッシュ追加とフラグ更新は同じエージェントに
- index.html はスクロール量が多く、エージェントが変更箇所を誤認しやすい → 挿入位置を行番号で明示する
**次回への提案**:
- plan.md に `[完了]` マークを付けるステップをSTEP 6として明示化
