# Wordドキュメントレビュー運用標準
— Excelを用いたコメント往復管理を実現するVBAツール —

この資料は生成AIを利用して作成した資料です。

## 1. 目的

本書は、システム開発における外部仕様書、試験要領書、導入手順書等のWordドキュメントレビューを対象とし、

* レビュー指摘の**視認性向上**
* 指摘対応状況の**一元管理**
* 多巡レビューにおける**運用の属人化防止**

を目的として、部門共通で適用可能なレビュー運用ルールおよび支援ツールの標準を定めるものである。

## 2. 適用範囲

* 本標準は、Microsoft Wordを正式図書としてレビューを行う全ての開発成果物に適用する。
* 電子署名・承認を伴う文書を含む。
* 本運用は **Microsoft Word（Microsoft 365版）のコメントおよび「解決済み」機能**を前提とする。

## 3. 背景と課題

Wordのコメント機能を用いたレビュー運用は、本文と指摘内容の文脈が明確であり、導入コストも低い。一方で、レビューが4〜5巡以上に及ぶ場合、以下の課題が顕在化する。

* コメント件数増加による**視認性の低下**
* 指摘対応状況（完了／未完了）の**把握困難**
* Word以外の管理表を併用した場合の**二重管理・転記ミス**

これらは、Wordコメントが「指摘の発生源」としては優れている一方で、「多件数・多巡レビューにおける一覧管理」を想定していないことに起因する。

## 4. Wordコメント運用の位置づけ

| 観点     | 評価                  |
| ------ | ------------------- |
| 文脈の明確化 | 本文と指摘が直結し、レビュー品質が高い |
| 一覧性    | コメント件数が増えると全体把握が困難  |
| 進捗管理   | 完了／未完了の集計が不十分       |
| 正式図書性  | 正式文書内に履歴を保持可能       |

以上より、**Wordは指摘管理の起点**、**Excelは一覧・進捗管理の補助**として役割分担する。

## 5. 標準レビュー運用ルール

以下の運用ルールは、ツール利用の有無に関わらず部門標準として遵守する。

### 5.1 コメントの解決運用

* 対応済みコメントは削除せず、必ず「解決済み」とする。
* Wordの表示設定で「解決済みのコメントを非表示」を有効化する。

### 5.2 コメント内議論の制限

* コメント欄でのやり取りは**最大3往復まで**とする。
* それ以上の議論は対面・チャット等で実施し、結論のみをコメントに反映する。

### 5.3 軽微修正の扱い

* 誤字脱字、表記揺れ等の軽微な修正はコメントを付与せず、変更履歴（赤入れ）で対応する。

### 5.4 コメント整理の実施基準

以下のいずれかに該当する場合、Excelへの一覧化を実施する。

* 未解決コメントが **100件を超過**
* レビューが **3巡を超過**

※ 本整理は、本書で定義するVBA支援ツールを用いて実施する。

## 6. レビュー支援ツールの概要

### 6.1 ツールの位置づけ

本ツールは、WordとExcel間でコメント情報を往復（ラウンドトリップ）させ、以下を実現する。

* コメントの一覧化・フィルタリング
* 修正方針・回答の集約入力
* Wordコメントスレッドへの回答一括反映

本ツールは **Microsoft Office VBA** により実装されており、外部サービスやクラウド連携を必要としない。

### 6.2 Excel出力レイアウト（標準）

| 列 | 内容              |
| - | --------------- |
| A | コメント番号          |
| B | ページ番号           |
| C | 行番号             |
| D | レビュアー           |
| E | コメント日時          |
| F | 対象テキスト          |
| G | コメント内容          |
| H | ステータス（未完了／解決済み） |
| I | 修正回答（担当者入力欄）    |

※ I列は修正担当者が入力する標準回答欄とする。

### 6.3 VBAソースコード

以下に、本標準運用で使用するVBAマクロを参考実装として示す。

#### 6.3.1 コメント一覧エクスポート（Word → Excel）

