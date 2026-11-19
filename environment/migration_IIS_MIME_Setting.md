# IIS MIMEタイプ移行手順書（Rev2）

## 1. 目的・対象

Windows Server 2016からWindows Server 2022へIISのWebサイト・Webアプリケーションを移行する際、追加設定したMIMEタイプを漏れなく移行する。

- 対象：IISマネージャーでWebサイト／Webアプリケーションに設定したMIMEタイプ（数種類程度）
- 方式：**有効MIME一覧をCSV出力 → 追加分を目視選別 → 移行先で追加予定を確認 → 不足分のみ登録 → 検証**
- サーバ全体の標準設定の一括コピー、既存設定の削除・上書きはしない。
- Webサイトのルート（`/`）と、IISの**アプリケーション**のルート（例：`/SystemA`）を対象とする。アプリケーション配下のフォルダ／仮想ディレクトリ固有の設定は対象外。

> **重要**：IISのMIME設定は親階層から継承される。CSVに出力した行は「その場所で有効なMIMEタイプ」であり、「その場所で追加したMIMEタイプ」ではない。追加分は必ず目視で選別する。

## 2. 前提条件

1. 移行元・移行先のIISに対象Webサイト／Webアプリケーションが構成されている。
2. 管理者権限の**Windows PowerShell 5.1（64ビット）**で実行する。
3. IIS管理コンポーネント `Microsoft.Web.Administration` を利用できる。
4. 移行先でサイト名・アプリケーションパスが異なる場合は、登録用CSVを修正する。
5. `web.config` を移行する場合、`staticContent` のMIME設定が含まれていないか先に確認する。移行済みの設定を再登録しない。
6. 作業中は他の管理者によるIIS設定変更を避ける。

## 3. 移行作業の流れ

| 順序 | 実施サーバ | 作業 |
|---|---|---|
| 1 | 移行元 | サイト・アプリケーション別の有効MIME一覧をCSV出力 |
| 2 | 移行元 | IISマネージャーと設定ファイルを照合し、追加分を選別 |
| 3 | 移行元 | 選別した行だけを `mime-to-import.csv` に保存し、重複を確認 |
| 4 | 移行先 | IIS設定のバックアップを取得し、既存のMIME設定を確認 |
| 5 | 移行先 | 登録スクリプトを確認モードで実行 |
| 6 | 移行先 | 問題がなければ `-Apply` を指定して登録 |
| 7 | 移行先 | IISマネージャーとHTTP応答を確認 |

## 4. 移行元：有効MIME一覧のCSV出力

次を `Export-IisMime.ps1` として保存する。IIS構成APIのコレクションを直接読み出す。**追加分の自動判定はしない。**

```powershell
param(
    [string]$OutputCsv = 'C:\Temp\iis-mime-all.csv'
)

$ErrorActionPreference = 'Stop'
Add-Type -AssemblyName 'Microsoft.Web.Administration'

$server = New-Object Microsoft.Web.Administration.ServerManager
try {
    $rows = @(
        foreach ($site in $server.Sites) {
            foreach ($app in $site.Applications) {
                $config = $server.GetWebConfiguration($site.Name, $app.Path)
                $section = $config.GetSection('system.webServer/staticContent')

                foreach ($entry in $section.GetCollection()) {
                    if ($entry.ElementTagName -ne 'mimeMap') { continue }
                    [PSCustomObject]@{
                        SiteName        = $site.Name
                        ApplicationPath = $app.Path
                        Extension       = [string]$entry['fileExtension']
                        MimeType        = [string]$entry['mimeType']
                    }
                }
            }
        }
    )
}
finally {
    $server.Dispose()
}

if ($rows.Count -eq 0) {
    throw 'MIMEタイプが0件でした。IIS構成の取得結果を確認してください。'
}

$folder = Split-Path -Parent $OutputCsv
if ($folder -and -not (Test-Path -LiteralPath $folder)) {
    New-Item -ItemType Directory -Path $folder -Force | Out-Null
}

$rows | Sort-Object SiteName, ApplicationPath, Extension |
    Export-Csv -LiteralPath $OutputCsv -NoTypeInformation -Encoding UTF8
Write-Host "Export: $OutputCsv / Count: $($rows.Count)"
```

実行例：

```powershell
.\Export-IisMime.ps1 -OutputCsv 'C:\Temp\iis-mime-all.csv'
```

