# 社内システム データベース運用要領書

生成AIを使って作成した要領書です。

**版数:** 1.0  
**作成日:** 2026年02月10日  
**対象システム:** イントラネット社内システム データベース

---

## 1. 目的
本ドキュメントは、イントラネット上で稼働する社内システムのデータベース（以下、DB）について、安定稼働の維持および障害時の迅速な復旧を目的とした運用管理業務を定めるものである。

## 2. システム環境・前提条件

* **DBMS:** Microsoft SQL Server
* **サーバー構成:** シングル構成（1台）
* **認証方式:**
    * **管理者:** Windows認証（Active Directory）
    * **システム接続:** SQL Server認証
    * **一般利用者:** アプリケーション側で制御（DB上のアカウント管理は行わない）
* **監視範囲の定義:**
    * **対象:** DBサービスの死活、ジョブ実行結果、バックアップ取得状況、DB内部のエラーログ。
    * **対象外:** CPU使用率、ディスク容量、メモリ使用率（※全社共通基盤にて別途監視するため）。

---

## 3. 運用体制

| 役割 | 担当 | 責務 |
| :--- | :--- | :--- |
| **運用責任者** | 課長 | 運用の統括、障害時の判断、承認 |
| **運用担当者** | [担当チーム名] | 日常点検、定期メンテナンス、一次対応 |

* **アクセス権限:** 運用担当者は、原則として自身のADアカウントにてサーバーOSへログオンし、作業を行う。

---

## 4. 日常運用業務（Daily Operations）

毎朝の始業時に実施する。

### 4.1. サービス稼働確認
* **確認対象:**
    1.  SQL Server (MSSQLSERVER) … **実行中**であること
    2.  SQL Server Agent … **実行中**であること
* **手順:** `SQL Server Configuration Manager` または PowerShell にて確認。

### 4.2. ジョブ実行結果の確認
* **確認対象:**
    * 夜間バックアップジョブ
    * 定期メンテナンスジョブ
* **判定基準:** ステータスが「成功」であること。
* **手順:** SSMS > SQL Server エージェント > [ジョブ活動モニター] にて確認。

### 4.3. エラーログ確認
* **確認対象:** SQL Server エラーログ
* **判定基準:**
    * Severity（重大度）17以上のエラーが出ていないか。
    * ログイン失敗（Login failed）が多発していないか。

---

## 5. 定期メンテナンス業務（Periodic Maintenance）

### 5.1. バックアップ運用
シングル構成のため、**「外部へのデータ待避」**を最重要事項とする。

* **完全バックアップ:** 毎日 1回（夜間バッチ処理後など）
* **トランザクションログ:** [1時間ごと]（※RPO要件に応じて設定）
* **データ保全:**
    * サーバーローカルディスクへの保存に加え、必ず**外部ストレージ（NAS、ファイルサーバー等）への複製**処理をジョブに組み込むこと。
* **保存期間:** [例: 2週間]（期間超過分は自動削除）

### 5.2. インデックスと統計情報のメンテナンス
検索パフォーマンスの劣化を防ぐため、**週次（週末推奨）**でメンテナンスを実施する。

#### (1) インデックス断片化対応方針
テーブルごとの断片化率（Fragmentation）に応じて処理を自動分岐させる。

| 断片化率 | アクション | 処理内容・留意点 |
| :--- | :--- | :--- |
| **30% 超** | **再構築 (REBUILD)** | ・インデックスを作り直す（オフライン処理/テーブルロック発生）。<br>・統計情報も自動的に更新されるため、追加作業は不要。 |
| **5% ～ 30%** | **再構成 (REORGANIZE)** | ・データの並びを整理する（オンライン処理）。<br>・統計情報は更新されないため、**別途更新が必要（後述）。** |
| **5% 未満** | (原則スキップ) | ・処理不要。 |

#### (2) 統計情報の更新ルール（重要）
日中の自動更新によるパフォーマンス変動（朝は速いが夕方遅い等）を防ぐため、以下の設定を遵守する。

* **サンプリング設定:** **`FULLSCAN`**（全件読込）
* **永続化設定:** **`PERSIST_SAMPLE_PERCENT = ON`**
    * *理由:* 手動でFULLSCANしても、日中の自動更新でサンプリング率が低く上書きされるのを防ぐため。

#### (3) メンテナンスの自動化
後述の「付録：メンテナンス用SQLスクリプト」をSQL Server Agentジョブに登録し、自動実行する。

### 5.3. パッチ適用
* **Windows Update / SQL Server Update:**
    * 月次または四半期ごとに実施検討。
    * シングル構成のため適用時はシステム停止（ダウンタイム）が発生する。関係各所への事前周知を必須とする。