```vba
Option Explicit

Sub ExportCommentsToExcel()
    Const wdActiveEndAdjustedPageNumber As Long = 3
    Const wdFirstCharacterLineNumber As Long = 10

    Dim doc As Document
    Dim cmt As Comment
    Dim xlApp As Object
    Dim xlWB As Object
    Dim xlSheet As Object
    Dim i As Long
    Dim headers As Variant
    Dim originalScreenUpdating As Boolean

    On Error GoTo ErrHandler

    Set doc = ActiveDocument

    If doc.Comments.Count = 0 Then
        MsgBox "この文書にはコメントがありません。", vbInformation
        Exit Sub
    End If

    Set xlApp = CreateObject("Excel.Application")
    If xlApp Is Nothing Then
        MsgBox "Excelを起動できませんでした。", vbCritical
        Exit Sub
    End If

    originalScreenUpdating = Application.ScreenUpdating
    Application.ScreenUpdating = False

    xlApp.Visible = True
    Set xlWB = xlApp.Workbooks.Add
    Set xlSheet = xlWB.Sheets(1)

    headers = Array("No", "ページ", "行", "レビュアー", "日付", "対象テキスト", "コメント内容", "ステータス", "修正回答")
    xlSheet.Range(xlSheet.Cells(1, 1), xlSheet.Cells(1, UBound(headers) + 1)).Value = headers

    xlApp.ActiveWindow.SplitRow = 1
    xlApp.ActiveWindow.FreezePanes = True

    i = 2
    For Each cmt In doc.Comments
        xlSheet.Cells(i, 1).Value = i - 1
        xlSheet.Cells(i, 2).Value = cmt.Reference.Information(wdActiveEndAdjustedPageNumber)
        xlSheet.Cells(i, 3).Value = cmt.Reference.Information(wdFirstCharacterLineNumber)
        xlSheet.Cells(i, 4).Value = cmt.Author
        xlSheet.Cells(i, 5).Value = Format(cmt.Date, "yyyy/mm/dd HH:mm")
        xlSheet.Cells(i, 6).Value = CleanText(cmt.Scope.Text)
        xlSheet.Cells(i, 7).Value = CleanText(cmt.Range.Text)
        xlSheet.Cells(i, 8).Value = IIf(cmt.Done, "解決済み", "未完了")
        i = i + 1
    Next cmt

    xlSheet.Columns("A:I").AutoFit

Cleanup:
    Application.ScreenUpdating = originalScreenUpdating
    Exit Sub

ErrHandler:
    MsgBox Err.Description, vbCritical
    Resume Cleanup
End Sub

Private Function CleanText(ByVal s As String) As String
    s = Replace(s, vbCrLf, " ")
    s = Replace(s, vbTab, " ")
    Do While InStr(s, "  ") > 0
        s = Replace(s, "  ", " ")
    Loop
    CleanText = Trim(s)
End Function
```

#### 6.3.2 修正回答インポート（Excel → Word）

```vba
Sub ImportAnswersFromExcel()
    Dim doc As Document: Set doc = ActiveDocument
    Dim xlApp As Object, xlWB As Object, xlSheet As Object
    Dim excelFile As Variant
    Dim i As Long, lastRow As Long
    Dim cmtIndex As Long, answerText As String

    excelFile = Application.GetOpenFilename("Excel Files (*.xlsx; *.xlsm), *.xlsx; *.xlsm")
    If excelFile = False Then Exit Sub

    Set xlApp = CreateObject("Excel.Application")
    Set xlWB = xlApp.Workbooks.Open(excelFile)
    Set xlSheet = xlWB.Sheets(1)

    lastRow = xlSheet.Cells(xlSheet.Rows.Count, 1).End(-4162).Row

    For i = 2 To lastRow
        cmtIndex = xlSheet.Cells(i, 1).Value
        answerText = xlSheet.Cells(i, 9).Value
        If answerText <> "" Then
            doc.Comments(cmtIndex).Replies.Add _
                Range:=doc.Comments(cmtIndex).Range, _
                Text:="【回答】" & answerText
        End If
    Next i

    xlWB.Close SaveChanges:=False
    xlApp.Quit
    MsgBox "インポートが完了しました。"
End Sub
```

## 7. 標準ワークフロー

1. **エクスポート**
   Word上でツールを実行し、全コメントをExcelに一覧化する。

2. **回答記入**
   修正担当者はExcelの「修正回答」欄（I列）に対応内容を記載する。

3. **インポート**
   Excelに記載された回答を、Wordコメントの返信として一括反映する。

4. **確認・解決**
   レビュアーが回答内容を確認し、問題なければコメントを「解決済み」とする。

5. **完了**
   全コメントが解決済みとなった時点でレビュー完了とする。

## 8. 期待される効果

* レビュー指摘の可視性・追跡性の向上
* 多巡レビューにおける管理負荷の低減
* レビュー運用の標準化・属人化防止
* 正式図書（Word）と管理表（Excel）の役割分離による統制強化

---

以上の運用を部門標準として適用することで、レビュー品質を維持したまま、効率的かつ統制の取れたレビュー運用を実現する。
