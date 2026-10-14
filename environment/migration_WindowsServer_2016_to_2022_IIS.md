# Windows Server 2016 → Windows Server 2022 IIS移行手順書

## 0. はじめに(前提条件)

本資料は生成AIを使って作成した資料です。

**運用上の基本原則**：小規模台数の移行を前提としています。本資料の例では3台規模としています。そのため、高機能な専用ツールは作らず、標準コマンドの出力とIISマネージャーで確認し、開発機で実証した手順を本番へ適用する。移行元は最終コピー時まで稼働可能な状態に保ち、切替後は停止・保持するとしています。

- 版数：1.0（案）
- 対象：開発サーバ2台、本番サーバ1台
- 対象アプリ：ASP.NET Web Forms（.NET Framework 3.5および4.x）
- 移行方法：IIS設定を確認して再構築、robocopyで事前コピー・当日差分コピー

> **注意**：以下のサーバ名・ドライブ・パスは例です。実環境に合わせて変更してください。まず開発機でコマンドと操作を検証し、確認結果を手順書に反映してから本番移行を実施します。

## 1. 目的と基本方針

Windows Server 2016で稼働しているIIS上のASP.NET Web Formsアプリケーションを、Windows Server 2022へ移行する。アプリケーションの機能変更や.NET Frameworkの同時更新は原則実施しない。

| 項目 | 方針 |
|---|---|
| 移行元／移行先 | Windows Server 2016 → 2022（IIS 10.0 → IIS 10.0） |
| 台数と順序 | 開発機①で手順確認 → 開発機②で再現検証 → 本番機 |
| IIS設定 | appcmdで構成を取得し、移行先で必要な設定を再構築 |
| アプリファイル | robocopyで事前コピーし、移行当日に差分コピー |
| コピー時の削除 | `/MIR`・`/PURGE`は使用しない（移行先ファイルを削除しない） |
| Web Deploy | 使用しない |
| 移行元サーバ | 移行後は停止し、一定期間保持（本手順に削除作業なし） |
| 切り戻し | 旧サーバ再開と接続先切替。ただし移行後のデータ更新を必ず確認する |

## 2. 移行スケジュールと役割

1. **開発機①（初回検証）**：既存構成を調査し、IIS再構築・ファイルコピー・動作確認を実施する。問題点と修正した手順を記録する。
2. **開発機②（再現検証）**：修正済みの手順を使い、別サーバでも再現可能か、差分コピーと切替・切り戻しまで確認する。
3. **本番機（事前作業）**：IIS構築、初回コピー、切替前の限定的な動作確認を行う。
4. **本番機（移行当日）**：旧環境の更新停止、差分コピー、最終確認、接続先切替を行う。
5. **移行後**：監視・業務確認を行い、旧サーバは停止した状態で保持する。

## 3. 事前確認・移行元の調査

### 3.1 調査結果の保存場所

移行元サーバで管理者権限の**コマンドプロンプト（cmd.exe）**を開く。

```cmd
mkdir C:\Temp\IISMigration
cd /d %windir%\system32\inetsrv

appcmd list site /config > C:\Temp\IISMigration\sites.txt
appcmd list apppool /config > C:\Temp\IISMigration\apppools.txt
appcmd list app /config > C:\Temp\IISMigration\apps.txt
appcmd list vdir /config > C:\Temp\IISMigration\vdirs.txt
```

**PowerShell（管理者）**で導入済み機能を取得する。

```powershell
Get-WindowsFeature |
    Where-Object Installed |
    Out-File 'C:\Temp\IISMigration\features.txt' -Encoding UTF8
```

- コマンドの実行エラーや空の出力がないか確認する。
- IISマネージャーと突き合わせ、サイト数、アプリケーション、アプリプール、物理パス、バインドを照合する。
- 出力には機密情報が含まれる場合がある。保存先・共有先のアクセス権を制限する。
- IIS設定の一覧だけでは、すべての依存コンポーネントや権限は把握できない。

### 3.2 手動確認項目

