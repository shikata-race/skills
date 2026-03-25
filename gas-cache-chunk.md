# gas-cache-chunk — GAS CacheService 100KB制限を回避するチャンク分割パターン

GASのCacheServiceは1エントリ最大100KB。大きなデータをキャッシュする際はチャンク分割が必要。

## 使い方

```
/gas-cache-chunk
```

カレントディレクトリの `Code.js` (または任意の `.js` ファイル) に下記ヘルパーを追加し、既存の `cache.put()` / `cache.get()` を置き換える。

---

## 実装パターン

### ヘルパー関数（Code.js の先頭付近に追加）

```js
var CACHE_CHUNK_SIZE_ = 90000; // 90KB（100KB上限に余裕を持たせる）

function cachePut_(cache, key, value, ttl) {
  var chunks = [];
  for (var i = 0; i < value.length; i += CACHE_CHUNK_SIZE_) {
    chunks.push(value.slice(i, i + CACHE_CHUNK_SIZE_));
  }
  var meta = JSON.stringify({ n: chunks.length });
  var obj = {};
  obj[key + '__meta'] = meta;
  chunks.forEach(function(c, i) { obj[key + '__' + i] = c; });
  cache.putAll(obj, ttl);
}

function cacheGet_(cache, key) {
  var meta = cache.get(key + '__meta');
  if (!meta) return null;
  var n = JSON.parse(meta).n;
  var keys = [];
  for (var i = 0; i < n; i++) keys.push(key + '__' + i);
  var parts = cache.getAll(keys);
  var chunks = [];
  for (var i = 0; i < n; i++) {
    if (!parts[key + '__' + i]) return null; // チャンク欠損 → キャッシュミス扱い
    chunks.push(parts[key + '__' + i]);
  }
  return chunks.join('');
}
```

### キャッシュの書き込み

```js
// 従来: cache.put('my_key', JSON.stringify(data), 300);
// 変更後:
cachePut_(cache, 'my_key', JSON.stringify(data), 300);
```

### キャッシュの読み込み

```js
// 従来: var cached = cache.get('my_key');
// 変更後:
var cached = cacheGet_(cache, 'my_key');
if (cached) {
  return JSON.parse(cached);
}
```

### キャッシュの削除（更新ボタン等で強制リフレッシュ）

```js
// チャンク数が不定なのでキー名のパターンで削除できない。
// メタを読んで全チャンクキーを組み立ててから removeAll する。
function clearCacheKey_(cache, key) {
  var meta = cache.get(key + '__meta');
  if (!meta) return;
  var n = JSON.parse(meta).n;
  var keys = [key + '__meta'];
  for (var i = 0; i < n; i++) keys.push(key + '__' + i);
  cache.removeAll(keys);
}
```

---

## 日付オブジェクトのシリアライズ注意

`Date` オブジェクトは `JSON.stringify` すると `"2024-03-15T00:00:00.000Z"` になる。
これをそのまま `fmtDate_()` に渡すと、関数が期待する `Date` 型でないため `YYYY/MM/DD` に変換されない場合がある。

**安全なパターン**: キャッシュに書き込む前に `fmtDate_(c)` で文字列化する。

```js
// NG: c instanceof Date ? c.toISOString() : c
// OK: fmtDate_(c)  → "2024/03/15" で統一
rows.map(function(r) {
  return r.map(function(c) { return c instanceof Date ? fmtDate_(c) : c; });
})
```

読み出し側の `toDate_()` は `YYYY/MM/DD` を受け取れるようにスラッシュ→ハイフン変換を入れる：

```js
function toDate_(v) {
  if (!v) return null;
  return new Date(String(v).replace(/\//g, '-'));
}
```

---

## よくある失敗

| 失敗 | 原因 | 対策 |
|---|---|---|
| `引数が大きすぎます: value` | `cache.put()` に100KB超のデータを渡した | `cachePut_()` に置き換え |
| キャッシュから読み出したのにガントチャートが表示されない | 日付を `toISOString()` でキャッシュしたため `fmtDate_()` が正しく処理できなかった | `fmtDate_(c)` でシリアライズ |
| チャンク欠損で `null` になる | TTLがチャンク数分書き終わる前に切れた（稀） | TTLを+10秒程度マージン追加 |

---

## 学習ログ

<!-- /skill-update で追記される -->
### 2026-03-25
- `putAll()` で全チャンクを1回のAPI呼び出しにまとめると速い
- 削除時はメタキーから逆算してチャンクキーを組み立てる（`removeAll`）
- 日付のシリアライズは `fmtDate_()` 統一が最も安全（`toISOString()` は避ける）
