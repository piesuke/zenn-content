---
title: "WYSIWYG エディタ「Quill」の紹介と、ペースト時の書式設定をカスタマイズする方法" # 記事のタイトル
emoji: "🎨" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["react", "nextjs", "typescript", "quill", "editor", "tech"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。
私たちは、マーダーミステリーを楽しむためのアプリ「ウズ」と、マーダーミステリーを制作してウズ上で遊べることが出来るアプリ「ウズスタジオ」、マーダーミステリーを検索できるサイト「マダミス.jp」を開発・運営しています。
私が最近やって良かったマーダーミステリーは「[死神はトリックをかたらない](https://mdms.jp/scenarios/2678)」です。

今回は、弊社で採用しているテキストエディタ「Quill」の紹介と、ペースト時の書式設定カスタマイズ方法を解説します。

## 背景

弊社ではマーダーミステリーのアプリを運営していますが、一般的にゲームのようなグラフィカルな画面ではなく、文章が主役のアプリであり、装飾の重要性が高いです。また、作家の方々が文章を書く際に、装飾を簡単に行えるようにすることが重要でした。

元々文章の装飾・表示には マークダウン を使用していましたが、以下の点で マークダウン では不十分であると感じていました。

- 装飾が限られる点
- 作家の方に マークダウンの記法を覚えてもらう必要がある点
- 文字色等の装飾を拡張する際に Flutter のマークダウンライブラリをフォークする必要があり、自前でメンテナンスし続けるのが辛かった点

そこで、WYSIWYG エディタである Quill を採用することにしました。

## Quill の選定理由

WYSIWYG エディタを導入するにあたっての要件は以下がありました。

- 作家の方が文章の編集や装飾を簡単に行えること
- 装飾が豊富で、拡張性が高いこと
- アプリネイティブと Web どちらでも動作すること
  - ただし編集は Web で行い、アプリでは表示のみ行う

このうち三つ目の「アプリネイティブと Web どちらでも動作すること」が重要で、アプリでは Flutter 、Web では Next.js を使用しているため、両方でライブラリが使えることが必要でした。Web のみで使えるライブラリは多いですが、選定を行なった去年時点では Flutter、Next.js 両方で安定して使えるライブラリが存在するのは Quill がほとんど唯一と言っていい状況だったので、Quill を採用することにしました。

一つ目と二つ目に関しては、動作面では違和感なく使えたこと、カスタマイズ性が高いようなデータ構造を持っていたことからも申し分ないと感じました。

## Quill とは

Quill は、リッチテキストエディタを実装するためのライブラリです。
https://quilljs.com/

スター数は 2024/07/10 時点で 42K を超えており、非常に人気のあるライブラリです。

データ構造は Delta という形式で表現され、以下のような形式です。

```json
{
  "ops": [
    { "insert": "Gandalf", "attributes": { "bold": true } },
    { "insert": " the " },
    { "insert": "Grey", "attributes": { "color": "#cccccc" } }
  ]
}
```

実際の文字列は insert で表現され、装飾は attributes で表現されます。

## Quill の導入

弊社では Web フロントエンドとアプリケーション側で Quill を使用していますが、今回は Web フロントエンドに絞って解説していきます。

Next.js、React で Quill を導入する場合、ライブラリを使わずとも実装できますが、react-quill というライブラリがよく使われています。

https://github.com/zenoamaro/react-quill

```bash
npm install react-quill
```

```tsx
import React, { useState } from "react";
import ReactQuill from "react-quill";
import "react-quill/dist/quill.snow.css";

function MyComponent() {
  const [value, setValue] = useState("");

  return <ReactQuill theme="snow" value={value} onChange={setValue} />;
}
```

これだけで最低限のエディターが表示されます。

## WYSIWYG エディタに付きまとうペースト処理の問題

WYSIWYG エディタを使う際に問題となることが多いのがペースト処理です。特に、他のエディターからコピーした文章をペーストする際に、書式設定が崩れてしまうことがあります。
作家の方々は各々文章を書く際に、Word や Google Docs などのエディターを使っていることもあり、その文章をそのままペーストすることが多いです。各エディターの書式設定が異なるため、ペースト時に崩れやすいです。また、自分たちのサービスでは UX の観点でいくつかの書式設定を禁止しています。例えば以下のような書式設定を禁止しています。

- 文字の背景色
- 外部リンク

また、アプリ側でカラーテーマを設定している為、文字色はカラーコードを持たずに、「red」や「blue」などの文字列で指定しておいて、アプリ側でカラーコードに変換するような処理を行っています。そのため、ペースト時に文字色がカラーコードで指定されている場合、そのカラーコードに近い文字列に変換しなければ文字色が失われてしまいます。

このような問題に対処するため、Quill デフォルトのペースト処理をカスタマイズすることが必要です。

## ペースト処理をカスタマイズする

Quill のペースト処理は、`clipboard` モジュールを使って行われています。このモジュールのソースコードは以下です。

https://github.com/slab/quill/blob/main/packages/quill/src/modules/clipboard.ts

このモジュールの

```ts
onPaste(range: Range, { text, html }: { text?: string; html?: string }) {
    const formats = this.quill.getFormat(range.index);
    const pastedDelta = this.convert({ text, html }, formats);
    debug.log('onPaste', pastedDelta, { text, html });
    const delta = new Delta()
      .retain(range.index)
      .delete(range.length)
      .concat(pastedDelta);
    this.quill.updateContents(delta, Quill.sources.USER);
    // range.length contributes to delta.length()
    this.quill.setSelection(
      delta.length() - range.length,
      Quill.sources.SILENT,
    );
    this.quill.scrollSelectionIntoView();
  }
```

の部分がペースト時に発火しています。元々は、`text` と `html` をそのまま Delta に変換しているだけですが、これをカスタマイズすることでペースト時の処理を変更することができます。

こちらがコード全文です。

```tsx
const Clipboard = Quill.import("modules/clipboard");

class PlainClipboard extends Clipboard {
  onPaste(range: Range, { text, html }: { text?: string; html?: string }) {
    const quillObj: Quill = this.quill as Quill;

    let delta: DeltaStatic = new Delta()
      .retain(range.index)
      .delete(range.length);
    const parser = new DOMParser();
    const doc = parser.parseFromString(html, "text/html");
    const elements = doc.body.querySelectorAll("*");
    elements.forEach((el) => {
      if (el instanceof HTMLElement) {
        // backgroundColorが指定されていたら削除する
        if (el.style.backgroundColor) {
          el.style.removeProperty("background-color");
        }
        // colorが指定されていたら近い色に変換する
        // closestColor関数は本筋と逸れるため割愛するが、カラーコードをRGBに変換して近い色を返す処理を行っている
        if (el.style.color) {
          const colorName = closestColor(el.style.color);
          if (colorName) {
            el.style.color = ("color", color);
          } else {
            el.style.removeProperty("color");
          }
        }
        // linkの場合はhrefを削除
        if (el.tagName === "A") {
          el.removeAttribute("href");
        }
      }
    });
    content = doc.body.innerHTML;
    delta = delta.concat(this.convert({ html: content }) as DeltaStatic);
    quillObj.updateContents(delta, "user");
    quillObj.setSelection(delta.length() - range.length, "silent");
  }
}

Quill.register("modules/clipboard", PlainClipboard, true);
```

細かく見ていきましょう。

まず、`Quill.import("modules/clipboard")` で Quill の `clipboard` モジュールを取得しています。

`onPaste`メソッドの内部を解説します。

まず、`DOMParser` を使って、ペーストされる HTML をパースし、`querySelectorAll("*")` で全ての要素を取得しています。

```tsx
const parser = new DOMParser();
const doc = parser.parseFromString(html, "text/html");
const elements = doc.body.querySelectorAll("*");
```

そして、それぞれの要素に対して、今回の要件に合う処理を行っています。

```tsx
// backgroundColorが指定されていたら削除する
if (el.style.backgroundColor) {
  el.style.removeProperty("background-color");
}
// colorが指定されていたら近い色に変換する
// closestColor関数は本筋と逸れるため割愛するが、カラーコードをRGBに変換して近い色を返す処理を行っている
if (el.style.color) {
  const colorName = closestColor(el.style.color);
  if (colorName) {
    el.style.color = ("color", color);
  } else {
    el.style.removeProperty("color");
  }
}
// linkの場合はhrefを削除
if (el.tagName === "A") {
  el.removeAttribute("href");
}
```

elements を変更したら、その変更を Delta 形式に変換し、Quill に反映させます。

```tsx
content = doc.body.innerHTML;
delta = delta.concat(this.convert({ html: content }) as DeltaStatic);
quillObj.updateContents(delta, "user");
```

このように、`onPaste` メソッドをオーバーライドすることで、ペースト時の処理をカスタマイズすることができます。
しかし、Quill のアップデートにより Clipboard モジュールの実装が変わる可能性があるため、バージョンアップの際には注意深く行う必要があります。

また、テキストエディタによっては、ペースト時の html が異なった構造の場合もあります。例えば、Mac の純正メモ帳アプリは以下の HTML 構造を持ちます。

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <meta http-equiv="Content-Style-Type" content="text/css" />
    <title></title>
    <meta name="Generator" content="Cocoa HTML Writer" />
    <meta name="CocoaVersion" content="2487" />
    <style type="text/css">
      p.p1 {
        margin: 0px 0px 0px 0px;
        font: 13px "Hiragino Sans";
        color: #fb0007;
      }
      span.s1 {
        text-decoration: line-through;
      }
    </style>
  </head>
  <body>
    <p class="p1"><span class="s1">テスト文章</span></p>
  </body>
</html>
```

それぞれの element に style が指定されておらず、style タグに全てのスタイルが記述されているため、おそらく上記のコードでは動きません。
そのため、先に style タグをパースして、それを元に各要素にスタイルを適用する処理を追加する必要があります。

このように、それぞれのエディターに合わせてペースト処理をカスタマイズする必要があるのが書式設定の辛いところですね。

## まとめ

今回は、WYSIWYG エディタである Quill の紹介と、ペースト時に書式設定を出来るだけ保つ方法について紹介しました。

Quill は豊富な装飾機能を持ち、カスタマイズ性も高いため、作家の方々が文章を書く際に、文章の装飾を簡単に行えるようにすることができます。

また、拡張性も高く、ペースト時の処理以外にも、カスタムフォーマットを追加するなど、様々なカスタマイズが可能です。

今後も機能拡張を続け、作家の方々が快適に文章を書ける環境を提供していきたいと考えています。

弊社では現在エンジニアを積極採用中です。Web フロントエンドの UX に興味がある方や、創作ツールの開発に興味がある方は、ぜひ一度お話しましょう！

https://uzu.one/s/recruit
