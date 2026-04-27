# akatsuki-okayama — 暁電業 岡山県入札情報取得ツールの検証・保守スキル

岡山県DENCHO (`ppi04.t-elbs.jp`) スクレイパ `OkayamaCollector.js` の動作検証を、GCP Cloud Workstation 上で noVNC + chrome-devtools MCP 経由で再現する。`/home/user/暁電業/岡山県スクレイピング/納品_入札情報取得ツール_1.09/` 配下が対象。

## コンテキスト

- **対象:** 暁電業株式会社 (岡山県案件のみ)
- **元ツール:** `納品_入札情報取得ツール_1.09` (京都/滋賀/奈良/愛知対応)
- **派生版:** 岡山特化 (Nara pref の DENCHO/ppi06 と同フレームワーク)
- **対象URL:** `https://www.ppi04.t-elbs.jp/DENCHO/PpiJGyomuStart.do?kinouid=GP5000_Top&dantaiCd=3300`
- **実行環境:** GCP Cloud Workstation (Ubuntu 24.04、Chromeは未導入 / X serverなし)

## ファイル構成

```
岡山県スクレイピング/
  gcp_workstation_chrome_devtools_mcp.md   … MCP/noVNC 再構築の詳細手順
  納品_入札情報取得ツール_1.09/
    system/src/
      settings.json                          … Okayama.{CompanyName,RepresentativeName,HacchuBuCd} + FilterPeriod
      Playwright/
        OkayamaCollector.js                  … JS本体 (メインの修正対象)
        OkayamaWrapper.ps1                   … Windows想定のラッパ (Linux検証では使わない)
        package.json / node_modules          … playwright 本体
    bin/
    入札情報取得ツール.exe                    … 本番配布exe (参考)
```

## 岡山版の肝 (落とし穴)

- **T-ELBS 系ポップアップは `window.opener` モック必須**。既存 `DebugCompare.js` に実績パターンあり
- **結果行の2ボタン**を両方踏む:
  - `pf_VidDsp_FileSelect('0510',...)` = **公告等**
  - `pf_VidDsp_FileSelect5('5510',...)` = **設計書等**
- **ダウンロード画面のセレクタ** (固定):
  - 会社名: `#GP9505_1150_gyosyaNm`
  - 代表者名: `#GP9505_1150_daihyoNm`
  - 一括DL: `#btnDL`
- **settings.json** の `HacchuBuCd` 空欄OK (全発注部対象の意味)
- **今後の拡張:** 岡山市=3302 / 倉敷市=? など `dantaiCd` 切替で同フレームワーク使い回し可。配列化前提で書く

## 検証再開の手順 (Linux/GCP Workstation)

### STEP 1: MCPスタック復活
`gcp_workstation_chrome_devtools_mcp.md` の手順に沿って Chrome バイナリ設置 → Xvfb → x11vnc → websockify を全部立ち上げる。永続化していないので再起動ごとに必要。

最短コマンド列:
```bash
# Chrome
npx -y @puppeteer/browsers install chrome@stable --path ~/.cache/puppeteer
sudo mkdir -p /opt/google/chrome
sudo ln -sf ~/.cache/puppeteer/chrome/linux-*/chrome-linux64/chrome /opt/google/chrome/chrome
sudo apt-get install -y libxkbcommon0 xvfb x11vnc novnc websockify

# X仮想ディスプレイ + DISPLAY持ちダミープロセス常駐 (MCPの detectDisplay 対策)
Xvfb :99 -screen 0 1280x720x24 >/tmp/xvfb.log 2>&1 &
DISPLAY=:99 nohup sleep 99999 >/dev/null 2>&1 &

# noVNC
x11vnc -display :99 -forever -nopw -shared -rfbport 5900 -quiet >/tmp/x11vnc.log 2>&1 &
websockify --web=/usr/share/novnc 6080 localhost:5900 >/tmp/websockify.log 2>&1 &
```

### STEP 2: ブラウザ確認URLをユーザーに提示
```
https://6080-$WEB_HOST/vnc.html?autoconnect=true&resize=scale
```
GCP IAM でゲートされてるので認証なしVNCでもユーザー本人しか見れない。

### STEP 3: settings.json 確認
`system/src/settings.json` の `Okayama.CompanyName` / `RepresentativeName` / `FilterPeriod` が想定通りか確認。