---

## 6. アカウント・セキュリティ管理

### 6.1. 管理者アカウント（AD認証）
* SQL Serverの管理者権限（sysadmin）は、**運用担当者の個人ADアカウント**に付与する。
* 共有アカウント（Administrator等）の常時利用は禁止する。
* 担当者の異動・退職時は速やかに権限を削除する。

### 6.2. アプリケーション接続用アカウント（SQL Server認証）
* アプリケーションが使用するID（例: `AppUser`）のパスワード変更は、アプリ改修スケジュールと同期して行う。
* 特権ID `sa` は、原則として無効化するか、複雑なパスワードを設定して封印し、通常運用では使用しない。

---

## 7. 障害対応フロー

1.  **検知:** ユーザーからの連絡、またはジョブ失敗アラート。
2.  **一次切り分け:**
    * サーバーOSへADログイン可能か？（NW/OS障害の切り分け）
    * SQL Serverサービスは起動しているか？
3.  **復旧措置:**
    * サービスの再起動。
    * エラーログに基づく対応。
    * 直近バックアップからのリストア（※手順書[別紙番号]を参照）。
4.  **報告:** 運用責任者への報告および障害レポートの作成。

---

## 付録：メンテナンス用SQLスクリプト

SQL Server Agentジョブに登録して使用する推奨スクリプト。

```sql
/* 社内システムDB インデックス・統計情報 メンテナンススクリプト 
  (REBUILD > 30% / REORGANIZE 5-30% + Update Stats FULLSCAN/PERSIST)
*/

DECLARE @SchemaName NVARCHAR(128);
DECLARE @TableName  NVARCHAR(128);
DECLARE @IndexName  NVARCHAR(128);
DECLARE @AvgFrag    FLOAT;
DECLARE @SQL        NVARCHAR(MAX);
DECLARE @PageCount  BIGINT;

-- カーソル定義（断片化情報を取得、10ページ以下の小テーブルは除外）
DECLARE MaintenanceCursor CURSOR FOR
SELECT
    s.name AS SchemaName,
    t.name AS TableName,
    i.name AS IndexName,
    stat.avg_fragmentation_in_percent,
    stat.page_count
FROM
    sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') AS stat
    INNER JOIN sys.tables t ON t.object_id = stat.object_id
    INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
    INNER JOIN sys.indexes i ON i.object_id = stat.object_id AND i.index_id = stat.index_id
WHERE
    stat.index_id > 0 
    AND stat.page_count > 10 
ORDER BY
    stat.avg_fragmentation_in_percent DESC;

OPEN MaintenanceCursor;
FETCH NEXT FROM MaintenanceCursor INTO @SchemaName, @TableName, @IndexName, @AvgFrag, @PageCount;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @SQL = '';

    -- ケース1: 断片化率 30% 超 -> [REBUILD]
    IF @AvgFrag > 30
    BEGIN
        PRINT 'Index: [' + @IndexName + '] Frag: ' + CAST(ROUND(@AvgFrag, 2) AS NVARCHAR(20)) + '% -> REBUILD';
        SET @SQL = 'ALTER INDEX [' + @IndexName + '] ON [' + @SchemaName + '].[' + @TableName + '] REBUILD;';
    END

    -- ケース2: 断片化率 5% ～ 30% -> [REORGANIZE] + [UPDATE STATISTICS]
    ELSE IF @AvgFrag >= 5
    BEGIN
        PRINT 'Index: [' + @IndexName + '] Frag: ' + CAST(ROUND(@AvgFrag, 2) AS NVARCHAR(20)) + '% -> REORGANIZE';
        SET @SQL = 'ALTER INDEX [' + @IndexName + '] ON [' + @SchemaName + '].[' + @TableName + '] REORGANIZE;';
        -- 統計情報更新 (FULLSCAN, PERSIST_SAMPLE_PERCENT = ON)
        SET @SQL = @SQL + ' UPDATE STATISTICS [' + @SchemaName + '].[' + @TableName + '] [' + @IndexName + '] WITH FULLSCAN, PERSIST_SAMPLE_PERCENT = ON;';
    END

    -- 実行処理
    IF @SQL <> ''
    BEGIN
        BEGIN TRY
            EXEC sp_executesql @SQL;
            PRINT '>> Success';
        END TRY
        BEGIN CATCH
            PRINT '>> Failed: ' + ERROR_MESSAGE();
        END CATCH
    END

    FETCH NEXT FROM MaintenanceCursor INTO @SchemaName, @TableName, @IndexName, @AvgFrag, @PageCount;
END

CLOSE MaintenanceCursor;
DEALLOCATE MaintenanceCursor;