| 分類 | 確認事項 |
|---|---|
| サイト | サイト名、ホスト名、ポート、HTTP/HTTPS、物理パス |
| アプリプール | CLR v2.0/v4.0、Integrated/Classic、32bit、実行ID |
| アプリ | 仮想ディレクトリ、アプリケーションへの変換状況 |
| 認証 | 匿名認証、Windows認証、その他の利用方式 |
| 証明書 | HTTPS証明書、秘密鍵、更新期限、バインド・SNI |
| OS/IIS機能 | ASP.NET 3.5/4.x、ISAPI、必要な役割サービス |
| 依存関係 | COM、32bit DLL、ODBC/OLE DB、追加IISモジュール、URL Rewrite等 |
| 権限 | フォルダNTFS権限、アプリプールID、共有先・DBへのアクセス権 |
| 運用 | タスクスケジューラ、バッチ、ログ、アップロード先、バックアップ |
| 設定 | web.config、外部設定、接続文字列、環境別の値 |

移行先で利用するアプリ実行用アカウント、証明書のエクスポート方法、DB・外部共有への通信許可も事前確認する。

### 3.3 バックアップ

- 移行元サーバの既存バックアップが正常に取得されていることを確認する。
- IIS構成の保全用に、必要なら移行元で次を実行する（**このバックアップを別OSへ丸ごと復元する手順ではない**）。

```cmd
cd /d %windir%\system32\inetsrv
appcmd add backup BeforeMigration
appcmd list backup
```

- アプリファイル、設定ファイル、DBなどの復旧手段を確認する。

## 4. 移行先Windows Server 2022の準備

### 4.1 IISと.NET Framework 4.x関連機能

移行元の`features.txt`に基づいて必要な役割・機能を有効にする。以下はASP.NET 4.x向けの例（PowerShell管理者）。不要な機能は追加しない。

```powershell
Install-WindowsFeature `
    Web-Server, `
    Web-Asp-Net45, `
    Web-Net-Ext45, `
    Web-ISAPI-Ext, `
    Web-ISAPI-Filter `
    -IncludeManagementTools
```

Windows認証を使用する場合は`Web-Windows-Auth`も導入する。

### 4.2 .NET Framework 3.5とASP.NET 3.5

Windows Server 2022では.NET Framework 3.5を別途有効にする。
Windows Updateを利用できない場合は、**対応するWindows Server 2022のインストールメディア**を準備し、`D:`を実際のメディアドライブに読み替える。

```powershell
Install-WindowsFeature NET-Framework-Core -Source 'D:\sources\sxs'
```

続いてASP.NET 3.5用のIIS役割サービスを追加する。

```powershell
Install-WindowsFeature `
    Web-Asp-Net, `
    Web-Net-Ext, `
    Web-ISAPI-Ext, `
    Web-ISAPI-Filter
```

結果は次で確認する。

```powershell
Get-WindowsFeature NET-Framework-Core,Web-Asp-Net,Web-Net-Ext,Web-Asp-Net45,Web-Net-Ext45
```

**注意**：IISで「ASP.NET 3.5」と表示される機能と、アプリプールの「.NET CLR v2.0」は異なる名称だが、.NET Framework 3.5アプリを稼働させる場合は通常CLR v2.0を選択する。.NET Framework 4.xアプリはCLR v4.0を使用する。インストール後、再起動が必要と表示された場合は実施する。

### 4.3 IISサイトとアプリプールの再構築

IISマネージャーなどで、調査結果に沿って必要な構成を作成する。

1. アプリケーションプールを作成（CLR、パイプライン、32bit、実行IDを合わせる）。
2. サイトを作成（物理パス、ホスト名、ポートを合わせる）。
3. アプリケーション・仮想ディレクトリを作成する。
4. HTTP/HTTPSバインドを設定し、証明書と秘密鍵を適切に導入する。
5. 認証方式、必要なIISモジュールを設定する。
6. NTFS権限、共有アクセス、DB接続等を設定する。
7. アプリ起動時のエラーを確認する。

