
VB.NETのコードをC#に変換する際に参考となる変換例です。<br/>
生成AIを調べながら書いているので、間違いがあればすいません。


## Windows Forms

|  No  | VB.NET                                     | C#                                            | 備考         |
| ----:|:------------------------------------------ |:--------------------------------------------- | ------------ |
|1     | gv.Rows(0).Item("列名").Value = "ああああ" | gv.Rows[0].Cells["列名"].Value = "ああああ";  |              |
|2     | gv.Items("列名",2).Value = "ああああ"      | gv["列名", 2].Value = "ああああ";             |              |

## 汎用

|  No  | VB.NET                               | C#                                            | 置換パターン | 備考         |
| ----:|:------------------------------------ |:--------------------------------------------- |:----------- |:------------ |
|1     | IsDBNull                             | Convert.IsDBNull                              |             |              |
|2     | Mid                                  | using Microsoft.VisualBasic;<br/>String.Mid   |             | Microsoft.VisualBasic 互換アセンブリを利用 |
|3     | Chr                                  | using Microsoft.VisualBasic;<br/>String.Chr   |             | Microsoft.VisualBasic 互換アセンブリを利用 |
|4     | Chr(13) & Chr(10) の改行処理          | Environment.NewLine                           | \bIsDate\(([^)]+)\) → Information.IsDate($1) | Microsoft.VisualBasic 互換アセンブリを利用             |

### Format関数

Microsoft.VisualBasic アセンブリの参照がある環境では、クラス名修飾を付与するだけでVB.NETの動作を完全に維持できる。

| VB.NET                                     | C#                                            | 備考         |
|:------------------------------------------ |:--------------------------------------------- | ------------ |
|                                            |                                               |              |

置換パターン : \bFormat\(([^,]+),\s*([^)]+)\) $\rightarrow$ Strings.Format($1, $2)

## モジュール

VB.NETの モジュール（Module） は、コンパイル時に static class（静的クラス） かつ すべてのメンバーが static なクラスとして扱われます。
また、VB.NETのモジュールはプロジェクト全体（同一名前空間内）から「クラス名を指定せずに直接メンバー（関数・変数）を呼び出せる」という暗黙的なスコープ機能を持っています。
C#へ移行する際の基本原則は以下の2点です。

定義側の変換: Module $\rightarrow$ public static class呼出側の変換:
方法A（標準的・明示的）: ClassName.MethodName() で呼び出す。
方法B（VB.NET同等の記述維持）: using static [Namespace].[ClassName]; ディレクティブを利用し、クラス名修飾を省略して呼び出す。

