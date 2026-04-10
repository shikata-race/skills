# bifuku-kosoku — 備福運送 拘束時間管理ロボットの保守スキル

BizRobo→VBA/PowerShell置き換えプロジェクトの保守・修正を行う。
リポジトリ: `shikata-race/bifuku-kosoku-jikan`

## コンテキスト

- **対象:** 備福運送株式会社の拘束時間管理システム
- **元システム:** BizRobo RPA (Robot 3: 拘束時間表, Robot 4: 作業内容転記)
- **置き換え先:** VBA (`mod_ConstraintTime.bas`) / PowerShell (`Process-ConstraintTime.ps1`)
- **CSVエンコーディング:** Shift-JIS (CP932)
- **担当者:** 佐子さん（備福運送）、四方さん（開発側）

## ファイル構成

```
src/
  mod_ConstraintTime.bas    … VBAモジュール（Excelに組み込み）
  Process-ConstraintTime.ps1 … PowerShellスクリプト（BizRoboから呼び出し）
  test_logic.py              … テスト検証用Python
組み込み手順書.html            … BizRoboへの組み込み手順（HTML）
RPA拘束時間管理表/             … テスト用CSVデータ
```

## 修正済み不具合（4件）

1. **行程重複** — 同日2CSV処理時の作業内容重複。全CSVをマージして日単位でルート構築。
2. **運転時間未集計** — 休息連続時の走行時間欠落。着地作業に関わらず走行時間を常に加算。
3. **深夜時間未集計** — 移動区間+作業区間の全拘束時間で22:00-05:00帯を計算。休憩・休息の作業部分は除外。
4. **月跨ぎ不具合** — 日単位で月を判定し、正しい月のシートに振り分け。

## ビジネスルール

- **日の切り替え:** 着日付の変化で判定
- **深夜時間:** 移動区間[前行発時刻→当行着時刻] + 作業区間[着時刻→発時刻]（休憩・休息除外）の22:00-05:00重なり
- **出庫/入庫:** 初日=出庫行の発時刻、中間日=前日最終発時刻、最終日=入庫行の着時刻
- **月跨ぎ:** 各日のデータを`日付.月`で振り分け、該当月シートに書き込み

## 修正作業時の手順

1. `src/test_logic.py` でテスト実行（Python3、Linuxで動作確認可）
2. VBA (`mod_ConstraintTime.bas`) と PowerShell (`Process-ConstraintTime.ps1`) の**両方**を同じロジックで修正
3. テスト再実行して全PASS確認
4. `組み込み手順書.html` に影響があれば更新

## 学習ログ

<!-- /skill-update で追記される -->
- 2026-04-10: 初版作成。4件の不具合修正をVBA/PowerShellで実装。
