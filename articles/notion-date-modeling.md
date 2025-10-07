---
title: "Notionのデータ構造がかなり美しいのでまとめる" # 記事のタイトル
emoji: "🤖" # アイキャッチとして使われる絵文字（1文字だけ）
type: "idea" # tech: 技術記事 / idea: アイデア記事
topics: ["notion", "idea"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。
私たちは、マーダーミステリーを遊べるアプリ「ウズ」と、マーダーミステリーを制作してウズ上で配信できるアプリ「ウズスタジオ」を開発しています。
最近良かったマーダーミステリーは「[Dual~死遊病／名探偵宍流累 最後の事件~](https://mdms.jp/scenarios/8182)」です。

弊社が運営しているマダミス制作サービス「ウズスタジオ」は、柔軟性を持たせるために NoSQL を採用していますが、2 年運営する中で少しずつ限界も見え始めており、NoSQL の難しさを実感しています。

そこで、他のサービスがどのようなデータ設計で柔軟性を実現しているのかを調査したところ、Notion のデータモデルが非常に美しかったため、本記事でまとめます。参考文献は[公式テックブログ](https://www.notion.com/blog/data-model-behind-notion)と[Notion API ドキュメント](https://developers.notion.com/reference/intro)です。

## Notion の核となるブロック

Notion に表示される全てのデータは「ブロック」と呼ばれる単位にまとめられています。テキスト、画像、ページなど、あらゆる要素がブロックとして管理されています。
![](/images/CleanShot2025-10-07.png)

### type

Notion 上でどのように表示するかを決定する値です。

### properties

ブロックごとの属性を設定します（タイトル、リスト、チェックボックスなど）。

### content

ブロック内で使われる子ブロックの id のリストを保持します（toggle 内で使われているブロックなど）。

### parent

親ブロックの id。主にアクセス許可の制御に使用されます。

### content と parent の関係性

![](/images/CleanShot2025-10-07_11.58.50.png)

ブロックはこのようなツリー構造を形成しています。content はページ上の表示順を決定し、各ブロックは一つだけ parent を持ちます。

この設計の秀逸な点は、type と properties の責任分離です。type は「どのように表示するか」のみに責任を持ち、properties は「実際のデータ」を保持することのみに責任を持ちます。この分離により、type の切り替えが容易になり、高い柔軟性が実現されています。

Notion のビューアーは、type に応じて必要な properties のみを取得して表示します。そのため、todo list でチェックをつけた要素を見出しに変更し、再び todo list に戻しても、チェック情報は保持され続けます。

## データベースの構造

次に、Notion のデータベースがどのような構造になっているのかを、Notion API のドキュメントから推察してみます。

2025 年 9 月から Data Source が導入されました。
![](/images/CleanShot2025-10-07_11.37.46.png)

従来はデータベース自体が properties を持っていましたが、これでは一つのデータソースしか扱えないという欠点がありました。中間レイヤーとして Data Source を導入することで、複数のデータソースを扱えるようになったようです。

では、Data Source の詳細を見てみましょう。
[https://developers.notion.com/reference/data-source]

Data Source の properties には、データベースの列を定義する type が格納されます。
![](/images/CleanShot2025-10-07_11.45.43.png)

Data Source の各行は page object となっており、page の properties には定義された type に準拠した値が入ります。
[https://developers.notion.com/reference/page#property-value-object]

```json
{
  "object": "page",
  "id": "be633bf1-dfa0-436d-b259-571129a590e5",
  "created_time": "2022-10-24T22:54:00.000Z",
  "last_edited_time": "2023-03-08T18:25:00.000Z",
  "created_by": {
    "object": "user",
    "id": "c2f20311-9e54-4d11-8c79-7398424ae41e"
  },
  "last_edited_by": {
    "object": "user",
    "id": "9188c6a5-7381-452f-b3dc-d4865aa89bdf"
  },
  "cover": null,
  "icon": {
    "type": "emoji",
    "emoji": "🐞"
  },
  "parent": {
    "type": "data_source_id",
    "data_source_id": "a1d8501e-1ac1-43e9-a6bd-ea9fe6c8822b"
  },
  "archived": true,
  "in_trash": true,
  // この部分がdata sourceのそれぞれのtypeに準拠している
  "properties": {
    "Due date": {
      "id": "M%3BBw",
      "type": "date",
      "date": {
        "start": "2023-02-23",
        "end": null,
        "time_zone": null
      }
    },
    "Status": {
      "id": "Z%3ClH",
      "type": "status",
      "status": {
        "id": "86ddb6ec-0627-47f8-800d-b65afd28be13",
        "name": "Not started",
        "color": "default"
      }
    },
    "Title": {
      "id": "title",
      "type": "title",
      "title": [
        {
          "type": "text",
          "text": {
            "content": "Bug bash",
            "link": null
          },
          "annotations": {
            "bold": false,
            "italic": false,
            "strikethrough": false,
            "underline": false,
            "code": false,
            "color": "default"
          },
          "plain_text": "Bug bash",
          "href": null
        }
      ]
    }
  },
  "url": "https://www.notion.so/Bug-bash-be633bf1dfa0436db259571129a590e5",
  "public_url": "https://jm-testing.notion.site/p1-6df2c07bfc6b4c46815ad205d132e22d"
}
```

## まとめ

Notion は、様々な種類のコンテンツを type と properties という 2 つの概念で表現しており、優れた抽象化を実現しています。具体的なコンテンツに引っ張られることなく、柔軟性と開発しやすさを両立するための抽象化設計の重要性を学ぶことができました。
