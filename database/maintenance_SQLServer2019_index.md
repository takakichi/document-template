# SQL Server 2019 サーバ移行時のインデックス調査・保守手順書

## 1. 目的・適用範囲

サーバ移行時に伴い、SQL Server 2019の**移行後の性能低下を防止し、主要業務の応答性能を維持する**。インデックスの再構築は手段であり、無条件の一括実行はしない。

- 対象：ユーザーデータベースの行ストアインデックスと関連する統計情報。列ストア、メモリ最適化テーブル、特殊なインデックスは別途判断する。
- 原則：移行元は参照・測定のみ。移行先の検証環境で試行し、本番は検証済みの処理に限定する。
- 優先順位：①業務性能の維持、②整合性・復元可能性、③必要なインデックス保守、④ディスク容量削減。
- **Shrinkは標準手順に含めない**。未使用領域の回収が別途必要と判定された場合のみ、性能への影響を検証したうえで別作業とする。

## 2. 作業計画・判定基準

| 工程 | 実施場所 | 実施内容 | 成果物 |
|---|---|---|---|
| 1. 移行前調査 | 移行元 | 構成、インデックス状態、統計情報、代表SQLの性能 | 移行前ベースライン |
| 2. 復元試験 | 検証環境 | バックアップ復元、整合性・接続確認 | 復元記録 |
| 3. 性能比較 | 検証環境 | 同条件で代表SQLを測定し移行前と比較 | 比較表 |
| 4. 原因分析 | 検証環境 | 遅いSQLの実行計画・待機・I/O・統計を調査 | 保守対象一覧 |
| 5. インデックス保守 | 検証環境 | 選定した対象のみREBUILD/REORGANIZE/統計更新 | 保守実績 |
| 6. 再測定・承認 | 検証環境 | 性能、断片化、ログ・ディスク使用量、整合性 | 本番実施可否 |
| 7. 本番移行 | 移行先 | 検証済みの手順を適用、業務試験 | 作業記録 |
| 8. 移行後監視 | 移行先 | 1週間・1か月の性能と容量を確認 | 監視結果 |

性能の合否は、代表業務ごとに事前に合意した閾値で判断する。例：同一データ・同一パラメータでの代表SQLの実行時間が移行前の120%以内。**120%は例示値であり正式基準ではない**。測定は複数回実施し、中央値、論理読み取り、CPU時間を併記する。

## 3. 移行前調査（移行元）

### 3.1 構成と設定を記録

SSMSでSQL Server 2019に接続し、以下を実行する。

```sql
SELECT @@VERSION AS SqlServerVersion;
SELECT name, compatibility_level, recovery_model_desc, state_desc,
       is_auto_create_stats_on, is_auto_update_stats_on
FROM sys.databases
WHERE database_id > 4
ORDER BY name;
```

加えて、移行元と移行先のCPU・メモリ・ストレージ、SQL Serverのビルド/CU、MAXDOP、cost threshold for parallelism、max server memory、tempdb構成を記録する。これらの差異はインデックス以外の性能低下要因になり得る。

### 3.2 インデックスの状態を記録

**対象データベースに切り替えて実行する。** 大規模DBではI/O負荷が生じるため、低負荷時間帯に実行する。`SAMPLED` は概算であり、小規模なインデックスでは内部的に詳細スキャンになる場合がある。

```sql
USE [TargetDB];
GO
SELECT
    OBJECT_SCHEMA_NAME(ps.object_id) AS SchemaName,
    OBJECT_NAME(ps.object_id) AS TableName,
    i.name AS IndexName,
    ps.index_id,
    ps.partition_number,
    ps.page_count,
    CAST(ps.avg_fragmentation_in_percent AS decimal(6,2)) AS FragmentationPct,
    CAST(ps.avg_page_space_used_in_percent AS decimal(6,2)) AS PageDensityPct,
    i.fill_factor
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') AS ps
JOIN sys.indexes AS i
  ON i.object_id = ps.object_id AND i.index_id = ps.index_id
JOIN sys.tables AS t ON t.object_id = ps.object_id
WHERE ps.index_id > 0
  AND ps.alloc_unit_type_desc = 'IN_ROW_DATA'
  AND i.type IN (1,2)
  AND ps.page_count >= 1000
  AND t.is_ms_shipped = 0
ORDER BY ps.page_count DESC, ps.avg_fragmentation_in_percent DESC;
```

結果をCSV保存する。パーティションごとに行が出るため、対象選定ではパーティション番号も確認する。断片化率だけでなく、ページ数・ページ密度・実際のアクセス特性を見る。

### 3.3 代表業務の性能を記録

- 業務で頻繁に使う画面、集計、バッチから代表処理を選定する。
- 同じデータ、同じSQLパラメータ、同等の同時実行負荷で測定する。
- 実行時間（複数回の中央値）、CPU時間、論理読み取り、実行計画を記録する。
- SQL Serverのキャッシュ差が結果に影響するため、初回と2回目以降を区別する。本番でキャッシュを強制クリアしない。

