# Windows ServerへのTomcat導入・Apache連携手順書

Windows版のTomcatをインストールする手順書を作る際のベースとなる資料です。
生成AIにて作成した資料をベースに一部修正したものです。
この資料の利用は自由ですが、情報や資料について、ご利用の際の最終的な責任は負いかねます。

## 1. 概要

本手順書では、社内イントラネット上のWindows Serverに以下のソフトウェアを導入します。

- Eclipse Temurin JDK 25
- Apache Tomcat 11
- Apache HTTP Serverとのリバースプロキシ連携

JDKおよびTomcatについては、インストーラを使用せず、ZIPファイルを展開して導入します。
Apache HTTP Serverはすでにインストールされているものとします。

### 構成イメージ

```text
利用者のブラウザ
      |
      | HTTP / HTTPS
      v
Apache HTTP Server
  80 / 443
      |
      | HTTP（リバースプロキシ）
      v
Apache Tomcat
    8080
      |
      v
Java Webアプリケーション
```

利用者はTomcatの8080番ポートへ直接アクセスせず、Apacheを経由してWebアプリケーションを利用する構成とします。

---

# 2. 使用するディレクトリ

本手順書では、以下のディレクトリを使用します。

```text
C:\Java\jdk-25
C:\Tomcat
```

実際のJDKには、

```text
jdk-25.0.x+x
```

のようなバージョン番号が付いたフォルダ名が含まれます。

本手順では説明を簡単にするため、

```text
C:\Java\jdk-25
```

として説明します。

---

# 3. Temurin JDK 25のダウンロード

Eclipse Adoptiumの公式サイトからTemurin JDK 25をダウンロードします。

ダウンロード時には、以下を選択します。

```text
Version      : 25 - LTS
Operating System : Windows
Architecture : x64
Package Type : JDK
JVM          : HotSpot
```

インストーラを使用しないため、**ZIP形式**をダウンロードします。

社内サーバがインターネットへ接続できない場合は、インターネット接続可能なPCでダウンロードし、社内で認められた方法でWindows Serverへコピーします。

---

# 4. Temurin JDK 25の配置

Windows Server上に以下のフォルダを作成します。

```text
C:\Java
```

ダウンロードしたZIPファイルを展開します。

最終的に次のような構成にします。

```text
C:\Java\jdk-25
    ├─ bin
    ├─ conf
    ├─ include
    ├─ jmods
    ├─ legal
    └─ lib
```

特に、

```text
C:\Java\jdk-25\bin\java.exe
```

が存在することを確認してください。

---

# 5. JAVA_HOMEの設定

JDKをZIP展開しただけでは、WindowsにJavaの場所が登録されませんので、環境変数の設定を行います。

## (1) JAVA_HOME

「システムのプロパティ」から、

```text
詳細設定
  ↓
環境変数
  ↓
システム環境変数
```

を開きます。

以下を追加します。

```text
変数名:
JAVA_HOME

変数値:
C:\Java\jdk-25
```

## (2) PATH

システム環境変数の `Path` に以下を追加します。

```text
%JAVA_HOME%\bin
```

設定後、新しいコマンドプロンプトを起動します。

次のコマンドを実行します。

```cmd
java -version
```

Temurin 25のバージョン情報が表示されれば成功です。

続いて、

```cmd
echo %JAVA_HOME%
```

を実行します。

以下のように表示されることを確認します。

```text
C:\Java\jdk-25
```

---

# 6. Tomcatのダウンロード

Apache Tomcat公式サイトからTomcat 11をダウンロードします。
https://tomcat.apache.org/download-11.cgi

Windows向けには複数のファイルがありますが、本手順ではインストーラを使用しないため、

```text
Binary Distributions
  Core
    Windows zip
```

をダウンロードします。

「Windows Service Installer」ではなく、**Windows zip**を選択してください。

---

# 7. Tomcatの配置

Windows Server上に以下のフォルダを作成します。

```text
C:\Tomcat
```

ダウンロードしたTomcatのZIPファイルを展開します。

最終的に次のような構成にします。

```text
C:\Tomcat
    ├─ bin
    ├─ conf
    ├─ lib
    ├─ logs
    ├─ temp
    ├─ webapps
    └─ work
```

次のファイルが存在することを確認してください。

```text
C:\Tomcat\bin\startup.bat
C:\Tomcat\bin\shutdown.bat
C:\Tomcat\bin\service.bat
C:\Tomcat\conf\server.xml
```

---

# 8. CATALINA_HOMEの設定

システム環境変数に以下を追加します。

```text
変数名:
CATALINA_HOME

変数値:
C:\Tomcat
```

新しいコマンドプロンプトを開き、

