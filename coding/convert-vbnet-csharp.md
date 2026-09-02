
# 本ドキュメントの説明

VB.NETのコードをC#に変換する際に参考となる変換例です。  
生成AIを調べながら書いているので、間違いがあればすいません。

## Windows Forms

|  No  | VB.NET                                     | C#                                            | 備考         |
| ----:|:------------------------------------------ |:--------------------------------------------- | ------------ |
|1     | MsgBox                                    |                                              |              |

## 汎用

|  No  | VB.NET                               | C#                                            | 置換パターン | 備考         |
| ----:|:------------------------------------ |:--------------------------------------------- |:----------- |:------------ |
|1     | IsDBNull                             | Convert.IsDBNull                              |             |              |
|2     | Mid                                  | using Microsoft.VisualBasic;<br/>String.Mid   |             | ※1          |
|3     | Chr                                  | using Microsoft.VisualBasic;<br/>String.Chr   |             | ※1          |
|4     | DateAdd                              |                                               |             |              |
|5     | DateDiff                             |                                               |             |              |
|6     | IsDate                               |                                               |             |              |
|7     | IsNumeric                            |                                               |             |              |
|8     | Now                                  | DateTime.Now()                                |             |              |


※1 Microsoft.VisualBasic 互換アセンブリを利用

## 文字コード

|  No  | VB.NET                               | C#                                            | 置換パターン | 備考         |
| ----:|:------------------------------------ |:--------------------------------------------- |:----------- |:------------ |
|1     | vbCr                                 | "\r"                                          |             |              |
|1     | vbLf                                 | "\n"                                          |             |              |
|3     | vbBack                               | "\b"                                          |             |              |
|4     | vbCrLf                               | \"r\n" もしくは Environment.NewLine            |             |              |


### Format関数

Microsoft.VisualBasic アセンブリの参照がある環境では、クラス名修飾を付与するだけでVB.NETの動作を完全に維持できる。  

| VB.NET                                     | C#                                            | 備考         |
|:------------------------------------------ |:--------------------------------------------- | ------------ |
| Format                                     | Strings.Format                                |              |

置換パターン : \bFormat\(([^,]+),\s*([^)]+)\) $\rightarrow$ Strings.Format($1, $2)

## モジュール

VB.NETの モジュール（Module） は、コンパイル時に static class（静的クラス） かつ すべてのメンバーが static なクラスとして扱われます。
また、VB.NETのモジュールはプロジェクト全体（同一名前空間内）から「クラス名を指定せずに直接メンバー（関数・変数）を呼び出せる」という暗黙的なスコープ機能を持っています。
C#へ移行する際の基本原則は以下の2点です。

定義側の変換: Module $\rightarrow$ public static class呼出側の変換:
方法A（標準的・明示的）: ClassName.MethodName() で呼び出す。
方法B（VB.NET同等の記述維持）: using static [Namespace].[ClassName]; ディレクティブを利用し、クラス名修飾を省略して呼び出す。

## 括弧,Item,インデクサ変換

| No. | 対象                      | VB.NET                                  | C#                                      | 変換ルール / 注意点                           |
| --- | ----------------------- | --------------------------------------- | --------------------------------------- | ------------------------------------- |
| 1   | 引数なしメソッド                | `obj.Close()` / `obj.Close`             | `obj.Close();`                          | C#ではメソッド呼び出しの `()` が必要                |
| 2   | 引数ありメソッド                | `obj.Execute(x)`                        | `obj.Execute(x);`                       | 基本的に `()` のまま                         |
| 3   | 通常プロパティ                 | `obj.Text`                              | `obj.Text`                              | `()` は付けない                            |
| 4   | 配列                      | `arr(i)`                                | `arr[i]`                                | `()` → `[]`                           |
| 5   | List                    | `list(i)`                               | `list[i]`                               | `Item` インデクサとして `[]`                  |
| 6   | Dictionary              | `dict(key)`                             | `dict[key]`                             | キー指定も `[]`                            |
| 7   | `Item` 明示               | `obj.Item(i)`                           | `obj[i]`                                | C#では通常インデクサへ変換                        |
| 8   | DataGridView Rows       | `dgv.Rows(i)`                           | `dgv.Rows[i]`                           | `[]` へ変換                              |
| 9   | DataGridView Columns    | `dgv.Columns(i)`                        | `dgv.Columns[i]`                        | `[]` へ変換                              |
| 10  | DataGridView Cells      | `row.Cells(i)`                          | `row.Cells[i]`                          | `[]` へ変換                              |
| 11  | DataGridView Item       | `dgv.Item(col, row)`                    | `dgv[col, row]`                         | 2引数インデクサ                              |
| 12  | DataGridView Cells.Item | `row.Cells.Item(i)`                     | `row.Cells[i]`                          | `Item` を省略して `[]`                     |
| 13  | SelectedRows            | `dgv.SelectedRows(i)`                   | `dgv.SelectedRows[i]`                   | `[]`                                  |
| 14  | SelectedCells           | `dgv.SelectedCells(i)`                  | `dgv.SelectedCells[i]`                  | `[]`                                  |
| 15  | DataTable Rows          | `table.Rows(i)`                         | `table.Rows[i]`                         | `[]`                                  |
| 16  | DataTable Columns       | `table.Columns(i)`                      | `table.Columns[i]`                      | `[]`                                  |
| 17  | DataRow 列番号             | `row(i)`                                | `row[i]`                                | `[]`                                  |
| 18  | DataRow 列名              | `row("NAME")`                           | `row["NAME"]`                           | 文字列キーも `[]`                           |
| 19  | DataRow Item            | `row.Item("NAME")`                      | `row["NAME"]`                           | `Item` → インデクサ                        |
| 20  | Controls                | `Controls(i)`                           | `Controls[i]`                           | コントロールコレクション                          |
| 21  | ComboBox Items          | `ComboBox1.Items(i)`                    | `comboBox1.Items[i]`                    | `[]`                                  |
| 22  | ListBox Items           | `ListBox1.Items(i)`                     | `listBox1.Items[i]`                     | `[]`                                  |
| 23  | CheckedListBox Items    | `CheckedListBox1.Items(i)`              | `checkedListBox1.Items[i]`              | `[]`                                  |
| 24  | メソッド + インデクサ混在          | `dgv.Rows(i).Cells(j).Value.ToString()` | `dgv.Rows[i].Cells[j].Value.ToString()` | `Rows/Cells` は `[]`、`ToString` は `()` |
| 25  | オブジェクト生成                | `New Hoge()` / `New Hoge`               | `new Hoge()`                            | C#では通常 `()` が必要                       |
