# gas-gotchas — GAS・スプレッドシート開発でハマりやすい落とし穴

Google Apps Script とスプレッドシート操作で実際にバグになった挙動と回避策をまとめる。
「なぜ壊れるか」の理由を一言添えることで、似た状況での判断に使えるようにする。

## スプレッドシート書き込みの罠

### boolean をそのまま渡すと TRUE/FALSE 文字列になる
```javascript
// ✕ boolean を setValue/appendRow に渡す
row[COL.FLAG - 1] = false;  // → スプレッドシートに "FALSE" が入る

// ◯ 文字列に変換してから渡す
row[COL.FLAG - 1] = 'NO';   // または '' など意味のある文字列
```
**なぜ壊れるか**: GAS は boolean を文字列変換せずセルに書き込むため、後で `=== false` で比較しても一致しない。

### setValues() に Invalid Date が1つあると**その行全体**が失敗する
```javascript
// ✕ 空文字や不正な日付文字列をそのまま Date に変換して渡す
dValues[i][DATE_COL] = new Date('');  // Invalid Date → 行全体がエラー

// ◯ パース前に空チェックして元の値を維持するヘルパーを使う
function parseUsageDate_(str, fallback) {
    if (!str) return fallback;
    var d = new Date(str);
    return isNaN(d.getTime()) ? fallback : d;
}
```
**なぜ壊れるか**: `setValues()` は配列全体を一括送信するため、1セルでも Invalid Date があるとその行が書き込まれない（しかもエラーメッセージが分かりにくい）。

### 行削除は必ず後ろからループ
```javascript
// ✕ 前からループすると削除後に行番号がズレる
for (var i = 0; i < data.length; i++) { sheet.deleteRow(i + 2); }

// ◯ 後ろからループ
for (var i = data.length - 1; i >= 0; i--) { sheet.deleteRow(i + 2); }
```

---

## キャッシュ・スナップショットの罠

### 申請者操作でもキャッシュ無効化が必要
集計レポートをキャッシュ（スナップショット）している場合、**申請の新規作成・更新・削除**でも無効化を呼ばないとレポートが古い値を返し続ける。
管理者側の承認・却下では無効化しているのに、申請者側の操作で呼び忘れるパターンが多い。

```javascript
// api_submitExpense / api_updateApplication / api_deleteApplication
// それぞれの書き込み後に必ず呼ぶ
invalidateSnapshots_(appDate);
```

---

## google.script.run の罠

### withFailureHandler を省略すると失敗が無音になる
```javascript
// ✕ 失敗ハンドラなし → エラーが起きても画面に何も出ない
google.script.run
    .withSuccessHandler(initData)
    .getInitData();

// ◯ 必ず両方書く
google.script.run
    .withSuccessHandler(initData)
    .withFailureHandler(e => showToast('エラー: ' + e.message, 'error'))
    .getInitData();
```
**特に危険なケース**: アプリ起動時の初期データ取得。失敗しても空白画面になるだけで原因が分からない。

---

## HtmlService IFRAME mode のフレームワークバグ

### 文字列リテラル内の `//` (連続2スラッシュ) で SyntaxError
```javascript
// ✕ 文字列内の // が GAS framework を破壊
img.src = 'https://lh3.googleusercontent.com/d/' + id + '=w100';
// → "Failed to execute 'write' on 'Document': Invalid or unexpected token"
//    at lt (mae_html_user_bin_i18n_mae_html_user__ja.js:318)

// ◯ 配列 + .join('/') で組み立てて回避
img.src = ['https:', '', 'lh3.googleusercontent.com', 'd', id + '=w100'].join('/');

// ◯ または + で分割
const url = 'https:/' + '/lh3.googleusercontent.com/d/' + id + '=w100';
```
**なぜ壊れるか**: GAS の IFRAME mode では `mae_html_user.js` がユーザーHTMLを子iframeに注入する。そのチャンク処理の中で**文字列リテラル内の `//` を JS のコメント開始と誤認**し、以降のコードが文字列の途中扱いになって壊れる。
- Node の V8 では parse 通るのに **Chrome の V8 (GAS経由) では失敗** するのが特徴
- DevTools には `userCodeAppPanel:NNNN Uncaught SyntaxError: Invalid or unexpected token` と出る (NNNNは問題の行番号)
- 単一スラッシュ複数 (`a/b/c`) は OK。**連続2文字 `//` だけ**がトリガー
- 該当箇所は `grep -nE "['\"\`][^'\"\`]*//[^'\"\`]*['\"\`]"` で全列挙できる

---

## フロントエンドDOM操作の罠

### innerHTML += はイベントリスナーを全破壊する
```javascript
// ✕ += で追記するとそれまでの全ノードが再生成され、リスナーが消える
list.innerHTML += '<div>new item</div>';

// ◯ appendChild で追加する
const div = document.createElement('div');
div.textContent = 'new item';
list.appendChild(div);
```
**なぜ壊れるか**: `innerHTML +=` は内部的に「全削除→全再挿入」を行うため、既存ノードへのイベントリスナーが全て失われる。

### amount=0 を falsy 扱いすると入力欄が空になる
```javascript
// ✕ 0 が falsy なので空文字に化ける
input.value = item.amount || '';  // → amount=0 のとき空欄になる

// ◯ null/undefined チェックを明示する
input.value = (item.amount !== undefined && item.amount !== null) ? item.amount : '';
```

---

## 学習ログ

