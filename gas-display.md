# gas-display — GAS HtmlService で確実に動くshow/hide パターン

GASのウェブアプリ（HtmlService）はiframe内でレンダリングされる。
`classList.add('hidden')` や `classList.toggle()` は動作しないケースがあるため、
**`style.display` で直接制御する**のが唯一の確実な方法。

## 使い方

```
/gas-display
```

カレントディレクトリの HTML ファイルの show/hide 処理を `style.display` パターンに統一する。

---

## 正しいパターン

### 表示

```js
element.style.display = '';       // デフォルト表示に戻す（block/flex どちらでも動く）
element.style.display = 'block';  // ブロック要素として表示
element.style.display = 'flex';   // Flexbox として表示（明示したいとき）
```

### 非表示

```js
element.style.display = 'none';   // 非表示
```

### NG パターン（GAS iframeでは動かない）

```js
// NG: classList は GAS iframe で挙動が不安定
element.classList.add('hidden');
element.classList.remove('hidden');
element.classList.toggle('active');

// NG: CSS の .hidden { display: none } を前提にした設計
```

---

## ローディング / エラー / コンテンツの典型パターン

```js
// ローディング表示
function showLoading(tabId) {
  var el = document.getElementById(tabId + '-loading');
  if (el) el.style.display = 'block';
  var content = document.getElementById(tabId + '-content');
  if (content) content.style.display = 'none';
  var err = document.getElementById(tabId + '-error');
  if (err) err.style.display = 'none';
}

// ローディング非表示
function hideLoading(tabId) {
  var el = document.getElementById(tabId + '-loading');
  if (el) el.style.display = 'none';
}

// エラー表示
function showError(tabId, msg) {
  hideLoading(tabId);
  var el = document.getElementById(tabId + '-error');
  if (el) {
    el.textContent = 'エラー: ' + msg;
    el.style.display = 'block';
  }
}
```

## タブ切り替えパターン

```js
function switchTab(tabId) {
  // 全タブボタンの active 状態を解除
  document.querySelectorAll('.tab-btn').forEach(function(btn) {
    btn.classList.remove('active'); // ← ボタンの見た目制御は classList でOK
  });
  var activeBtn = document.getElementById('btn-' + tabId);
  if (activeBtn) activeBtn.classList.add('active');

  // パネルの表示切り替えは style.display で
  document.querySelectorAll('.tab-panel').forEach(function(p) {
    p.style.display = 'none';
  });
  var panel = document.getElementById('tab-' + tabId);
  if (panel) panel.style.display = ''; // '' でそのパネルのデフォルト表示に戻す
}
```

> **ポイント**: ボタンの `.active` class（見た目のみ）は `classList` でOK。
> コンテンツ領域の出し入れは必ず `style.display`。

---

## HTML の初期状態

```html
<!-- コンテンツは最初 none で隠す -->
<div id="inv-loading" style="display:none;">...</div>
<div id="inv-error"   style="display:none;">...</div>
<div id="inv-content" style="display:none; flex-direction:column; gap:1rem;">...</div>

<!-- タブパネルの初期状態: 最初のタブだけ表示、残りは none -->
<div id="tab-summary"    class="tab-panel">...</div>
<div id="tab-allocation" class="tab-panel" style="display:none;">...</div>
```

---

## よくある失敗

| 失敗 | 原因 | 対策 |
|---|---|---|
| ボタンを押してもパネルが切り替わらない | `classList.add('hidden')` / `classList.remove('hidden')` を使っている | `style.display` に置き換え |
| ローディングが消えてもコンテンツが出ない | CSS に `.hidden { display:none }` を書いているが GAS が適用しない | インラインスタイルに統一 |
| タブが重なって表示される | 非アクティブパネルを `display:none` にしていない | 全パネルを `display:none` にしてから対象だけ `display:''` |

---

## 学習ログ

<!-- /skill-update で追記される -->
### 2026-03-25
- `classList` の見た目制御（`.active` など）は問題ない。`display` の切り替えだけ `style.display` にする
- Flexbox コンテナは `display:''` に戻すと元の `flex` が効かないことがある → `display:'flex'` を明示するか、CSS側に `display:flex` を書いておく
- 初期状態を HTML の `style="display:none"` で書いておくのが最も安全（CSSファイルに書くより確実）

### 2026-04-08
- **効いたパターン**: タブ切り替えで共有パネルの表示/非表示を制御する場合、`switchTab()` 内で `panel.style.display = (関連タブか) ? '' : 'none'` を入れる。パネルをタブ外に配置しても、無関係なタブで表示されて混乱を招く
- **注意**: flex コンテナの子要素がスクロールしない場合、`overflow-y: auto` の追加ではなく `min-height: 0` を親に指定するのが正解。`overflow-y: auto` を親にも子にも入れると二重スクロールになる
- **コツ**: ヘッダーからフォーム要素を分離してパネル化する場合、そのパネルが複数タブから参照されるかを先に確認する。タブ内に入れると他タブから参照できなくなる