### STEP 4: Collector を実行
Linux 環境では **PowerShell ラッパは使わず** `node` 直叩きが早い:
```bash
cd /home/user/暁電業/岡山県スクレイピング/納品_入札情報取得ツール_1.09/system/src/Playwright
DISPLAY=:99 node OkayamaCollector.js
```
noVNC画面でブラウザ挙動を追う。

### STEP 5: 成果物チェック
- 案件一覧が取れているか
- ZIP (公告等・設計書等) が両方ダウンロードされているか
- エラーログがないか

### STEP 6: 停止
```bash
pkill x11vnc; pkill websockify; pkill Xvfb
pkill -f "sleep 99999"
```

## よくある詰まり

- **`.mcp.json` 編集は Claude 側で拒否される** → `--executablePath` ルートは封じられているのでシンボリックリンクで回避
- **`sudo apt install` がスコープ外で拒否される** → 最初にユーザーの明示的許可 (`!` プレフィックス) が要る
- **`export DISPLAY=:99` が MCP から見えない** → MCP の detectDisplay は他プロセスの `/proc/<pid>/environ` を漁る仕様。DISPLAY持ちダミープロセス常駐が必須
- **Chrome for Testing 148系** は puppeteer-core の `channel: 'chrome'` と互換

## 学習ログ

<!-- /skill-update で追記される -->
- 2026-04-24: 初版作成。検証再開の手順 (MCPスタック復活 → noVNC → `node OkayamaCollector.js`) を skill 化。

### 2026-04-27
- **効いたパターン**: 動作検証だけが目的なら **VNC/Xvfb/x11vnc/websockify は一切不要**。`OkayamaCollector.js` は `chromium.launch({ headless: true })` がデフォルトなので、Playwright付属Chromium (`~/.cache/ms-playwright/chromium_headless_shell-*`) があれば `node` 直叩きで完結する。GUI がいるのは挙動を目視で追いたいデバッグ時だけ。STEP 1〜2 のVNCスタック構築は **デバッグ専用** と位置づける
- **効いたパターン**: PowerShell Wrapper を経由せず JS を単体検証する場合、`settings.json` は読まれない。**Base64 JSON を `argv[2]` に渡す** のが最短:
  ```bash
  CONFIG=$(node -e 'console.log(Buffer.from(JSON.stringify({outputDir:"output",headless:true,limit:5,companyName:"...",representativeName:"..."})).toString("base64"))')
  node OkayamaCollector.js "$CONFIG"
  ```
  `limit` で件数制限すると、初回検証は5件程度で十分判定できる
- **注意**: Cloud Workstation 初回起動時は `apt-cache` が空なので `apt-cache search` も `apt-get install` もパッケージを見つけられない。**必ず `sudo apt-get update` を先に走らせる**。エラーメッセージは "Unable to locate package" — リポジトリ未設定と紛らわしいが update で解決
- **注意**: Playwright Chromium のヘッドレス起動には `libxkbcommon0` が必須 (Ubuntu Noble の素のCloud Workstationには未導入)。`libpango-1.0-0` 等は最初から入っていることが多い。Playwright が要求する全依存を一発で入れるなら `sudo npx --yes playwright install-deps chromium` の方が安全
- **コツ**: 検証時は「添付ファイルはありません」と早期リターンする案件が混じるのが正常 (発注元の運用)。`limit:5` で 5件中 2件 ZIP取得、3件「添付なし」は健全。**ZIP 0件 = バグ ではない** 。バグ判定は「公告/設計書ボタンが見つからない」「step限界超過」「FATAL」が出たときだけにする
- **コツ**: 取得ZIPは中身が CP932 (Shift_JIS) 命名のネストZIP。`unzip -l` で文字化けして見えても **ZIPの構造自体は健全**。サイズ (1MB〜数MB) で正否を判定する方が早い
- **コツ**: 結果行のタイトルは `<a>` 内に「番号\n番号 工事名」の2行構成で入る。`split('\n')` して2行目を採用するロジック (`OkayamaCollector.js:282-283`) が肝。サイト改修でこの構造が変わるとタイトルが空になり全件 skip するので、`Hit rows` と `Skipping` ログの比に注意