### 2026-03-25
- **注意**: `setValues()` の Invalid Date は「その行だけ失敗」ではなく「その行全体が書き込まれない」。エラーメッセージも不親切なのでデバッグに時間がかかる
- **注意**: キャッシュ無効化の呼び忘れは「管理者操作には入れたのに申請者操作には入れなかった」という非対称ミスが多い。書き込みを伴う全操作にチェックリストとして確認する
- **注意**: `withFailureHandler` の省略は起動時の初期データ取得で特に危険。ユーザーには空白画面だけが表示され原因不明になる
- **効いたパターン**: `innerHTML +=` を使っている箇所を `appendChild` に置き換えると、カートやリスト系のUIでイベント消失バグが一掃できる
- **コツ**: boolean フラグをスプレッドシートに書き込む場合は、定数定義の時点で文字列値（`'YES'`/`'NO'` 等）にしておくと混入を防げる

### 2026-04-08
- **注意**: UIフィールドを `display:none` で隠す際、そのフィールドを参照するバリデーション・保存処理・初期値設定を上流から下流まで全て追跡すること。隠したフィールドが空になりバリデーション失敗→ユーザーが修正不能、という CRITICAL バグになる
- **注意**: 非表示フィールドの値を hidden input に変換する場合、元が `<select>` だったときは `.value` の挙動が変わらないか確認。hidden input は `.value` で文字列を返すが、select は選択された option の value を返す
- **効いたパターン**: 実装直後に「敵対的セルフレビュー」を行う。自分の変更を攻撃者目線で精査すると、CRITICAL 級のバグ（バリデーション崩壊・データ消失）を本番前に潰せる
- **コツ**: カラムを「非表示にするが内部保持」する場合、デフォルト値は元のまま残すこと。空文字にすると集計処理で「空カテゴリ」が発生する

### 2026-04-13
- **注意**: `getValues()` は「セルが日付書式でも値が数値」の場合 Date オブジェクトではなく**そのまま数値（Excelシリアル値）を返す**。Excel由来のxlsxを取り込んだシートで頻発。`val instanceof Date` だけで判定するとシリアル値がスルーされてUIに「45738」などの謎数字が出る
- **対策**: シリアル値→Date変換は `new Date((serial - 25569) * 86400000)` で UTC基準にして、表示は `d.getUTCFullYear()`/`getUTCMonth()+1`/`getUTCDate()` で取り出すとタイムゾーンずれを回避できる。20000〜80000の範囲外は日付として扱わない（誤変換防止）
- **効いたパターン**: サーバ側共通ヘルパー `_formatCellValue(val, header, code2name)` を作ってヘッダー名で分岐（`/日|年月|期限|いつ/.test(h)` で日付列判定、`/担当|営業マン/.test(h)` で担当コード→氏名変換）。表示整形を1箇所に集約すると全画面で一貫して直せる
- **効いたパターン**: 重いテーブル取得は「2段階ロード」で体感速度を改善できる。Phase1: `page=1, pageSize=50` で先頭だけ取得→即描画、Phase2: バックグラウンドで全件取得→キャッシュ置換。ソート／検索／全文フィルタは全件データが必要なので「準備できるまで待つ」ヘルパー（`_ensureFullAlertData`）を経由させる
- **注意**: キャッシュキーをバージョニング（`alert_` → `alert_v2_`）して古い形式のJSONが混ざらないようにする。フォーマット変更時の必須作業。古いキーの削除箇所も全部grepして更新

### 2026-04-15
- **CRITICAL**: GAS HtmlService IFRAME mode で **文字列リテラル内の `//` (連続スラッシュ)** が SyntaxError を起こす。`'https://...'` のような URL リテラルが全滅する。Node ではparse通るのに Chrome ではダメ、という非対称が特徴的な症状
- **対策**: URL構築は `[parts].join('/')` か `'https:/' + '/...'` で分割する。コードベースから一括検出は `grep -nE "['\"\`][^'\"\`]*//[^'\"\`]*['\"\`]"`
- **デバッグの教訓**: bisection でファイルを単純truncateすると関数の途中で切れて常に "Unexpected end of input" になり原因が見つからない。**問題行を別の安全な内容に置換**して有無を確認する方が早い
- **コツ**: エラー位置 `userCodeAppPanel:NNNN` の NNNN を信じて、**同じ行を変えても同じ番号で出続ける**ようなら、その行の文字列リテラル中身を疑う

### 2026-04-09
- **注意**: Web NFC API は iOS 完全非対応（Safari・Chrome 含む全ブラウザ）。現場の出席認証にはQRコード方式を選ぶこと。html5-qrcode ライブラリなら iOS Safari 14.3+ / Android Chrome 両対応
- **注意**: YouTube IFrame API の `postMessage` は GAS の `HtmlService` iframe 内で制限される。再生進捗の正確なトラッキングが必要な場合はタイマーベース（setInterval + 動画duration）で代替する
- **効いたパターン**: 公開アプリ（ANYONE_ANONYMOUS デプロイ）で社外メンバーにもアクセス可能にしつつ、管理者機能だけ PIN 認証（Settings シート保存 + sessionStorage）で保護する構成。Googleアカウント不要で現場作業員のハードルが下がる
- **効いたパターン**: 1台の端末で複数人の出席を記録する場合、sessionId（UUID）で同時視聴者をグループ化し、「もう1人追加」ボタンで Step 2（本人確認）に戻るループUIが直感的。動画は再視聴不要にすることで時間節約
- **コツ**: スプレッドシートの boolean 列を `getValues()` で読む際、`true` / `'TRUE'` / `'true'` すべてのパターンが来る。`=== true` だけでは漏れるので `a === true || a === 'TRUE' || a === 'true'` で判定する