```cmd
echo %CATALINA_HOME%
```

を実行します。

以下のように表示されれば問題ありません。

```text
C:\Tomcat
```

---

# 9. Tomcatの単体起動確認

Apacheと連携する前に、Tomcatだけで正常に起動することを確認します。

コマンドプロンプトを開きます。

```cmd
cd /d C:\Tomcat\bin
```

次のコマンドを実行します。

```cmd
startup.bat
```

Tomcatが起動します。

Windows Server上のブラウザから、

```text
http://localhost:8080/
```

へアクセスします。

Tomcatの初期画面が表示されれば、Tomcat単体の起動は成功です。

## 停止方法

```cmd
cd /d C:\Tomcat\bin
shutdown.bat
```

で停止できます。

---

# 10. TomcatをWindowsサービスとして登録する

Windows Serverの起動時にTomcatも自動的に起動できるよう、Windowsサービスとして登録します。

**管理者としてコマンドプロンプトを起動**します。

以下を実行します。

```cmd
cd /d C:\Tomcat\bin
service.bat install Tomcat11
```

正常に終了したら、

```text
services.msc
```

を実行し、「サービス」を開きます。

Tomcatのサービスが登録されていることを確認します。サービスの「スタートアップの種類」は、必要に応じて

```text
自動
```

に設定します。

Tomcatを開始し、

```text
http://localhost:8080/
```

へアクセスして正常に表示されることを確認します。

## サービスを削除する場合

管理者権限のコマンドプロンプトから、

```cmd
cd /d C:\Tomcat\bin
service.bat remove Tomcat11
```

を実行します。

---

# 11. ApacheとTomcatの連携方式

本手順ではApacheからTomcatへの接続に、HTTPリバースプロキシを使用します。

例えば利用者が、

```text
http://server-name/myapp/
```

へアクセスすると、Apacheが内部的に、

```text
http://127.0.0.1:8080/myapp/
```

へリクエストを転送します。

利用者からはTomcatの8080番ポートを意識する必要がありません。

---

# 12. Apacheのmod_proxyを確認する

Apacheの `httpd.conf` を開きます。

Apacheのインストール先によって場所は異なりますが、例えば以下です。

```text
C:\Apache24\conf\httpd.conf
```

以下の設定を探します。

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

行頭に `#` が付いている場合は、

```apache
#LoadModule proxy_module modules/mod_proxy.so
#LoadModule proxy_http_module modules/mod_proxy_http.so
```

となっています。

この場合は `#` を削除します。

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

これによりHTTPリバースプロキシが使用できるようになります。

---

# 13. ApacheからTomcatへ転送する

例として、Tomcat上に

```text
/myapp
```

というWebアプリケーションを配置するとします。

Apacheの設定ファイルに以下を追加します。

```apache
ProxyRequests Off

ProxyPass        "/myapp/" "http://127.0.0.1:8080/myapp/"
ProxyPassReverse "/myapp/" "http://127.0.0.1:8080/myapp/"
```

これにより、

```text
http://server-name/myapp/
```

へのアクセスが、

```text
http://127.0.0.1:8080/myapp/
```

へ転送されます。

`ProxyRequests Off` は、Apacheを一般利用可能なフォワードプロキシとして動作させないための設定です。

---

# 14. Apacheの設定チェック

Apacheを再起動する前に、設定ファイルに間違いがないか確認します。

Apacheの `bin` ディレクトリへ移動します。

例：

```cmd
cd /d C:\Apache24\bin
```

以下を実行します。

```cmd
httpd.exe -t
```

正常であれば、

```text
Syntax OK
```

と表示されます。

エラーが表示された場合は、Apacheを再起動せず、先に設定ファイルを修正してください。

---

# 15. Apacheの再起動

ApacheをWindowsサービスとして登録している場合は、「サービス」画面からApacheを再起動します。

```text
services.msc
```

を実行してApacheのサービスを探し、

```text
再起動
```

します。

---

# 16. Apache経由での動作確認

Tomcatにアプリケーション `myapp` が配置されている場合、クライアントPCから以下へアクセスします。

```text
http://server-name/myapp/
```

アプリケーションが表示されれば、ApacheとTomcatの連携は成功です。

このとき利用者は、

```text
http://server-name:8080/myapp/
```

ではなく、

```text
http://server-name/myapp/
```

へアクセスします。

---

# 17. Tomcatの8080番ポートを外部公開しない

ApacheとTomcatを同じWindows Server上で動かす場合、Tomcatの8080番ポートを他のPCへ公開する必要はありません。

`C:\Tomcat\conf\server.xml` のHTTP Connectorを確認します。