**確認事項**：CSV内に既知のMIMEタイプが出ているか、IISマネージャーの「MIMEの種類」と照合する。処理中に構成取得エラーが発生した場合はスクリプトを停止し、不完全なCSVを移行に使用しない。実行前から同名CSVが残っている場合は、更新日時も確認する。

### 4.1 追加分の目視選別

1. IISマネージャーで対象サイト／アプリケーションを選択し、「MIMEの種類」を開く。
2. 「エントリの種類」（ローカル／継承）を確認する。判断が難しければ対象の `web.config`、`applicationHost.config` の `location` 設定を確認する。
3. 出力CSVから**移行が必要な行だけ**を残して `C:\Temp\mime-to-import.csv` として保存する。
4. `SiteName`・`ApplicationPath`・`Extension` の組合せに重複がないことを確認する。`/` と `/SystemA` を混同しない。
5. Windows PowerShell 5.1で読み込めるよう、CSVは**UTF-8（BOM付き）**で保存する。

CSV例（サンプル値）：

```csv
SiteName,ApplicationPath,Extension,MimeType
Intranet,/,.abc,application/x-abc
Intranet,/SystemA,.xyz,application/x-xyz
```

> アプリケーションが親サイトから継承するMIMEタイプは、原則として親側の移行状況も確認する。`<remove>` や `<clear>` がある場合はCSVだけでは再現できないため、設定ファイルを個別確認する。

## 5. 移行先：IIS設定のバックアップ

管理者権限で実行する。

```powershell
& "$env:windir\System32\inetsrv\appcmd.exe" add backup BeforeMimeMigration
if ($LASTEXITCODE -ne 0) { throw 'IIS構成バックアップに失敗しました。' }
```

バックアップの成功を確認してから次に進む。

> **切り戻し注意**：`appcmd restore backup BeforeMimeMigration` はMIMEタイプだけでなく、バックアップ対象のIIS構成全体を復元する。ほかの設定変更も巻き戻すおそれがあるため、MIMEタイプだけの切り戻しでは、今回追加した設定をIISマネージャーで個別に削除する。アプリケーション配下の `web.config` はこのバックアップで保護されるとは限らないため、必要に応じて別途退避する。

## 6. 移行先：不足分のみ登録

次を `Import-IisMime.ps1` として保存する。`-Apply` がない場合は**確認のみ**。既存拡張子がある場合は上書きしない。CSVの重複や不正な対象は登録前に検出して停止する。

```powershell
param(
    [string]$InputCsv = 'C:\Temp\mime-to-import.csv',
    [switch]$Apply
)

$ErrorActionPreference = 'Stop'
Add-Type -AssemblyName 'Microsoft.Web.Administration'

$items = @(Import-Csv -LiteralPath $InputCsv -Encoding UTF8)
if ($items.Count -eq 0) { throw '登録対象CSVが空です。' }

$seen = @{}
foreach ($item in $items) {
    $site = [string]$item.SiteName
    $app  = [string]$item.ApplicationPath
    $ext  = [string]$item.Extension
    $mime = [string]$item.MimeType

    if ([string]::IsNullOrWhiteSpace($site) -or
        [string]::IsNullOrWhiteSpace($app) -or
        [string]::IsNullOrWhiteSpace($ext) -or
        [string]::IsNullOrWhiteSpace($mime) -or
        -not $app.StartsWith('/') -or
        -not $ext.StartsWith('.')) {
        throw "CSVの項目が不正です: $site / $app / $ext / $mime"
    }
    $key = "$($site.ToLowerInvariant())|$($app.ToLowerInvariant())|$($ext.ToLowerInvariant())"
    if ($seen.ContainsKey($key)) { throw "CSVに重複があります: $site $app $ext" }
    $seen[$key] = $true
}

$server = New-Object Microsoft.Web.Administration.ServerManager
try {
    # 全行を先に検査。競合があれば登録を始めない。
    $plans = @(
        foreach ($item in $items) {
            $siteName = $item.SiteName.Trim()
            $appPath  = $item.ApplicationPath.Trim()
            $ext      = $item.Extension.Trim()
            $mime     = $item.MimeType.Trim()

            $site = $server.Sites[$siteName]
            if ($null -eq $site) { throw "サイトがありません: $siteName" }
            $app = @($site.Applications | Where-Object { $_.Path -ieq $appPath })
            if ($app.Count -ne 1) { throw "アプリケーションがありません: $siteName $appPath" }

            $section = $server.GetWebConfiguration(
                $siteName, $appPath).GetSection('system.webServer/staticContent')
            $matches = @($section.GetCollection() | Where-Object {
                $_.ElementTagName -eq 'mimeMap' -and
                [string]$_['fileExtension'] -ieq $ext
            })
            if ($matches.Count -gt 0) {
                $current = [string]$matches[0]['mimeType']
                if ($current -eq $mime) {
                    Write-Host "SKIP (same): $siteName $appPath $ext"
                } else {
                    throw "CONFLICT: $siteName $appPath $ext : $current / $mime"
                }
                continue
            }
            [PSCustomObject]@{
                SiteName = $siteName; AppPath = $appPath
                Extension = $ext; MimeType = $mime
            }
        }
    )

    foreach ($plan in $plans) {
        Write-Host "PLAN ADD: $($plan.SiteName) $($plan.AppPath) $($plan.Extension) -> $($plan.MimeType)"
    }
    if (-not $Apply) {
        Write-Host '確認のみです。登録する場合は -Apply を指定してください。'
        return
    }

    foreach ($plan in $plans) {
        $section = $server.GetWebConfiguration(
            $plan.SiteName, $plan.AppPath).GetSection('system.webServer/staticContent')
        $collection = $section.GetCollection()
        $entry = $collection.CreateElement('mimeMap')
        $entry['fileExtension'] = $plan.Extension
        $entry['mimeType'] = $plan.MimeType
        $collection.Add($entry)
    }
    if ($plans.Count -gt 0) {
        $server.CommitChanges()
    }
    Write-Host "Completed: Added $($plans.Count) MIME type(s)."
}
finally {
    $server.Dispose()
}
```

