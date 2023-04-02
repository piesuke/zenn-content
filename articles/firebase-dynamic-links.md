---
title: "firebase dynamic linksのURLのクエリをWebに遷移しなくても取得できるようにする" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["flutter", "firebase"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

## 問題

共有リンクを firebase dynamic links で実装しているが、アプリ内で firebase dynamic links をタップして違う画面に遷移させたい、という時にそのままだと一回アプリ内ブラウザが開いてしまうという問題があった。

## 解決策

`getDynamicLink`を使う。
https://firebase.google.com/docs/dynamic-links/flutter/receive?hl=ja

```
string getLink(string dynamicLink) {
  final url = Uri.parse(dynamicLink);
  final originalUrl = await FirebaseDynamicLinks.instance.getDynamicLink(url);
}
```

あとは `uri.queryParameters`などで任意のパラメータを取得して、`Navigator.push`で画面遷移すれば OK。

簡単だけど公式ドキュメントの下の方に書いてあり気づかなかったので自戒の為に残しておく。
