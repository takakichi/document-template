# IIS MIMEタイプ移行手順書

## 1. 目的・対象

Windows Server 2016からWindows Server 2022へIISのWebサイト・Webアプリケーションを移行する際、追加設定したMIMEタイプを漏れなく移行する。

- 対象：IISマネージャーでWebサイト／Webアプリケーションに設定したMIMEタイプ
- 件数：数種類程度を想定
- 方式：設定一覧のCSV出力 → 移行対象の目視選別 → 移行先で不足分だけ登録 → 確認
- 標準設定の一括コピーや既存設定の削除・上書きは行わない

> **重要**：IISのMIME設定は継承される。IISマネージャーに表示された項目すべてが、そのアプリケーションに独自追加されたものとは限らない。`IsLocallyStored` のみを根拠に追加分を断定しない。

## 2. 前提条件

1. 移行元・移行先でIISと対象Webサイト／アプリケーションが構成されていること。
2. 管理者権限のWindows PowerShell 5.1で実行すること。
3. IIS管理用の `WebAdministration` モジュールが利用できること。
4. 移行先に同じサイト名・アプリケーションパスがあること。異なる場合はCSVを修正する。
5. `web.config` をコピーする場合、MIME設定がその中に含まれていないか確認すること（重複登録防止）。

## 3. 移行作業の流れ

| 順序 | 実施サーバ | 作業 |
|---|---|---|
| 1 | 移行元 | Webサイト・アプリケーション別の有効MIME一覧をCSV出力 |
| 2 | 移行元 | IISマネージャーと設定ファイルを照合し、独自追加分を選別 |
| 3 | 移行元 | 選別した行だけを `mime-to-import.csv` に保存 |
| 4 | 移行先 | IIS設定をバックアップ |
| 5 | 移行先 | CSVを使って不足分だけ追加 |
| 6 | 移行先 | IIS設定とHTTP応答を確認 |

## 4. 移行元：有効MIME一覧を出力

以下を `Export-IisMime.ps1` として保存する。これは**有効な設定の一覧**を出力するものであり、追加分だけを自動識別するものではない。

```powershell
param(
    [string]$OutputCsv = 'C:\Temp\iis-mime-all.csv'
)

$ErrorActionPreference = 'Stop'
Import-Module WebAdministration

$rows = foreach ($site in Get-Website) {
    $apps = @(Get-WebApplication -Site $site.Name)
    $paths = @('/') + @($apps | ForEach-Object { $_.Path })

    foreach ($appPath in $paths) {
        $location = if ($appPath -eq '/') {
            $site.Name
        } else {
            $site.Name + $appPath
        }

        $entries = Get-WebConfigurationProperty `
            -PSPath 'MACHINE/WEBROOT/APPHOST' `
            -Location $location `
            -Filter 'system.webServer/staticContent' `
            -Name '.'

        foreach ($entry in $entries.Collection) {
            if ($entry.ElementTagName -ne 'mimeMap') { continue }

            [PSCustomObject]@{
                SiteName        = $site.Name
                ApplicationPath = $appPath
                Extension       = [string]$entry.GetAttributeValue('fileExtension')
                MimeType        = [string]$entry.GetAttributeValue('mimeType')
            }
        }
    }
}

$folder = Split-Path -Parent $OutputCsv
if ($folder -and -not (Test-Path $folder)) {
    New-Item -ItemType Directory -Path $folder -Force | Out-Null
}

$rows | Sort-Object SiteName, ApplicationPath, Extension |
    Export-Csv -Path $OutputCsv -NoTypeInformation -Encoding UTF8
Write-Host "Export: $OutputCsv"
```

実行例：

```powershell
.\Export-IisMime.ps1 -OutputCsv 'C:\Temp\iis-mime-all.csv'
```

**確認事項**：実行環境で `Get-WebConfigurationProperty` の返却形式が異なる場合があるため、まず1サイトでCSV内容をIISマネージャーの「MIMEの種類」と照合する。アプリケーション配下のサブフォルダ固有設定はこの一覧の対象外。

### 4.1 追加分を選別する

1. IISマネージャーで対象サイト／アプリケーションの「MIMEの種類」を開く。
2. 設定のエントリ種類（ローカル／継承）を確認する。画面表示だけで判定できない場合は、対象の `web.config` や `applicationHost.config` 内の `location` 設定を確認する。
3. CSVから移行する行だけを残し、`mime-to-import.csv` として保存する。
4. サイトの `/` とアプリケーションの `/AppName` を混同しない。

CSV例（サンプル値）：

```csv
SiteName,ApplicationPath,Extension,MimeType
Intranet,/, .abc,application/x-abc
Intranet,/SystemA,.xyz,application/x-xyz
```

※実際のCSVでは値の前後に余分な空白を含めないこと。以下が正しい例。

```csv
SiteName,ApplicationPath,Extension,MimeType
Intranet,/,.abc,application/x-abc
Intranet,/SystemA,.xyz,application/x-xyz
```

