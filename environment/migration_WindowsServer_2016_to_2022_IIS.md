# Windows Server 2016 → Windows Server 2022 IIS移行手順書

- 版数：1.1（案）
- 対象：開発サーバ2台、本番サーバ1台
- 対象：ASP.NET Web Forms（.NET Framework 3.5 / 4.x）
- 方針：Web Deploy不使用。appcmd XMLでIISオブジェクト移行、robocopyでコンテンツを事前・最終差分コピー

> **重要**：例示するパス・サーバ名は適宜変更する。`appcmd /xml` → `/in` はIIS設定全体の完全複製ではない。既定サイトやルートアプリ等の重複を避け、開発サーバで検証してから本番で実行する。2016の `applicationHost.config` を2022へ丸ごと上書きしない。

## 0. はじめに(前提条件)

本資料は生成AIを使って作成した資料です。この資料の利用は自由ですが、情報や資料について、ご利用の際の最終的な責任は負いかねます。

**運用上の基本原則**：小規模台数の移行を前提としています。本資料の例では3台規模としています。
そのため、高機能な専用ツールは作らず、標準コマンドの出力とIISマネージャーで確認し、開発機で実証した手順を本番へ適用する。
移行元は最終コピー時まで稼働可能な状態に保ち、切替後は停止・保持するとしています。

## 1. 目的と基本方針

Windows Server 2016のIIS上で稼働するASP.NET Web FormsをWindows Server 2022へ移行する。アプリ機能や.NET Frameworkの同時改修は原則行わない。

| 項目 | 方針 |
|---|---|
| 順序 | 開発①で検証→開発②で再現確認→本番 |
| IIS設定 | appcmdのXML出力・入力を基本にし、適用されない設定を補完 |
| コンテンツ | robocopyで事前コピー＋切替当日の差分コピー |
| 削除 | `/MIR`・`/PURGE`は使用しない |
| 移行元 | 切替後は停止し、一定期間保持。削除作業なし |
| 切り戻し | 旧サーバ再開・接続先切替。ただし移行後の更新データの整合性を確認 |

## 2. 移行スケジュール

1. 開発①：設定採取、XML編集・インポート、ファイルコピー、動作試験を検証し、問題を記録する。
2. 開発②：修正した手順を再実行し、再現性と切り戻しを確認する。
3. 本番事前：IIS機能導入、設定移行、初回コピー、限定的な動作確認を実施する。
4. 本番当日：更新停止、最終差分コピー、確認、接続先切替。
5. 移行後：監視、利用部門確認、旧サーバ停止・保持。

## 3. 移行元の調査・バックアップ

### 3.1 IIS設定の採取

管理者の **cmd.exe** で実行する。

```cmd
mkdir C:\Temp\IISMigration
cd /d %windir%\system32\inetsrv
appcmd list site /config > C:\Temp\IISMigration\sites_config.txt
appcmd list apppool /config > C:\Temp\IISMigration\apppools_config.txt
appcmd list app /config > C:\Temp\IISMigration\apps_config.txt
appcmd list vdir /config > C:\Temp\IISMigration\vdirs_config.txt
appcmd list site /xml > C:\Temp\IISMigration\sites.xml
appcmd list apppool /xml > C:\Temp\IISMigration\apppools.xml
appcmd list app /xml > C:\Temp\IISMigration\apps.xml
appcmd list vdir /xml > C:\Temp\IISMigration\vdirs.xml
appcmd add backup BeforeMigration
appcmd list backup
```

`/config`は調査用、`/xml`はインポート候補の原本。**原本は保存し、登録用にはコピーを編集**する。バックアップは元サーバの復旧用であり、異OSへの復元には使わない。

PowerShell（管理者）：

```powershell
Get-WindowsFeature | Where-Object Installed |
  Out-File 'C:\Temp\IISMigration\features.txt' -Encoding UTF8
```

別途、`%windir%\System32\inetsrv\config\applicationHost.config` と必要なサイトの `web.config` を参照用に保管する。設定ファイルやXMLは秘密情報を含み得るためアクセス制限する。

### 3.2 個別確認項目

| 分類 | 確認事項 |
|---|---|
| サイト | 名称、ID、物理パス、ポート、ホスト名、HTTP/HTTPS、SNI |
| アプリプール | CLR v2.0/v4.0、パイプライン、32bit、実行ID、起動設定 |
| アプリ・仮想ディレクトリ | 階層、物理パス、接続資格情報 |
| IIS詳細 | 匿名/Windows認証、ハンドラ、モジュール、既定ドキュメント、MIME、要求制限、カスタムエラー、委任・ロック |
| 証明書 | 秘密鍵を含む証明書、バインド、更新期限 |
| 依存関係 | COM、DLL、ODBC/OLE DB、URL Rewrite等の追加モジュール |
| 権限・運用 | NTFS ACL、共有、DB、タスク、バッチ、ログ、アップロード先 |
| 環境差分 | 接続文字列、サービスアカウント、ホスト名、証明書、web.config |

## 4. 移行先Windows Server 2022の準備とIIS設定移行

