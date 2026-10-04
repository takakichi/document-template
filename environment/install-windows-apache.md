# Windows ServerへのApache HTTP Server導入手順

Windows版のApacheをインストールする手順書を作る際のベースとなる資料です。
生成AIにて作成した資料をベースに一部修正したものです。
この資料の利用は自由ですが、情報や資料について、ご利用の際の最終的な責任は負いかねます。

## 1. 概要

本テキストは、Windows Serverに **Apache HTTP Server 2.4** を導入するための手順です。

### 1.1 想定環境

想定する環境は次のとおりです。

- Windows Server 2022 (64bit版)
- Apache HTTP Server 2.4
- Apache本体の配置先：`C:\Apache24`
- HTTPポート：`80`

Apache 本体についてはインストーラを使用せず、Apache Loungeから提供されているWindows向けZIPファイルを展開して使用することを前提とします。

## 2. 導入手順

最低限動作するための Apache の導入手順は次の手順になります。

1. Apache LoungeからWindows版Apacheをダウンロード
2. 必要なVisual C++ Redistributableを確認
3. C:\Apache24 にZIPを展開
4. conf\httpd.confを確認・設定
5. httpd.exe -t で設定チェック
6. httpd.exe で手動起動
7. http://localhost/ で動作確認
8. htdocsへテストページを配置して確認
9. httpd.exe -k install -n "Apache24" でサービス登録
10. Windowsサービスとして起動

## 3. Apacheのダウンロード・配置・設定

### 3.1 Apacheのダウンロード

Apache LoungeのダウンロードページからWindows版Apacheをダウンロードします。

**Apache Lounge - Apache Downloads**

https://www.apachelounge.com/download/

Apache Loungeのダウンロードページから、Windows 64bit版（Win64）のApacheをダウンロードします。
通常の64bit版Windows Serverであれば、**Win64版**を選択します。
ダウンロードするファイルはZIP形式としてください。

- Win64
- Apache 2.4.x
- ZIP形式

バージョン番号やファイル名はリリースによって変わるため、最新もしくは指定された安定版を選択してください。

例：

```text
httpd-2.4.xx-xxxx-Win64-VS17.zip
```

---

### 3.2 Visual C++ Redistributableの確認

Apache Loungeで配布されているApacheは、Microsoft Visual C++ Runtimeを使用します。
必要なVisual C++ Redistributableがサーバにインストールされていることを確認し、インストールされていない場合は導入します。

**Microsoft Visual C++ Redistributable 最新サポート版**

https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist/

64bit版Windows ServerでWin64版Apacheを使用する場合は、基本的にx64版が対象です。

Visual C++ Redistributableがすでにサーバへ導入されている場合は、改めてインストールする必要がない場合があります。

> **注意**
>
> Apache本体はZIP展開だけで導入できますが、必要なVisual C++ Redistributableがサーバに存在しない場合は、別途Visual C++ Redistributableの導入が必要です。

---

### 3.3 Apacheの配置と初期設定

#### 3.3.1 ZIPファイルの展開

ダウンロードしたZIPファイルを展開します。展開したファイルの中に `Apache24` フォルダがあります。
これを次の場所に配置します。

```text
C:\Apache24
```

配置後は、おおむね次のような構成になります。

```text
C:\Apache24
│
├─ bin
│   └─ httpd.exe
├─ conf
│   └─ httpd.conf
├─ htdocs
├─ logs
└─ modules
```

主なフォルダの用途は次のとおりです。

| フォルダ | 用途 |
|---|---|
| `bin` | Apacheの実行ファイル |
| `conf` | Apacheの設定ファイル |
| `htdocs` | Webページを配置する場所 |
| `logs` | Apacheのログ |
| `modules` | Apacheの追加モジュール |

Apache本体の実行ファイルは次のファイルです。

```text
C:\Apache24\bin\httpd.exe
```

---

### 3.3.2. Apache 基本設定の確認

Apacheの主な設定は次のファイルで行います。

```text
C:\Apache24\conf\httpd.conf
```

メモ帳などのテキストエディタで `httpd.conf` を開きます。

#### (1) ServerRoot

Apacheを `C:\Apache24` に配置した場合、次のようになっていることを確認します。

```apache
Define SRVROOT "c:/Apache24"

ServerRoot "${SRVROOT}"
```

Windowsのパスであっても、Apacheの設定ファイルでは `/` を使用します。

#### (2) ポート番号

HTTPの標準ポートである80番を使用する場合は、次の設定を確認します。
IISなどの別のWebサービス(サーバ)を実行している場合は、別のポート(例えば8000)にしてください。

- 変更前
```apache
Listen 80
```

- 変更後
```apache
Listen 8000
```

#### (3) ServerName

`ServerName` を設定します。

初期確認では次の設定でも構いませんが、すでにサーバ名が決まっている場合は、変更してください。
ServerName には、このApacheサーバへアクセスする際に使用するサーバ名またはDNS名を指定します。
Windows Serverのコンピュータ名を使用する場合は、そのサーバ名を指定します。(下記例ではWIN_SVR_NAME)

- 変更前
```apache
ServerName localhost:80
```
- 変更後
```apache
ServerName WIN_SVR_NAME:8000
```

#### (4) DocumentRoot

Webページを配置する場所を確認します。標準では次のようになります。
なお、本手順ではディレクトリ一覧を表示しない設定とします。

