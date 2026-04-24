# Skills Registry

| スキル | コマンド | 説明 | 最終更新 |
|---|---|---|---|
| GAS並列実装 | `/gas-parallel` | GASプロジェクトをファイル競合なしで4並列実装→push→deploy | 2026-03-25 |
| GASキャッシュ | `/gas-cache-chunk` | CacheService 100KB制限をチャンク分割で回避するパターン | 2026-03-25 |
| GAS表示切替 | `/gas-display` | HtmlService iframeで確実に動くshow/hideパターン（style.display） | 2026-04-08 |
| GAS PDF生成 | `/gas-pdf-export` | Sheets→PDF変換のA4横・1ページ収め・中央配置パターン | 2026-03-25 |
| GASロック制御 | `/gas-lockservice` | LockService + 処理中フラグによる二重実行防止パターン | 2026-03-25 |
| GAS落とし穴集 | `/gas-gotchas` | GAS・スプレッドシート開発でハマりやすいバグパターンと回避策 | 2026-04-17 |
| スキル同期 | `/skill-sync` | GitHubから最新スキルをpullして`~/.claude/commands/`に反映 | 2026-03-25 |
| スキル学習 | `/skill-update` | 作業後のコツ・パターンをスキルに追記してGitHubにpush | 2026-03-25 |
| Win Python配布 | `/win-python-build` | Linux→Windows向けPython exeビルドキット作成の注意点 | 2026-04-08 |
| 備福拘束時間 | `/bifuku-kosoku` | 備福運送 拘束時間管理ロボットの保守（BizRobo→VBA/PS置き換え） | 2026-04-10 |
| 暁電業岡山 | `/akatsuki-okayama` | 暁電業 岡山県入札情報取得ツールの検証・保守（DENCHO / noVNC + MCP） | 2026-04-24 |

## カテゴリ別

**開発フロー**
- `/gas-parallel` — GASの並列実装パターン

**GAS固有パターン**
- `/gas-cache-chunk` — CacheService 100KB制限をチャンク分割で回避
- `/gas-display` — HtmlService iframeで確実に動くshow/hideパターン
- `/gas-pdf-export` — Sheets→PDF変換のA4横・1ページ収め・中央配置
- `/gas-lockservice` — LockService + 処理中フラグによる二重実行防止
- `/gas-gotchas` — setValues/boolean/innerHTML等の落とし穴と回避策

**Windows配布**
- `/win-python-build` — Linux→Windows向けPython exeビルドキット（エンコーディング・バージョン注意点）

**顧客プロジェクト**
- `/bifuku-kosoku` — 備福運送 拘束時間管理ロボットの保守・修正（VBA/PowerShell/BizRobo）
- `/akatsuki-okayama` — 暁電業 岡山県入札情報取得ツールの検証・保守（DENCHO / noVNC + MCP 検証手順含む）

**スキル管理**
- `/skill-sync` — リポジトリとの同期
- `/skill-update` — 学習ログの書き込み