### 4.1 IIS役割・.NET Frameworkの導入

移行元の`features.txt`に基づいて必要な機能を導入する。以下は例。

```powershell
Install-WindowsFeature Web-Server,Web-Asp-Net45,Web-Net-Ext45,Web-ISAPI-Ext,Web-ISAPI-Filter -IncludeManagementTools
```

Windows認証が必要なら`Web-Windows-Auth`も追加する。Framework 3.5のアプリがあるため、Windows Server 2022対応メディアの`D:\sources\sxs`を使う例：

```powershell
Install-WindowsFeature NET-Framework-Core -Source 'D:\sources\sxs'
Install-WindowsFeature Web-Asp-Net,Web-Net-Ext,Web-ISAPI-Ext,Web-ISAPI-Filter
Get-WindowsFeature NET-Framework-Core,Web-Asp-Net,Web-Net-Ext,Web-Asp-Net45,Web-Net-Ext45
```

必要なら再起動。Framework 3.5のアプリプールは通常CLR v2.0、Framework 4.xはCLR v4.0を使用する。

### 4.2 移行先の初期状態を保存

```cmd
mkdir C:\Temp\IISMigration
cd /d %windir%\system32\inetsrv
appcmd add backup BeforeImport
appcmd list site /xml > C:\Temp\IISMigration\target_sites_before.xml
appcmd list apppool /xml > C:\Temp\IISMigration\target_apppools_before.xml
appcmd list app /xml > C:\Temp\IISMigration\target_apps_before.xml
appcmd list vdir /xml > C:\Temp\IISMigration\target_vdirs_before.xml
```

### 4.3 XML登録前の整理（必須）

1. 移行元のXMLを移行先へコピーする。原本は編集せず、登録用XMLを別名で作る。
2. 移行先の既存オブジェクトと照合し、重複する`Default Web Site`、`DefaultAppPool`、ルートアプリ`/`、ルート仮想ディレクトリ`/`等を**登録用XMLから除外**する。既存オブジェクトを残す場合はその設定差分を別途反映する。
3. サイトID、サイト名、バインド、アプリプール名、親サイト・親アプリ、物理パスの整合性を確認する。
4. 移行先で利用できない設定値や実行IDを修正する。パスワード等の機密値はXML移行で再現できると仮定せず、個別に再設定する。
5. XMLが正しいこと、対象件数と除外理由を記録する。**全件を無条件に投入しない**。

### 4.4 登録（開発サーバで実証したXMLのみ）

登録順序はアプリプール→サイト→アプリ→仮想ディレクトリ。以下は登録用XMLを整理した後の例。

```cmd
cd /d %windir%\system32\inetsrv
appcmd add apppool /in < C:\Temp\IISMigration\apppools_import.xml
appcmd add site /in < C:\Temp\IISMigration\sites_import.xml
appcmd add app /in < C:\Temp\IISMigration\apps_import.xml
appcmd add vdir /in < C:\Temp\IISMigration\vdirs_import.xml
```

**注意**：親サイト作成時にルートアプリ・ルート仮想ディレクトリが既に作成される場合があるため、登録直前に状態を確認し重複分を除外する。コマンドがエラーになったら続行せず、対象を特定して修正する。XMLを投入するだけで認証・SSL・ハンドラ等の設定がすべて移行されるわけではない。

### 4.5 詳細設定の補完と比較

- `applicationHost.config`、`web.config`、`/config`出力を参照して、サイト/アプリ単位の認証、ハンドラ、モジュール、要求制限等を確認・設定する。
- 証明書と秘密鍵を適切に導入し、HTTPSバインドを確認する。
- アプリプールのカスタムID・パスワード、仮想ディレクトリの資格情報、NTFS ACL、DB/共有への権限を設定する。
- 移行先で`appcmd list ... /config`および`/xml`を再取得し、移行元の構成と比較する。サーバ固有の差分は記録する。
- 旧Server 2016の`applicationHost.config`は丸ごと上書きしない。

## 5. Webアプリケーションファイルの初回コピー

移行先の管理者cmdで実行する例。移行元`\\OLD-SERVER\D$\WebApps`、移行先`D:\WebApps`。

```cmd
mkdir C:\Temp\IISMigration
robocopy \\OLD-SERVER\D$\WebApps D:\WebApps /E /COPY:DAT /DCOPY:T /R:2 /W:5 /LOG:C:\Temp\IISMigration\robocopy_initial.log
echo RobocopyExitCode=%ERRORLEVEL%
```

- `/E`は空ディレクトリもコピーするが、移行先だけのファイルは削除しない。
- `/COPY:DAT`はデータ・属性・日時のみ。ACLは別途設定する。
- 終了コード`0～7`は失敗を示さず、`8以上`は失敗あり。`0～7`でもログを確認する。
- `web.config`等の環境固有設定、アップロード、ログ、キャッシュ、サーバ固有ファイルの除外・退避方針を決める。
- `/MIR`、`/PURGE`は使用しない。削除済みの古いファイルが残る場合は個別に確認する。

## 6. 事前の動作試験