```apache
DocumentRoot "${SRVROOT}/htdocs"

<Directory "${SRVROOT}/htdocs">
    Options FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

この場合、

```text
C:\Apache24\htdocs
```

に配置したHTMLファイルがApacheから公開されます。

---

### 3.3.3. Apacheの設定ファイルをチェックする

Apacheを起動する前に、設定ファイルに誤りがないか確認します。管理者権限でコマンドプロンプトを起動します。
次のコマンドを実行します。

```cmd
cd /d C:\Apache24\bin
httpd.exe -t
```

設定に問題がなければ次のように表示されます。

```text
Syntax OK
```

エラーが表示された場合は、Apacheを起動する前に `httpd.conf` の設定を修正します。
今後 `httpd.conf` を変更した場合も、
```cmd
httpd.exe -t
```
を実行してからApacheを再起動することを推奨します。

---

### 3.3.4. Apacheの起動確認

#### (1) 手動起動

最初からWindowsサービスとして登録せず、まずApacheが正常に動作することを確認します。
管理者権限のコマンドプロンプトで次のコマンドを実行します。

```cmd
cd /d C:\Apache24\bin
httpd.exe
```

正常に起動すると、Apacheが動作した状態になります。このコマンドプロンプトは閉じないでください。

#### (2) ブラウザから確認する

サーバ上のブラウザから次のアドレスへアクセスします。(ポート番号は設定したポート番号にしてください)

ポート80を設定した場合：
```text
http://localhost/
```
ポート8000を設定した場合：
```text
http://localhost:8000/
```

Apacheの初期ページが表示されれば、Apacheは正常に動作しています。

#### (3) Apacheを停止する

手動起動している場合は、Apacheを実行しているコマンドプロンプトで、

```text
Ctrl + C
```

を押して終了します。

### 3.3.5. Webコンテンツを配置する

Apacheの標準設定では、Webコンテンツの配置場所は次のフォルダです。

```text
C:\Apache24\htdocs
```

動作確認用として、例えば次のファイルを作成します。

```text
C:\Apache24\htdocs\test.html
```

内容を次のようにします。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>Apache Test</title>
</head>
<body>
    <h1>Apache Test</h1>
    <p>Apache HTTP Server is running.</p>
</body>
</html>
```

Apacheを起動して、ブラウザから次のアドレスへアクセスします。

ポート80を設定した場合：
```text
http://localhost/test.html
```
ポート8000を設定した場合：
```text
http://localhost:8000/test.html
```

作成したページが表示されれば、Webコンテンツの配置とApacheの基本動作は正常です。


### 3.3.6. ApacheをWindowsサービスとして登録

手動起動で正常に動作することを確認したら、ApacheをWindowsサービスとして登録します。

これにより、Windows Serverの起動時にApacheを自動的に起動できるようになります。

#### (1) サービスの登録

管理者権限のコマンドプロンプトを開きます。

次のコマンドを実行します。

```cmd
cd /d C:\Apache24\bin
httpd.exe -k install -n "Apache24"
```

正常に登録されれば、Windowsサービスとして `Apache24` が作成されます。

Windowsの「サービス」を開き、

```text
Apache24
```

が登録されていることを確認します。

#### (2) サービスを起動する

次のコマンドでApacheを起動できます。

```cmd
httpd.exe -k start -n "Apache24"
```

または、

```cmd
net start Apache24
```

でも起動できます。

#### (3) サービスを停止する

```cmd
httpd.exe -k stop -n "Apache24"
```

または、

```cmd
net stop Apache24
```

を実行します。

#### (4) Apacheを再起動する

`httpd.conf` を変更した場合などは、最初に設定を確認します。

```cmd
httpd.exe -t
```

`Syntax OK` であることを確認してから、Apacheを再起動します。

```cmd
httpd.exe -k restart -n "Apache24"
```

#### (5) Apacheの自動起動を確認する

Windowsの「サービス」を開き、次の内容を確認します。

- サービス名：Apache24
- スタートアップの種類：自動
- 状態：実行中

その後、Windows Server再起動したうえで、次のことを確認してください。

- Apache24サービスが自動起動している
- ブラウザから正常表示することを確認

ポート80を設定した場合：
```text
http://localhost/test.html
```
ポート8000を設定した場合：
```text
http://localhost:8000/test.html
```

### 3.3.7 サービス登録を解除する

ApacheのWindowsサービス登録を解除する必要がある場合は、次のコマンドを実行します。

```cmd
httpd.exe -k uninstall -n "Apache24"
```

---

## 4. トラブル発生時の確認方法

Apacheが起動しない場合は、まず設定ファイルを確認します。

```cmd
cd /d C:\Apache24\bin
httpd.exe -t
```

次にApacheのエラーログを確認します。

```text
C:\Apache24\logs\error.log
```

アクセス状況については次のログを確認します。

```text
C:\Apache24\logs\access.log
```

Apacheが起動しない場合は、基本的に次の順番で確認します。

1. `httpd.exe -t` を実行する
2. `Syntax OK` になるか確認する
3. `C:\Apache24\logs\error.log` を確認する
4. Apacheを手動で起動してエラーが表示されないか確認する
5. 設定したポート番号が他のWebサーバなどで使用されていないか確認する
6. 問題が解決してからWindowsサービスを起動する

---

## 5. 参考サイト

### Apache Lounge

Windows向けApache HTTP Serverのバイナリを配布しています。

https://www.apachelounge.com/download/

### Apache HTTP Server公式ドキュメント

Windows上でApacheを使用する場合の公式ドキュメントです。

https://httpd.apache.org/docs/current/platform/windows.html

ApacheのWindowsサービスへの登録方法、起動・停止方法なども説明されています。

### Apache HTTP Server 2.4 Documentation

Apache HTTP Server全般の公式ドキュメントです。

https://httpd.apache.org/docs/2.4/

### Microsoft Visual C++ Redistributable

Visual C++ RedistributableのMicrosoft公式ページです。

https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist/

Apache Lounge版Apacheが必要とするVisual C++ Runtimeを確認・導入する際に使用します。