## 5. 移行先：バックアップ

管理者権限で実行する。

```powershell
& "$env:windir\System32\inetsrv\appcmd.exe" add backup BeforeMimeMigration
```

バックアップの成否を確認してから次に進む。

## 6. 移行先：不足分のみ登録

以下を `Import-IisMime.ps1` として保存する。`-Apply` を指定しない場合は**確認のみ**で、設定は変更しない。

```powershell
param(
    [string]$InputCsv = 'C:\Temp\mime-to-import.csv',
    [switch]$Apply
)

$ErrorActionPreference = 'Stop'
Import-Module WebAdministration

$items = Import-Csv -Path $InputCsv -Encoding UTF8

foreach ($item in $items) {
    $site = $item.SiteName.Trim()
    $app = $item.ApplicationPath.Trim()
    $extension = $item.Extension.Trim()
    $mimeType = $item.MimeType.Trim()

    if (-not $site -or -not $app -or -not $extension -or -not $mimeType) {
        Write-Warning 'SKIP: CSVに空欄があります'
        continue
    }
    if (-not $extension.StartsWith('.')) {
        Write-Warning "SKIP: 拡張子が不正です: $extension"
        continue
    }

    $location = if ($app -eq '/') { $site } else { $site + $app }

    if (-not (Get-Website -Name $site -ErrorAction SilentlyContinue)) {
        Write-Warning "SKIP: サイトがありません: $site"
        continue
    }
    if ($app -ne '/') {
        $apps = @(Get-WebApplication -Site $site)
        if (-not @($apps | Where-Object { $_.Path -eq $app }).Count) {
            Write-Warning "SKIP: アプリケーションがありません: $location"
            continue
        }
    }

    $config = Get-WebConfigurationProperty `
        -PSPath 'MACHINE/WEBROOT/APPHOST' `
        -Location $location `
        -Filter 'system.webServer/staticContent' `
        -Name '.'

    $found = @($config.Collection | Where-Object {
        $_.ElementTagName -eq 'mimeMap' -and
        [string]$_.GetAttributeValue('fileExtension') -ieq $extension
    })

    if ($found.Count -gt 0) {
        $current = [string]$found[0].GetAttributeValue('mimeType')
        if ($current -eq $mimeType) {
            Write-Host "SKIP (same): $location $extension"
        } else {
            Write-Warning "CONFLICT: $location $extension : $current / $mimeType"
        }
        continue
    }

    if (-not $Apply) {
        Write-Host "PLAN ADD: $location $extension -> $mimeType"
        continue
    }

    try {
        Add-WebConfigurationProperty `
            -PSPath 'MACHINE/WEBROOT/APPHOST' `
            -Location $location `
            -Filter 'system.webServer/staticContent' `
            -Name '.' `
            -Value @{ fileExtension = $extension; mimeType = $mimeType }
        Write-Host "ADDED: $location $extension -> $mimeType"
    } catch {
        Write-Warning "ERROR: $location $extension : $_"
    }
}
```

実行例：

```powershell
# まず追加予定だけ確認
.\Import-IisMime.ps1 -InputCsv 'C:\Temp\mime-to-import.csv'

# 内容確認後に登録
.\Import-IisMime.ps1 -InputCsv 'C:\Temp\mime-to-import.csv' -Apply
```

**注意**：同一拡張子が既に存在してMIMEタイプが異なる場合は `CONFLICT` と表示し、自動で変更しない。継承された設定が存在する場合も同様に上書きしない。

## 7. 移行後の確認

1. IISマネージャーで対象サイト／アプリケーションの「MIMEの種類」を開き、移行対象の拡張子とMIMEタイプがあることを確認する。
2. 対象URLへアクセスし、HTTPステータスと `Content-Type` を確認する（アクセス制御やファイルの存在にも注意）。
3. `CONFLICT` や `ERROR` があった場合は個別に調査し、必要な修正のみ手動で行う。
4. 設定ファイルの移行によって既に反映済みの場合は、追加登録しない。

## 8. 対象外・留意事項

- サーバ全体の標準MIMEタイプを丸ごと移行することはしない。
- Webサイト／アプリケーションの**下位ディレクトリだけ**に設定したMIMEタイプは別途確認する。
- `web.config` の `<remove>` や `<clear>` による継承の打消しは本手順のCSVに含めない。これらがある場合は設定ファイルごと確認する。
- IISマネージャーからの登録でも保存先が `web.config` または `applicationHost.config` の `location` となる場合がある。保存先の確認を省略しない。
- CSVを編集する場合、サイト名・アプリケーションパス・拡張子の誤入力に注意する。

## 9. 完了条件

- [ ] 移行対象MIMEタイプのCSVを確認済み
- [ ] 移行先のIIS設定バックアップ取得済み
- [ ] 登録前の `PLAN ADD` と `CONFLICT` を確認済み
- [ ] 移行対象がすべて登録または同一設定として確認済み
- [ ] 対象ファイルのHTTP応答と `Content-Type` を確認済み