本番DNSは変更せず、検証端末のhosts等で新サーバへ接続する。証明書名・ホスト名に注意。

| 確認項目 | 判定 |
|---|---|
| サイト・アプリプール起動 | □ OK / □ NG |
| Framework 3.5 / 4.xアプリ | □ OK / □ NG |
| 主要画面・画面遷移・表示内容 | □ OK / □ NG |
| 認証・ログイン | □ OK / □ NG |
| DB参照・更新（テストデータのみ） | □ OK / □ NG |
| アップロード・ダウンロード・帳票 | □ OK / □ NG |
| DLL/COM/32bit依存処理 | □ OK / □ NG |
| バッチ・外部連携 | □ OK / □ NG |
| IIS・イベント・アプリログ | □ OK / □ NG |
| 移行元との設定差分の説明・記録 | □ OK / □ NG |

事前試験で本番DBや外部システムに意図しない更新・通知を行わないこと。開発①で手順を修正し、開発②で再現する。

## 7. 本番移行当日

### 7.1 開始前確認

- [ ] 利用部門への連絡、作業時間、切り戻し判断者を確認
- [ ] 旧環境のバックアップと復旧手段を確認
- [ ] IIS設定比較・不足設定の解消を確認
- [ ] コピー除外・環境固有設定の退避を確認
- [ ] DNS/IP/ロードバランサー等の切替方式を確認

### 7.2 旧サーバの更新停止

1. 利用者アクセス・書込みを停止する。
2. バッチ・タスク・外部書込みを停止する。
3. 必要に応じて旧サイト/アプリプールを停止する。
4. 実行中処理とファイル更新が完了したことを確認する。
5. **最終コピーのため旧OSはまだシャットダウンしない**。

### 7.3 最終差分コピー

```cmd
robocopy \\OLD-SERVER\D$\WebApps D:\WebApps /E /COPY:DAT /DCOPY:T /R:2 /W:5 /LOG:C:\Temp\IISMigration\robocopy_final.log
echo RobocopyExitCode=%ERRORLEVEL%
```

ログ・終了コードを確認。移行先用`web.config`等が上書きされていないか確認し、必要な値を再適用する。`/E`では削除ファイルが残るため個別確認。DBや別保管データは必要に応じ別途同期する。

### 7.4 切替

1. コピー結果と設定を確認し、新サイト・アプリプールを起動する。
2. 主要画面、認証、DB、ファイル操作を確認する。
3. DNS/IP等の接続先を切り替える。
4. 利用者端末で動作確認する。
5. 新サーバ側のバッチ・監視を有効化する（旧側との二重実行を防止）。
6. 利用再開を連絡する。

DNSキャッシュ等で旧サーバへの接続が残る場合がある。旧アプリは停止状態を維持する。

## 8. 切り戻し

重大な業務エラー、認証不可、許容時間内に復旧不能等の場合：

1. 障害・切り戻しを連絡し、新サーバのアクセス・更新・バッチを停止する。
2. 新サーバ稼働後のDB・アップロードファイル等の更新を確認し、必要な整合性対処を行う。
3. 旧サーバのサイト・バッチを再開する。
4. 接続先を旧サーバに戻す。
5. 業務機能・データ整合性を確認し利用再開を連絡する。

旧サーバを残しても新サーバでの更新は自動的に戻らない。

## 9. 移行後の確認・旧サーバ保持

- [ ] 利用部門による主要業務確認完了
- [ ] IIS・イベント・アプリログに重大異常なし
- [ ] 定期バッチ・外部連携正常
- [ ] ファイル・DB更新正常
- [ ] 新サーバのバックアップ正常
- [ ] 旧サーバは停止して保持（削除しない）
- [ ] 保持期限・廃止判断者を記録

## 10. 作業記録

| 記録項目 | 記入欄 |
|---|---|
| 対象（開発①／開発②／本番） | |
| 移行元／先ホスト名 | |
| 日時・担当者 | |
| IIS XML原本・登録用XMLの保管場所 | |
| XML登録件数・除外理由・エラー | |
| IIS詳細設定の差分・補完内容 | |
| Framework 3.5・IIS機能確認 | |
| 初回・最終コピーのログと結果 | |
| 試験結果・未解決事項 | |
| 切替・切り戻し判断 | |

## 11. 参考資料

- [Microsoft Learn: AppCmd.exeによるIIS管理](https://learn.microsoft.com/en-us/iis/get-started/getting-started-with-iis/getting-started-with-appcmdexe)
- [Microsoft Learn: IIS構成の管理](https://learn.microsoft.com/en-us/iis/manage/managing-your-configuration-settings/)
- [Microsoft Learn: Windows Serverの機能オンデマンド](https://learn.microsoft.com/en-us/windows-server/administration/server-manager/configure-features-on-demand-in-windows-server)
- [Microsoft Learn: robocopy](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy)

---

**基本原則**：専用ツールを作らず、標準コマンドとIISマネージャーで確認する。XMLの一括登録は開発サーバで実証し、重複オブジェクトと未移行の詳細設定を整理した上で本番へ適用する。
