# gas-pdf-export — GAS Sheets→PDF エクスポートのレイアウト調整パターン

Google Apps Script で Spreadsheet を PDF に変換して Drive に保存する際の、
A4横・1ページ収め・中央配置のための URL パラメータとレイアウト設計パターン。

## 使い方

```
/gas-pdf-export
```

---

## PDF エクスポート URL パラメータ（A4横・1ページ収め）

```javascript
function exportAsPdf(ssId) {
  const url = `https://docs.google.com/spreadsheets/d/${ssId}/export`
    + '?format=pdf'
    + '&size=A4'
    + '&portrait=false'       // 横向き（landscape）
    + '&fitw=true'            // 印刷幅に合わせてスケール
    + '&sheetnames=false'
    + '&printtitle=false'
    + '&pagenumbers=false'
    + '&gridlines=false'
    + '&fzr=false'
    + '&top_margin=0.25'      // 余白を0.5→0.25inに縮める（1ページ収めに重要）
    + '&bottom_margin=0.25'
    + '&left_margin=0.25'
    + '&right_margin=0.25'
    + '&gid=0';               // シートID

  const resp = UrlFetchApp.fetch(url, {
    headers: { Authorization: 'Bearer ' + ScriptApp.getOAuthToken() },
    muteHttpExceptions: true,
  });
  if (resp.getResponseCode() !== 200) throw new Error('PDF出力失敗: ' + resp.getResponseCode());
  return resp.getBlob().setContentType('application/pdf');
}
```

---

## 1ページ収め：行高の合計を「印刷可能高さ」以内に

A4横（297mm）でマージン0.25in（6.35mm）上下の場合:
- 印刷可能高さ ≈ 297 - 6.35×2 = 284mm ≈ **720px**（GASの1px≈0.375mm換算）

```javascript
// NG: 余白0.5inデフォルト → 印刷可能高さ ≈ 624px 相当
// OK: 余白0.25inに変更 → 印刷可能高さ ≈ 720px 相当

// 行高の合計がこの値を超えると2ページになる
// 実際には5〜10px余裕を持たせる（合計710px以内を目安に）
```

---

## 表を中央に配置：列幅の合計 = 印刷可能幅

A4横でマージン0.25in左右の場合:
- 印刷可能幅 ≈ 210mm×√2（A4横）- 6.35mm×2 ≈ **1060〜1070px**

`fitw=true` はスケールダウンのみで、スケールアップはしない。
→ **列幅の合計を印刷可能幅ぴったりに設定する**のが唯一の確実な中央配置方法。

```javascript
// 例: A4横・8列・合計1065px
sheet.setColumnWidth(1, 185); // 工程名（広め）
sheet.setColumnWidth(2, 160); // 工程名（続き）
sheet.setColumnWidth(3, 130); // 加工日
sheet.setColumnWidth(4, 130); // 加工日（続き）
sheet.setColumnWidth(5, 130); // 作業者
sheet.setColumnWidth(6, 130); // 作業者（続き）
sheet.setColumnWidth(7, 110); // 数値
sheet.setColumnWidth(8,  90); // 押印欄（狭め）
// 合計: 1065px ≈ A4横印刷幅 → 左右ぴったり = 自然に中央
```

---

## 一時シートの確実な削除（try-finally）

```javascript
function generatePdf(data) {
  const tmpSS = SpreadsheetApp.create('tmp_' + data.id);
  try {
    _buildLayout(tmpSS.getActiveSheet(), data);
    SpreadsheetApp.flush();
    const blob = exportAsPdf(tmpSS.getId());
    const file = folder.createFile(blob);
    return file.getId();
  } finally {
    // 成功・失敗どちらでも必ず削除
    try {
      DriveApp.getFileById(tmpSS.getId()).setTrashed(true);
      Drive.Files.remove(tmpSS.getId()); // Drive Advanced Service（任意）
    } catch (e) {
      Logger.log('一時シート削除スキップ: ' + e.message);
    }
  }
}
```

---

## よくある失敗

| 失敗 | 原因 | 対策 |
|---|---|---|
| 2ページになる | デフォルト余白0.5inで印刷可能高さが足りない | `top/bottom_margin=0.25` で余白を縮める |
| 表が左寄りになる | 列幅合計 < 印刷可能幅（`fitw`はスケールアップしない） | 列幅合計を印刷可能幅ぴったりに合わせる |
| 一時シートがゴミ箱に残る | 例外発生時に削除処理が走らない | try-finally で必ず削除 |
| `portrait=false` なのに縦向きになる | `size=A4` の前に `portrait` を書くと無視されることがある | 記載順: `size` → `portrait` → その他 |

---

## 学習ログ

<!-- /skill-update で追記される -->
### 2026-03-25
- 余白0.5in（デフォルト）→0.25inにするだけで使える高さが約100px増える。これで2ページ問題の多くが解決
- `fitw=true` はスケールダウンのみ。列幅の合計を印刷可能幅に揃えるのが中央配置の唯一の確実手段
- A4横の印刷可能幅は ~1060〜1070px（マージン次第）。1065pxを基準にするとズレが少ない
- 一時シートの `try-finally` 削除パターンは必須。PDF生成が失敗しても毎回ゴミが残らなくなる
