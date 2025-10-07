---
title: "Notionのデータ構造がかなり美しいのでまとめる" # 記事のタイトル
emoji: "🤖" # アイキャッチとして使われる絵文字（1文字だけ）
type: "idea" # tech: 技術記事 / idea: アイデア記事
topics: ["notion", "idea"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。
私たちは、マーダーミステリーを遊べることが出来るアプリ「ウズ」と、マーダーミステリーを制作してウズ上で遊べることが出来るアプリ「ウズスタジオ」を開発しています。
最近良かったマーダーミステリーは「[Dual~死遊病／名探偵宍流累 最後の事件~](https://mdms.jp/scenarios/8182)」です。

弊社が運営しているマダミス制作サービス「ウズスタジオ」は柔軟性を持たせるために NoSQL を採用しているのですが、2 年運営していく中で少しずつ限界も見え始めており、やはり NoSQL は難しいと感じることも増えています。

そんな中で、他のサービスはどのようなデータ設計で柔軟性を持たせているのかと情報を漁っていると、Notion のデータ構造がかなり美しかったのでメモがてらまとめようと思います。参考文献は[こちらの公式テックブログ](https://www.notion.com/blog/data-model-behind-notion)と[Notion API ドキュメント](https://developers.notion.com/reference/intro)です。

## Notion の核となるブロック

Notion に表示される全てのデータはブロックと呼ばれるものにまとめられています。(テキストや画像、ページなども)
![](/images/CleanShot 2025-10-07.png)

### type

この値で Notion 上でどのように表示するかを決定しています。

### properties

ブロックごとの属性を設定しています。(タイトルやリスト、チェックボックスなど)

### content

content はブロック内で使われるブロックの id のリストを所有しています。(toggle の中のテキストなど)

### parent

ブロックの親の id。アクセス許可のみに使用されているらしい。

### content と parent の関係性

![](/images/CleanShot 2025-10-071_1.26.13.png)

このようにツリー構造となっており、content はページ上の表示順を決めるものとして機能しており、content はひとつだけ parent を持ちます。

type と properties が秀逸で、type がどのようなビューで表示されるかにのみ責任を持ち、properties が実際の中身の情報を持つことのみに責任を持っているので、type の切り替えが容易で、柔軟性もかなり高い構造になっています。Notion のビュワーは、type によって必要な properties だけを取得して表示しています。ですので、todo list でチェックをつけた要素を見出しに変え、また todo list に戻した時にもチェック情報を保持し続けることができます。

## データベースはどのような構造なのか

次に、Notion でよく使うデータベースはどのような構造になっているのかを、Notion API のドキュメントを見ながら推察してみます。

2025 年 9 月から、Data Source が導入されたようです。
![](/images/CleanShot 2025-10-07_11.37.46.png)

ここを見ると、元々データベース自体が properties を持っていたようですが、これだと一つのデータしか扱えないという欠点があったようで、中間レイヤーとして Data Source を加えることで複数のデータを扱えることが可能になったようです。

では、Data Source の中を見てみましょう。
[https://developers.notion.com/reference/data-source]

Data Source の properties には、データベースの列を定義する type が入ることが想定されています。
![](/images/CleanShot 2025-10-07_11.45.43.png)

Data Source の各行は page object となっており、page の properties の中に定義した type に準拠した値が入る想定のようです。
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

Notion のような様々な種類のコンテンツを表示するにあたって、基本的には type と properties という 2 種類で表現しているのはかなり上手く抽象化を行なっていると思いました。具体的なコンテンツに引っ張られず、どうやって抽象化すると柔軟性と開発しやすさを両立できるのかを考える必要があると思いました。
