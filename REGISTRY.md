# Skills Registry

| スキル | コマンド | 説明 | 最終更新 |
|---|---|---|---|
| GAS並列実装 | `/gas-parallel` | GASプロジェクトをファイル競合なしで4並列実装→push→deploy | 2026-03-25 |
| GASキャッシュ | `/gas-cache-chunk` | CacheService 100KB制限をチャンク分割で回避するパターン | 2026-03-25 |
| GAS表示切替 | `/gas-display` | HtmlService iframeで確実に動くshow/hideパターン（style.display） | 2026-03-25 |
| GAS PDF生成 | `/gas-pdf-export` | Sheets→PDF変換のA4横・1ページ収め・中央配置パターン | 2026-03-25 |
| GASロック制御 | `/gas-lockservice` | LockService + 処理中フラグによる二重実行防止パターン | 2026-03-25 |
| GAS落とし穴集 | `/gas-gotchas` | GAS・スプレッドシート開発でハマりやすいバグパターンと回避策 | 2026-03-25 |
| スキル同期 | `/skill-sync` | GitHubから最新スキルをpullして`~/.claude/commands/`に反映 | 2026-03-25 |
| スキル学習 | `/skill-update` | 作業後のコツ・パターンをスキルに追記してGitHubにpush | 2026-03-25 |

## カテゴリ別

**開発フロー**
- `/gas-parallel` — GASの並列実装パターン

**GAS固有パターン**
- `/gas-cache-chunk` — CacheService 100KB制限をチャンク分割で回避
- `/gas-display` — HtmlService iframeで確実に動くshow/hideパターン
- `/gas-pdf-export` — Sheets→PDF変換のA4横・1ページ収め・中央配置
- `/gas-lockservice` — LockService + 処理中フラグによる二重実行防止
- `/gas-gotchas` — setValues/boolean/innerHTML等の落とし穴と回避策

**スキル管理**
- `/skill-sync` — リポジトリとの同期
- `/skill-update` — 学習ログの書き込み
