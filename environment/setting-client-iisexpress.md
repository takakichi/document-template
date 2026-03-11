
# ローカル開発環境 構築ガイド

本資料は、管理者権限のないWindows環境において、**IIS Express**と**MIIS**を組み合わせてMarkdownドキュメント管理システムを構築し、VS Codeと連携させる手順についてまとめています。
なお、本資料は生成AIで作成した資料をベースとしています。

## 1. 構成

* **Webサーバー:** IIS Express（管理者権限不要で動作）
* **レンダリング:** [MIIS](https://github.com/jmalarcon/MIIS) (Markdown-based CMS)
* **エディタ:** Visual Studio Code
* **セキュリティ:** `localhost` 限定バインドによるローカルアクセス制限

---

## 2. 導入手順

### 2.1 MIISの配置

1. MIISの最新リリースをダウンロードし、任意のフォルダ（例: `C:\Work\Docs`）に解凍します。
2. フォルダ直下に `bin` フォルダや `web.config` があることを確認します。

### 2.2 ディレクトリ参照の有効化 (`web.config`)

ファイル一覧を表示させるため、`web.config` の `<system.webServer>` セクションに以下を追記します。

```xml
<system.webServer>
  <directoryBrowse enabled="true" />
  <modules runAllManagedModulesForAllRequests="true"/>
  <handlers>
    <add name="MarkdownHandler" path="*.md" verb="GET" type="MIISHandler.MIISHandler, MIISHandler" resourceType="Unspecified" />
  </handlers>
</system.webServer>

```

---

## 3. 起動・運用の自動化

### 3.1 起動用バッチファイル (`start_miis.bat`)

管理者権限なしで、安全に（localhost限定で）起動するためのバッチです。

```batch
@echo off
setlocal
set IIS_EXE="C:\Program Files\IIS Express\iisexpress.exe"
set DOC_PATH="%~dp0"
set PORT=8080

echo Starting MIIS at http://localhost:%PORT%
%IIS_EXE% /path:%DOC_PATH% /port:%PORT% /hostname:localhost /clr:4.0
pause

```

### 3.2 VS Code タスク連携 (`.vscode/tasks.json`)

VS Code上から `Ctrl + Shift + B` でサーバーを起動できるようにします。

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Start MIIS (IIS Express)",
            "type": "shell",
            "command": "${workspaceFolder}/start_miis.bat",
            "isBackground": true,
            "problemMatcher": []
        }
    ]
}

```

---

## 4. 開発における注意点と棲み分け

### 4.1 Python (FastAPI/Flask) との共存

IIS ExpressでPythonを動かすのは設定が複雑になるため、**「ポート分離」**による個別管理を推奨します。

| サービス | サーバーツール | 推奨ポート | 備考 |
| --- | --- | --- | --- |
| **Markdown資料** | IIS Express (MIIS) | 8080 | ドキュメント閲覧用 |
| **FastAPI** | Uvicorn | 8000 | `uvicorn main:app --reload` |
| **Flask** | Flask Development Server | 5000 | `flask run` |

### 4.2 トラブルシューティング

* **404エラーが発生する場合:**
* `iisexpress.exe` の `/path` 引数が、`web.config` のある物理フォルダを正しく指しているか確認してください。
* URLの末尾にファイル名（`index.md`など）を付けてアクセスを試みてください。


* **ポートが競合する場合:**
* 別のプロセスがポートを使用している場合、バッチファイルの `PORT` 変数を変更してください。



---

## 5. 推奨されるVS Code設定

Markdown編集を快適にするため、`settings.json` への追記を推奨します。

* `editor.wordWrap`: "on" （長い行の折り返し）
* `files.autoSave`: "afterDelay" （自動保存によるブラウザへの即時反映）

---

