
## 本資料について

この資料は、プリザンターのCodeDefinerによるバージョンアップ処理で
SQL Serverのログインとデータベースユーザーの関連付けが競合している場合の対応方法や手順について、生成AIを使いながらまとめた資料です。

## 参考資料

[別サーバでバックアップしたファイルをリストアすると接続できない(SQL Server)](https://pleasanter.org/ja/manual/FAQ/backup-restore/faq-unable-login-after-restore/?utm_source=chatgpt.com)

## 通常の対応

プリザンターの公式マニュアルにも、リストア後のユーザーマッピングを修復する方法が掲載されています。</br>
ただし、今回のエラーでは単純な ALTER USER だけでは解決しない可能性があるため、先に現在の対応関係を確認することをおすすめします。

### 1. SQL Serverのログインとユーザーを確認

```sql
USE [Implem.Pleasanter];
SELECT
    dp.name AS DatabaseUser,
    dp.type_desc AS UserType,
    sp.name AS MappedLogin,
    dp.sid AS UserSID
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE dp.name LIKE 'Implem.Pleasanter%';
```
### 2. ログインが既に別ユーザーに割り当てられていないか確認

OwnerやUserのログインが想定外のデータベースユーザーに関連付けられていないかの確認。
```
USE [Implem.Pleasanter];
SELECT
    dp.name AS DatabaseUser,
    sp.name AS ServerLogin
FROM sys.database_principals dp
INNER JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE sp.name IN ( 
    'Implem.Pleasanter_Owner',
    'Implem.Pleasanter_User'
);
```

### 3. マッピング後の修正

競合がなく、単純なマッピング不整合であることを確認できた場合は、以下のSQLで修復できます。

```sql
USE [Implem.Pleasanter];
ALTER USER [Implem.Pleasanter_Owner] WITH LOGIN = [Implem.Pleasanter_Owner];
ALTER USER [Implem.Pleasanter_User] WITH LOGIN = [Implem.Pleasanter_User];
```

## 確認
- Implem.Pleasanter_Owner と Implem.Pleasanter_User が、それぞれ想定したログインに関連付いているか。
- 同じログインが、想定外のデータベースユーザーに関連付いていないか。

```sql
USE [Implem.Pleasanter];
SELECT
    dp.name AS DatabaseUser,
    dp.type_desc AS UserType,
    sp.name AS MappedLogin,
    dp.authentication_type_desc AS AuthType
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp
    ON dp.sid = sp.sid
WHERE dp.type = 'S'
  AND dp.principal_id > 4
ORDER BY dp.name;
```

