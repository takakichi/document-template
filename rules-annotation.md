# ソースコード・コメントアノテーション運用ルール(案)

本ルールは、生成AIで作成したルールです。

## 1. 目的
本ルールは、ソースコード内に記述する特殊なコメント（アノテーション）の記述方法を統一することを目的とします。
統一されたフォーマットで運用することで、以下の効果を期待します。

* **可読性の向上:** コードの意図や注意点が即座に伝わるようにする。
* **タスクの可視化:** 未実装機能や修正が必要な箇所を検索可能にし、技術的負債の放置を防ぐ。
* **責任の明確化:** 「いつ」「誰が」その判断をしたかを記録し、問い合わせ先を明確にする。

## 2. アノテーションとは

本ルールにおける「アノテーション」とは、プログラミング言語の機能（例: Javaの`@Override`など）ではなく、ソースコードのコメント内に特定のキーワード（タグ）を埋め込む手法を指します。
IDE（統合開発環境）のタスク管理機能や、grep検索などで抽出することを前提として記述します。

## 3. 使用するアノテーション

### 3.1 記述フォーマット

アノテーションコメントは、以下のフォーマットで記述してください。
**末尾に必ず「日付」と「記述者名」を含める**ローカルルールを適用します。

> `// [TAG]: コメント内容 (YYYY/MM/DD Name)`

* **TAG:** すべて大文字で記述する。
* **コメント内容:** 簡潔に、何をする必要があるか、またはなぜそうしたかを記述する。
* **日付:** 西暦で `YYYY/MM/DD` 形式とする。
* **Name:** チーム内で識別可能な名前（苗字、ニックネーム、またはアカウント名）。

### 3.2 定義済みタグ一覧

使用可能なタグは以下の5種類とします。個人の判断で独自のタグを増やさないでください。

| タグ | 意味 | 使用タイミング |
| --- | --- | --- |
| **TODO** | 実装予定・残タスク | 後で実装する必要がある機能、リファクタリングの予定など。 |
| **FIXME** | 修正が必要 | バグの可能性がある箇所、動作はするが修正が必須な既知の不具合。 |
| **HACK** | 暫定対応・回避策 | 根本解決ではないが、一時的な回避策（ワークアラウンド）として実装した箇所。「なぜそうしたか」の理由記述が必須。 |
| **NOTE** | 注意事項・備忘録 | 複雑なロジックの補足説明、背景知識、特定の仕様に関するメモ。 |
| **REVIEW** | 要レビュー | 実装に自信がない箇所、有識者に確認してほしい箇所。プルリクエスト前に解消することが望ましい。 |

## 4. 記述例

### TODO（あとでやる）

```javascript
// TODO: バリデーション処理を共通関数に切り出す (2026/01/22 Yamadataro)
if (input.length > 100) {
    return error;
}

```

### FIXME（直すべき不具合）

```python
def calculate_tax(price):
    # FIXME: 消費税率がハードコーディングされているため、設定ファイルから読み込むように修正する (2026/02/10 Suzuki)
    return price * 1.10

```

### HACK（苦肉の策・暫定対応）

```java
// HACK: ライブラリのバグ回避のため、一時的にスリープを入れている。v2.1への更新時に削除すること (2026/03/15 Sato)
Thread.sleep(100);
process();

```

### NOTE（なぜこう書いたか）

```csharp
// NOTE: パフォーマンス要件により、ここではLINQではなくforループを使用している (2026/01/20 Tanaka)
for (int i = 0; i < list.Count; i++) {
    // 処理...
}

```

### REVIEW（見てほしい）

```typescript
// REVIEW: この正規表現で全てのメールアドレス形式をカバーできているか不安。確認求む (2026/01/22 Takahashi)
const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

```

---

## 5. VS Code 設定（推奨）

チームで統一した開発環境を構築するため、以下の設定を推奨します。

### 5.1 スニペット設定（入力支援）

`todo` + `Tab` キーで、日付・名前入りのフォーマットを呼び出せるようにします。
各自のユーザースニペット、またはプロジェクトの `.vscode/*.code-snippets` に以下を設定してください。

**注意:** `YOUR_NAME` の部分は各自の名前に書き換えてください。

```json
{
	"Annotation TODO": {
		"prefix": "todo",
		"body": ["TODO: $1 ($CURRENT_YEAR/$CURRENT_MONTH/$CURRENT_DATE YOUR_NAME)"],
		"description": "TODOコメント"
	},
	"Annotation FIXME": {
		"prefix": "fixme",
		"body": ["FIXME: $1 ($CURRENT_YEAR/$CURRENT_MONTH/$CURRENT_DATE YOUR_NAME)"],
		"description": "FIXMEコメント"
	},
	"Annotation HACK": {
		"prefix": "hack",
		"body": ["HACK: $1 ($CURRENT_YEAR/$CURRENT_MONTH/$CURRENT_DATE YOUR_NAME)"],
		"description": "HACKコメント"
	},
	"Annotation NOTE": {
		"prefix": "note",
		"body": ["NOTE: $1 ($CURRENT_YEAR/$CURRENT_MONTH/$CURRENT_DATE YOUR_NAME)"],
		"description": "NOTEコメント"
	},
	"Annotation REVIEW": {
		"prefix": "review",
		"body": ["REVIEW: $1 ($CURRENT_YEAR/$CURRENT_MONTH/$CURRENT_DATE YOUR_NAME)"],
		"description": "REVIEWコメント"
	}
}

```

### 5.2 Todo Tree 設定（可視化）

拡張機能「[Todo Tree](https://marketplace.visualstudio.com/items?itemName=Gruntfuggly.todo-tree)」を使用し、タグを色分け表示します。
プロジェクト直下の `.vscode/settings.json` に以下を追加することで、チーム全員に設定が共有されます。

```json
{
    "todo-tree.general.tags": [
        "TODO", "FIXME", "HACK", "NOTE", "REVIEW"
    ],
    "todo-tree.highlights.customHighlight": {
        "TODO":   { "icon": "check", "foreground": "#33CC33", "iconColour": "#33CC33" },
        "FIXME":  { "icon": "flame", "foreground": "#FF3333", "iconColour": "#FF3333", "fontWeight": "bold" },
        "HACK":   { "icon": "tools", "foreground": "#FFCC00", "iconColour": "#FFCC00" },
        "NOTE":   { "icon": "note",  "foreground": "#3399FF", "iconColour": "#3399FF" },
        "REVIEW": { "icon": "eye",   "foreground": "#CC33FF", "iconColour": "#CC33FF", "gutterIcon": true }
    },
    "todo-tree.tree.showScanModeButton": false,
    "todo-tree.tree.grouped": true,
    "todo-tree.tree.expanded": true
}

```