既存Server 2016の`applicationHost.config`をServer 2022に丸ごと上書きすることはしない。

## 5. アプリケーションファイルの初回コピー

移行先サーバで、管理者権限の**コマンドプロンプト（cmd.exe）**から実行する例。

**例の前提**：移行元`\\OLD-SERVER\D$\WebApps`、移行先`D:\WebApps`。先に`C:\Temp\IISMigration`を作成する。

```cmd
mkdir C:\Temp\IISMigration
robocopy \\OLD-SERVER\D$\WebApps D:\WebApps /E /COPY:DAT /DCOPY:T /R:2 /W:5 /LOG:C:\Temp\IISMigration\robocopy_initial.log
 echo RobocopyExitCode=%ERRORLEVEL%
```

- `/E`：空フォルダも含める。移行先にしかないファイルは削除しない。
- `/COPY:DAT`：データ、属性、タイムスタンプをコピー（NTFS ACLはコピーしない）。
- `/DCOPY:T`：ディレクトリのタイムスタンプをコピー。
- `/R:2 /W:5`：失敗時に2回再試行、5秒間隔。
- `/LOG:`：ログを出力。

**確認事項**

- robocopy終了コード：`0～7`はコピー失敗を示さず、`8以上`は失敗あり。ただし`0～7`でも差異・余剰ファイルの内容は確認する。
- ログ上の失敗ファイルがないこと。
- コピー元とコピー先のフォルダ構成・対象ファイルが一致すること。
- アプリプール等のNTFS権限が適切であること。
- アップロードファイル、ログ、キャッシュ、サーバ固有ファイル等をコピーするか除外するかを決めること。

**重要**：`/E`では移行元で削除済みのファイルが移行先に残る。移行先で作成したファイルも保護される代わりに、不要ファイルは差分確認して個別対応する。

## 6. 事前の動作試験

本番用DNS等は切り替えず、検証端末のhosts設定などにより、新サーバへ接続する（ホスト名依存、HTTPS証明書に留意）。

| 確認項目 | 判定 |
|---|---|
| サイト・アプリプールが起動する | □ OK / □ NG |
| .NET Framework 3.5対象アプリが動作する | □ OK / □ NG |
| .NET Framework 4.x対象アプリが動作する | □ OK / □ NG |
| 主要画面、画面遷移、入力検証 | □ OK / □ NG |
| Windows認証・ログイン | □ OK / □ NG |
| DB参照・更新（適切なテストデータのみ） | □ OK / □ NG |
| ファイルのアップロード・ダウンロード | □ OK / □ NG |
| 帳票、外部DLL・COM、32bit依存処理 | □ OK / □ NG |
| バッチ・タスク・外部連携（安全に試験） | □ OK / □ NG |
| IISログ・イベントログ・アプリログ | □ OK / □ NG |

事前試験で本番DBや外部システムに予期せぬ更新・通知を行わないよう、接続先と実行可否を調整する。開発機①で不具合を修正し、開発機②で手順を再現する。

## 7. 本番移行当日の作業

### 7.1 作業開始前の確認

- [ ] 作業開始・終了見込みを利用部門と共有した。
- [ ] 最新バックアップと復旧手段を確認した。
- [ ] 前回のコピー結果・未解決課題を確認した。
- [ ] DNS／IP／ロードバランサー等の切替方法・反映時間を確認した。
- [ ] 切り戻し判断者、連絡先、判断期限を決めた。
- [ ] 新サーバの環境別設定ファイル（web.config等）の退避・除外方針を確認した。

### 7.2 旧サーバの更新停止

1. 利用者への停止連絡とアクセス遮断を実施する。
2. バッチ、タスクスケジューラ、外部からの書込みを停止する。
3. 必要に応じてIISサイト・アプリプールを停止する。
4. アップロードやファイル更新などの処理が完了したことを確認する。
5. **旧サーバOS自体はシャットダウンしない**（最終コピーに必要）。

