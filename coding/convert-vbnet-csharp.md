
Vb.NETのコードをC#に変換する際に参考となる変換例です。<br/>
生成AIを調べながら書いているので、間違いがあればすいません。


## Windows Forms

|  No  | VB.NET                                     | C#                                            | 備考         |
| ----:|:------------------------------------------ |:--------------------------------------------- | ------------ |
|1     | gv.Rows(0).Item("列名").Value = "ああああ" | gv.Rows[0].Cells["列名"].Value = "ああああ";  |              |
|2     | gv.Items("列名",2).Value = "ああああ"      | gv["列名", 2].Value = "ああああ";             |              |
|3     | IsDBNull                                   | Convert.IsDBNull                              |              |

## 汎用

|  No  | VB.NET                                     | C#                                            | 備考         |
| ----:|:------------------------------------------ |:--------------------------------------------- | ------------ |
|1     | IsDBNull                                   | Convert.IsDBNull                              |              |
|2     | Mid                                        | using Microsoft.VisualBasic;<br>String.Mid    | Microsoft.VisualBasic 互換アセンブリを利用 |
