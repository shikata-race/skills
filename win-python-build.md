# win-python-build — Linux環境からWindows向けPython exe配布キットを作るパターン

Linux（Claude Code等）でPythonスクリプトを開発し、Windows上でPyInstallerでexe化して配布するときの注意点。

## ファイル別エンコーディング原則

| ファイル種別 | エンコーディング | 理由 |
|---|---|---|
| `.bat` | **Shift-JIS (cp932)** | Windows cmd.exeがデフォルトcp932で読むため |
| `.py` / `.spec` | **UTF-8 (BOM無し)** | Python 3のデフォルト。`# -*- coding: utf-8 -*-` 宣言があればBOM不要 |

Linux上で `.bat` を書くときは `encode('cp932')` で保存する:
```python
with open('build.bat', 'wb') as f:
    f.write(content.encode('cp932').replace(b'\n', b'\r\n'))
```

## OS差異チェックリスト

- `/dev/null` → `NUL`（Windowsにはdeviceファイルが無い）
- パス区切り `/` → `\`
- 改行コード LF → CRLF（.batは特に重要）

## Python バージョン選定

- PyInstaller等のビルドツールは最新Pythonに即対応しない。**安定版（3.12等）を選ぶ**
- `Requires-Python` の上限に注意（例: `<3.14` で3.14は不可）

## Microsoft Store エイリアス問題

- Windowsで `python` コマンドがMicrosoft Storeのエイリアスに奪われてPATH設定が効かない場合がある
- **回避策**: バッチ内で `%LOCALAPPDATA%\Programs\Python\PythonXXX\python.exe` のフルパスを使う
- `set PYTHON="%LOCALAPPDATA%\Programs\Python\Python312\python.exe"` で変数化すると保守しやすい

## 管理者権限なしのインストール

- Pythonインストーラーで「Install for all users」のチェックを外すと `%LOCALAPPDATA%` にインストールされ管理者権限不要

## 学習ログ

### 2026-04-08
- **注意**: `.bat`をLinuxでUTF-8のまま保存するとWindows cmd.exeで日本語ファイル名が化ける。Shift-JISで保存必須
- **注意**: `.spec`ファイルがShift-JISだとPyInstallerが`utf-8 codec can't decode`エラー。specはUTF-8で保存必須
- **注意**: `/dev/null` はWindowsに存在しない。バッチ内では `NUL` を使う。「指定されたパスが見つかりません」の原因になる
- **注意**: Python 3.14はPyInstaller 6.x が `Requires-Python <3.14` で未対応。3.12を使うのが安全
- **効いたパターン**: `set PYTHON="%LOCALAPPDATA%\Programs\Python\Python312\python.exe"` でMS Storeエイリアス問題を完全回避
- **コツ**: ZIPで配布する場合、bat=cp932 / py,spec=UTF-8 のエンコーディング混在に注意。ZIPを再作成するたびに両方確認する