### 7.3 最終差分コピー

初回と同じコピー元・コピー先・オプションを使う。ログ名のみ変更する。

```cmd
robocopy \\OLD-SERVER\D$\WebApps D:\WebApps /E /COPY:DAT /DCOPY:T /R:2 /W:5 /LOG:C:\Temp\IISMigration\robocopy_final.log
 echo RobocopyExitCode=%ERRORLEVEL%
```

- 変更されたファイル等がコピーされることをログで確認する。
- 移行先用に変更した`web.config`などが上書きされないよう、コピー前に退避・除外し、コピー後に必要な値を再確認する。
- 移行元で削除したファイルは`/E`で移行先から消えないため、不要ファイルを手動で確認する。
- robocopyでカバーしないデータやDBについて、必要な同期・確認を行う。

### 7.4 最終確認と切替

1. robocopy終了コード・ログ・対象ファイルを確認する。
2. 必要に応じて新サーバのサイト・アプリプールを起動する。
3. 主要画面、認証、DB接続、ファイル操作を確認する。
4. DNS・IP・ロードバランサー等、決めた方式で接続先を切り替える。
5. 利用者端末からアクセス可能であることを確認する。
6. バッチ・タスク・監視等を新サーバ側で有効にする（旧環境との二重実行に注意）。
7. 作業完了と利用再開を連絡する。

DNSの場合、TTLやキャッシュにより旧サーバ宛の接続が残る可能性がある。切替後の二重更新が生じないよう、旧アプリの停止状態を維持する。

## 8. 切り戻し

切り戻し条件の例：主要画面が動かない、認証不可、重要業務処理で重大エラー、復旧に許容時間以上かかる。

1. 利用者に障害と切り戻しを連絡する。
2. 新サーバのアクセス・更新処理、バッチを停止する。
3. **新サーバ稼働後のDB・アップロードファイル等の更新有無を確認し、必要なら整合を取る。**
4. 旧サーバのIISサイト・バッチを再開する。
5. DNS・IP等の接続先を旧サーバへ戻す。
6. 主要機能とデータ整合性を確認し、利用再開を連絡する。

旧サーバを残していても、新サーバ稼働後の更新内容は自動的には旧サーバへ戻らない点に注意する。

## 9. 移行後の確認と旧サーバ保持

- [ ] 利用部門による主要業務の確認が完了した。
- [ ] IISログ、アプリログ、Windowsイベントログに重大な異常がない。
- [ ] 定期バッチ・外部連携が正常終了している。
- [ ] ファイル出力・アップロードとDB更新に問題がない。
- [ ] 新サーバのバックアップが正常に稼働している。
- [ ] 旧サーバは予定どおり停止して保持している。
- [ ] 保管期限、最終廃止の判断者を記録した。

## 10. 作業記録（各サーバ共通）

| 記録項目 | 記入欄 |
|---|---|
| 対象サーバ（開発①／開発②／本番） |  |
| 移行元／移行先ホスト名 |  |
| 実施日時・担当者 |  |
| 設定調査・バックアップ確認 |  |
| IIS・.NET Framework 3.5設定確認 |  |
| 初回コピーの結果・ログ |  |
| 動作確認結果・未解決事項 |  |
| 当日差分コピーの結果・ログ |  |
| 切替結果・利用確認 |  |
| 切り戻し要否／判断者 |  |

## 11. 参考資料（Microsoft公式）

- [Windows Serverのオンデマンド機能と.NET Framework 3.5の導入](https://learn.microsoft.com/ja-jp/windows-server/administration/server-manager/configure-features-on-demand-in-windows-server)
- [Windows ServerのIIS役割サービス一覧](https://learn.microsoft.com/en-us/windows-server/administration/server-core/server-core-roles-and-services)
- [IISの構成バックアップ（AppCmd）](https://learn.microsoft.com/ja-jp/iis/manage/managing-your-configuration-settings/shared-configuration_264)
- [robocopyのオプション・終了コード](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy)

---