通常は次のような設定があります。

```xml
<Connector port="8080"
           protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

ApacheとTomcatが同一サーバにあり、Tomcatへの直接アクセスを不要とする場合は、必要に応じて次のようにlocalhostのみに待ち受けさせます。

```xml
<Connector port="8080"
           address="127.0.0.1"
           protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

変更後はTomcatを再起動します。

これにより、

```text
Apache
   ↓
127.0.0.1:8080
   ↓
Tomcat
```

というサーバ内部だけの通信にできます。

---

# 18. Webアプリケーションの配置

WARファイル形式のJava Webアプリケーションの場合、基本的にはTomcatの

```text
C:\Tomcat\webapps
```

へ配置します。

例えば、

```text
myapp.war
```

を配置すると、

```text
C:\Tomcat\webapps\myapp.war
```

となります。

Tomcat起動後、通常は

```text
http://localhost:8080/myapp/
```

としてアクセスできます。

Apache側に前述のProxyPassを設定していれば、

```text
http://server-name/myapp/
```

としてアクセスできます。

---

# 19. ログの確認場所

問題が発生した場合は、最初にログを確認します。

## Tomcat

```text
C:\Tomcat\logs
```

主に以下のログを確認します。

```text
catalina.YYYY-MM-DD.log
localhost.YYYY-MM-DD.log
```

アプリケーションによっては独自のログファイルが作成されます。

## Apache

Apacheのインストール先によりますが、一般的には、

```text
C:\Apache24\logs
```

にあります。

主に、

```text
error.log
access.log
```

を確認します。

---

# 20. トラブル発生時の確認方法

接続できない場合は、いきなりApacheやアプリケーションの設定を変更せず、以下の順番で確認します。

1. Javaが動作するか
2. Tomcatが起動するか
3. localhost:8080へアクセスできるか
4. アプリケーションがTomcat単体で動作するか
5. Apacheが起動するか
6. Apache → TomcatのProxyPassが動作するか
7. クライアントPCからApacheへアクセスできるか
```

### Java確認

```cmd
java -version
```

### JAVA_HOME確認

```cmd
echo %JAVA_HOME%
```

### Tomcat確認

```text
http://localhost:8080/
```

### アプリケーション確認

```text
http://localhost:8080/myapp/
```

### Apache設定確認

```cmd
httpd.exe -t
```

### Apache経由

```text
http://server-name/myapp/
```

この順番で確認すると、問題が

- Java
- Tomcat
- Java Webアプリケーション
- Apache
- Apache/Tomcat連携
- ネットワーク

のどこにあるのか判断しやすくなります。

---

# 21. 導入後の確認チェックリスト

## JDK

- [ ] Temurin JDK 25をZIP形式で入手した
- [ ] `C:\Java\jdk-25` に展開した
- [ ] `JAVA_HOME` を設定した
- [ ] `%JAVA_HOME%\bin` をPATHへ追加した
- [ ] `java -version` が正常に実行できた

## Tomcat

- [ ] Tomcat 11 Windows ZIPを入手した
- [ ] `C:\Tomcat` に展開した
- [ ] `CATALINA_HOME` を設定した
- [ ] `startup.bat` で起動できた
- [ ] `http://localhost:8080/` を表示できた
- [ ] Windowsサービスとして登録した
- [ ] サービスからTomcatを起動できた

## Apache

- [ ] `mod_proxy` が有効になっている
- [ ] `mod_proxy_http` が有効になっている
- [ ] `ProxyRequests Off` を設定した
- [ ] `ProxyPass` を設定した
- [ ] `ProxyPassReverse` を設定した
- [ ] `httpd.exe -t` で `Syntax OK` になった
- [ ] Apacheを再起動した

## 最終確認

- [ ] Tomcat単体でアプリケーションを表示できた
- [ ] Apache経由でアプリケーションを表示できた
- [ ] 必要に応じてTomcatの8080番ポートをlocalhost限定にした
- [ ] Tomcatのログの場所を確認した
- [ ] Apacheのログの場所を確認した

---

# 22. 構成まとめ

最終的な構成は以下のようになります。

```text
Windows Server

C:\Java\jdk-25
       │
       │ JAVA_HOME
       v
C:\Tomcat
       │
       │ HTTP :8080
       │ 127.0.0.1のみ
       │
       ^
       │ mod_proxy / mod_proxy_http
       │
Apache HTTP Server
       ^
       │ HTTP :80
       │ HTTPS :443（設定している場合）
       │
社内クライアントPC
```

Apacheをフロントエンド、Tomcatをバックエンドとして動作させることで、利用者はTomcatのポート番号を意識せずWebアプリケーションを利用できます。