実行例：

```powershell
# 1. 追加予定のみ確認
.\Import-IisMime.ps1 -InputCsv 'C:\Temp\mime-to-import.csv'

# 2. 内容を確認後、実際に登録
.\Import-IisMime.ps1 -InputCsv 'C:\Temp\mime-to-import.csv' -Apply
```

- `SKIP (same)`：有効な設定がすでに同一なので登録しない。
- `CONFLICT`：同一拡張子の有効なMIMEタイプが異なるため、**登録前に停止**する。必要な変更は個別に判断する。
- `PLAN ADD`：追加予定。`-Apply` を付けない限り変更しない。
- 事前検査で問題がなくても、設定ファイルの書込み権限・ロック・継承制約などにより登録が失敗する可能性がある。`-Apply` 実行後は必ず実設定を確認する。
- このスクリプトは**有効設定の一致**を判定する。MIMEタイプの設定保存先が移行元と一致することまでは保証しない。登録後、必要に応じて `web.config` と `applicationHost.config` の保存場所を確認する。

## 7. 移行後の確認

1. IISマネージャーで対象サイト／アプリケーションの「MIMEの種類」を開き、拡張子とMIMEタイプが期待どおりであることを確認する。
2. 対象ファイルが実際に存在するURLでHTTP応答を確認する。例：

   ```powershell
   curl.exe -I 'http://移行先ホスト/SystemA/sample.xyz'
   ```

   HTTPステータスと `Content-Type` を確認する。認証が必要な場合やHEADを受け付けない場合は、ブラウザまたは適切な認証付きGETで確認する。
3. `CONFLICT` や実行エラーが出た場合は、原因を確認し、必要な設定だけ個別に修正する。
4. `web.config` の移行によって設定済みだった場合は二重登録しない。

## 8. 対象外・留意事項

- サーバ全体の標準MIMEタイプは移行しない。
- Webサイト／アプリケーションの下位フォルダ・仮想ディレクトリ固有のMIME設定は別途確認する。
- `web.config` の `<remove>`、`<clear>` などの継承制御はCSVに含めない。
- IISマネージャーで追加した設定の保存先は、`web.config` または `applicationHost.config` の `location` など、構成により異なる。
- CSV編集時はサイト名・アプリケーションパス・拡張子の誤入力に注意する。
- スクリプトは**開発サーバで既知のMIME設定を使って試行**し、出力と登録結果を確認してから本番で利用する。

## 9. 完了条件

- [ ] 移行元のCSV出力結果をIISマネージャーと照合済み
- [ ] 追加分だけを選別し、CSVの重複・保存形式を確認済み
- [ ] 移行先のIIS設定バックアップ取得済み
- [ ] 登録前の `PLAN ADD`・`SKIP`・`CONFLICT` を確認済み
- [ ] `-Apply` 実行後、すべての対象設定が一致することを確認済み
- [ ] 対象ファイルのHTTP応答と `Content-Type` を確認済み
