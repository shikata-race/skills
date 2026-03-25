# gas-lockservice — GAS LockService + 処理中フラグによる二重実行防止パターン

Google Apps Script で複数ユーザーが同時に同じ操作を行う場合の競合防止。
LockService（行レベルロック）とシート上の「処理中フラグ」を組み合わせる。

## 使い方

```
/gas-lockservice
```

---

## 問題

GASはリクエストごとに独立したスレッドで動く。
`LockService` なしで在庫更新や PDF 生成を行うと、
2人が同時ボタンを押した際に同じレコードを二重処理してしまう。

---

## 基本パターン（単純ロック）

```javascript
function updateInventory(params) {
  const lock = LockService.getScriptLock();
  lock.waitLock(10000); // 最大10秒待機
  try {
    // 在庫読み書き
  } finally {
    lock.releaseLock();
  }
}
```

---

## フラグ付きパターン（バッチ処理・PDF生成など）

「処理中」フラグをシートに書くことで、ロック解放後の二重発行も防ぐ。

```javascript
// Config.gs
const FLAG = {
  UNISSUED:   '未発行',
  PROCESSING: '処理中',
  ISSUED:     '発行済',
};

// 発行ロジック（手動ボタン用）
function issueOne(id) {
  const lock = LockService.getScriptLock();
  lock.waitLock(10000);
  try {
    // ① フラグ確認（ロック内で読む）
    const row = getRowById(id);
    if (row.フラグ !== FLAG.UNISSUED) {
      return { error: 'すでに処理済みです: ' + row.フラグ };
    }
    // ② 処理中マーク（クラッシュしても「処理中」で止まる→再処理可能）
    updateFlagTo(id, FLAG.PROCESSING);
  } finally {
    lock.releaseLock();
  }

  // ③ ロック外で時間のかかる処理（PDF生成など）
  const result = issueCore(id);

  // ④ 完了マーク
  updateFlagTo(id, FLAG.ISSUED);
  return result;
}

// バッチ処理（autoGeneratePdf など）
function autoGeneratePdf() {
  const lock = LockService.getScriptLock();
  lock.waitLock(10000);
  let targets;
  try {
    // ① 未発行レコードを全取得
    targets = getUnissuedRows();
    // ② まとめて処理中マーク（ロック内でやることで二重取得防止）
    targets.forEach(row => updateFlagTo(row.ID, FLAG.PROCESSING));
  } finally {
    lock.releaseLock();
  }

  // ③ ロック外でPDF生成
  targets.forEach(row => issueCore(row.ID));
}
```

---

## core 関数分離のパターン

手動発行とバッチ発行で処理を共有する場合、
**ロック管理は呼び出し元、コア処理は別関数** に分離する。
`issueCore()` 内でさらにロックを取ると二重ロックになるので注意。

```javascript
// NG: issueCore の中でロックを取ると、autoGeneratePdf から呼ぶと二重ロック
function issueCoreNG(id) {
  const lock = LockService.getScriptLock(); // ← NG
  lock.waitLock(10000);
  ...
}

// OK: ロックは呼び出し元だけ
function issueCore(id) {
  const pdfResult = generatePdf(id); // ロックなし
  savePdfFileId(id, pdfResult.fileId);
  return pdfResult;
}
```

---

## タイムアウト設計

| 用途 | 推奨 waitLock |
|---|---|
| 在庫更新（軽量） | 5000ms |
| フラグ読み書き（処理振り分け） | 10000ms |
| バッチ処理の振り分けロック | 30000ms |

PDF生成など時間のかかる処理は **ロックの外** で行い、
ロック内は「フラグの確認と更新」だけに絞る。

---

## よくある失敗

| 失敗 | 原因 | 対策 |
|---|---|---|
| 二重発行が起きる | `waitLock` 後にフラグを確認していない | ロック取得直後にフラグを再確認 |
| デッドロック | core関数がロックを二重取得 | ロック管理は呼び出し元に一本化 |
| 処理中のまま止まる | PDF生成中に例外 → ISSUED にならない | `issueCore` を try-catch してエラー時は UNISSUED に戻す or 運用で「処理中」を手動リセット |
| 10秒でタイムアウト | 同時リクエストが多く待ち行列が詰まった | バッチ処理はトリガー経由、手動操作と分離 |

---

## 学習ログ

<!-- /skill-update で追記される -->
### 2026-03-25
- フラグを「未発行→処理中→発行済」の3状態にすると、クラッシュ時に「処理中」で止まるため調査しやすい（2状態だと成功か失敗か判断できない）
- ロック内は「フラグ確認 + 処理中マーク」だけ。PDF生成などの重い処理はロック外に出す
- 手動発行とバッチ発行でコアロジックを共有するときは「core関数はロック不要」として設計する