SSMSで単独の代表SQLを測る際の例：

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
-- ここに代表SQLを記載（実行計画もSSMSで保存）
-- SELECT ...;
SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
```

## 4. 移行先検証環境での比較

1. 移行元のバックアップを検証環境へ復元する。
2. DB状態、互換性レベル、統計自動更新設定、アプリケーション接続を確認する。
3. **インデックス保守前に**代表業務を同条件で測定し、移行元と比較する。
4. 遅いSQLがあれば実行計画、論理読み取り、CPU時間、I/O待機、統計情報の鮮度を確認する。
5. 移行後の性能低下がインデックス以外（設定差、ストレージ、実行計画の変化等）に起因する場合は、その原因を優先して対処する。

## 5. インデックス保守対象の選定

| 状態 | 原則的な対応 |
|---|---|
| 性能問題がなく、小規模 | 保守しない |
| 断片化が大きく、範囲走査で性能影響が確認できる | REBUILDまたはREORGANIZEを検討 |
| ページ密度が低い | FILLFACTOR・更新頻度を確認し、必要ならREBUILD |
| 統計情報の鮮度が原因と考えられる | 対象統計情報を更新 |
| インデックス保守では改善しない | SQL・実行計画・設定・I/Oを再調査 |

断片化率30%以上・ページ数1,000以上は**調査候補を絞るための仮の目安**であり、無条件のREBUILD基準ではない。書き込みの多いテーブルはページ密度を高めすぎるとページ分割が増える場合がある。

## 6. インデックス保守の実行（検証環境）

実行前にバックアップ/再復元手段、十分なデータ・ログ・tempdb容量、作業時間、ロック影響を確認する。SQL Server 2019 Standard Editionでは通常の行ストアインデックスのオンラインREBUILDが使えないため、業務停止時間内で実行する。

### 6.1 REBUILDの例

```sql
USE [TargetDB];
GO
ALTER INDEX [IX_TargetTable_Column]
ON [dbo].[TargetTable]
REBUILD;
GO
```

### 6.2 REORGANIZEの例

```sql
USE [TargetDB];
GO
ALTER INDEX [IX_TargetTable_Column]
ON [dbo].[TargetTable]
REORGANIZE;
GO
```

REBUILDはインデックス統計情報も更新するが、REORGANIZEは統計情報を更新しない。必要な場合は対象統計を個別更新する。

```sql
UPDATE STATISTICS [dbo].[TargetTable] [IX_TargetTable_Column];
```

`ALTER INDEX ALL ... REBUILD` や全DB一括REBUILDは標準手順としない。処理中はトランザクションログの増加とディスク空き容量を監視する。

### 6.3 保守後の確認

1. 3.2の調査SQLを再実行し、対象インデックスの断片化率・ページ密度を比較する。
2. 3.3と同じ代表SQLを複数回測定する。
3. 実行計画、論理読み取り、CPU時間、処理時間を比較する。
4. MDF/NDF/LDFのサイズ、空き容量、再拡張の有無を記録する。
5. 以下の整合性検査を検証環境で実施する。時間・tempdb容量を確保する。

```sql
DBCC CHECKDB (N'TargetDB') WITH NO_INFOMSGS;
```

性能が改善しない場合は、インデックス再構築を他のDBに横展開せず、原因を再調査する。

## 7. 本番移行時のチェックリスト

- [ ] 移行元の構成・代表SQL性能・インデックス状態を記録済み
- [ ] 検証環境でバックアップ復元と性能比較を実施済み
- [ ] インデックス保守対象・実行SQL・処理時間を確定済み
- [ ] 必要なディスク・ログ・tempdb容量を確認済み
- [ ] 最終バックアップと切戻し方針を確認済み
- [ ] アプリケーションを停止しDB更新を停止
- [ ] 移行先へ最終バックアップを復元
- [ ] DB接続・構成設定・整合性を確認
- [ ] 検証済みのインデックス保守を必要な対象にのみ実施
- [ ] 主要画面・バッチの動作と応答時間を確認
- [ ] DBファイル容量・ログ容量を記録
- [ ] 合否判定後に本番切替

**切戻し**：本番切替前の問題は移行元の運用継続を基本とする。切替後に更新が発生した場合のデータ同期・切戻し方法は別途決める。

## 8. 移行後監視

| 時期 | 監視内容 |
|---|---|
| 移行直後 | 主要業務の処理時間、エラー、SQL接続、CPU/I/O、ファイル容量 |
| 1週間後 | 遅延SQL、実行計画変化、統計情報、自動拡張、ログ使用量 |
| 1か月後 | 性能傾向、インデックス保守の追加要否、容量推移 |

## 9. 記録様式

| DB | 代表業務/SQL | 移行前中央値(ms) | 移行先保守前(ms) | 保守後(ms) | 論理読み取りの変化 | 対応内容 | 判定 |
|---|---|---:|---:|---:|---|---|---|
| （記入） | （記入） | | | | | | |

| DB | テーブル | インデックス | ページ数 | 断片化率(前→後) | ページ密度(前→後) | 実施処理 | 処理時間 | 備考 |
|---|---|---|---:|---|---|---|---|---|
| （記入） | | | | | | | | |

## 10. 実施上の結論

今回の優先目的は**サーバ移行による性能低下を防止すること**である。したがって、移行前後の代表業務の性能測定を先に実施し、問題があるインデックスに限って保守する。**Shrinkは実施しないことを標準**とし、ディスク使用量削減は性能維持が確認された後に別途判断する。
