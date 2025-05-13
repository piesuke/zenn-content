---
title: "リッチテキスト（Quill）をOpenAI APIで翻訳するときの tips" # 記事のタイトル
emoji: "🧊" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["ai", "tech"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。
私たちは、マーダーミステリーを遊べることが出来るアプリ「ウズ」と、マーダーミステリーを制作してウズ上で遊べることが出来るアプリ「ウズスタジオ」を開発しています。
最近良かったマーダーミステリーは「[黒と白の狭間に](https://mdms.jp/scenarios/345)」です。

## はじめに

弊社では海外展開に合わせ、アプリ内の文言を多言語化しています。ワンライナーの UI ラベルや通知文は i18n ライブラリに載せるだけですが、リッチテキスト（マークダウンや Quill など）になると話は別です。文字装飾・リスト・埋め込み要素など、構造そのものが UX の一部であり、単にテキストを訳すだけでは済まないからです。

本記事では、弊社が実運用する Quill Delta × OpenAI を例に、いくつかの tips を解説します。Quill 以外のリッチテキスト（Draft.js Raw、ProseMirror JSON など）でも応用可能です。

## 1. リッチテキストは基本的に構造ごと渡す

Quill は内部で Delta と呼ばれる操作履歴ベースの JSON を持ちます。

```json
{
  "ops": [
    { "insert": "Hello " },
    { "insert": "world", "attributes": { "bold": true } },
    { "insert": "!\n" }
  ]
}
```

初期は ops の配列の各要素ごとに翻訳をかけていましたが、装飾で一単語のみ分けられている箇所などでは前後の文脈を考慮しない翻訳になってしまう事があり、ops の中身ごと渡すという方法に切り替えました。
しかし、トークンのサイズが大きくなってくると逆に翻訳の精度が落ちるので、見出しや改行など前後の文脈に比較的影響を受けない箇所で区切るというアプローチを取っても良いでしょう。
とはいえ Gemini 2.5pro など、コンテキストウインドウが大きなモデルだとさほど気にしなくてもいいのかもしれません。

また、構造ごと渡して翻訳させたら稀に装飾要素が抜け落ちる時があります。そのような事象を防ぐためにプロンプトを工夫します。

### プロンプト例

英語のプロンプトを渡す方が精度が高まる気がするので英語で渡しています。

> Input is a Quill Delta Object. Ensure that the output also adheres to the JSON and Quill format. When translating, make sure no meaning is lost, the context is preserved and the number of attribute keys and their contents are the same.

ここでは、翻訳時に気をつけることを列挙しています。

- 意味を不必要に取りこぼさないように
- キーの数とそれぞれのコンテンツが同じであることを確認するように

## 2. AI にセルフチェックさせる二段階プロンプト

以下のようなステップで翻訳する事で、AI がセルフチェックしてくれます。

1. 翻訳ステップ：Delta を訳す。

2. レビュー ステップ：別プロンプトで「元の Delta と翻訳後 Delta を比較し、構造破壊・未訳を列挙せよ」と指示。

3. 改善ステップ：列挙結果を System メモリに渡し、修正翻訳を再出力させる。

大きなトークンサイズの Delta を渡すと一部が未翻訳のままになってしまう事象がたまに発生したり、装飾の要素が変わる事があるので一度 AI によるセルフチェックを挟むことでそのようなミスを自己修正してくれます。

2 のプロンプトは以下のように設定しています。

```ts
const reviewPrompt = `Based on the results of the ${targetLangLabel} translation, identify specific issues. Avoid vague expressions and provide accurate descriptions. 
There is no need to add content or formats that were not present in the original text. This includes but is not limited to: 
・If it does not conform to ${targetLangLabel} expression habits, clearly indicate where it does not conform. 
・For clumsy sentences, specify the location; there is no need to offer suggestions for modification as this will be fixed during free translation. 
・If it is obscure and difficult to understand, attempts to explain may be made. However, if the ${targetLangLabel} characters included match those in the following list of technical terms, please do not propose any changes. If characters with different forms but the same meanings as those in the list of technical terms are used, please propose corrections. 
Technical terms are these. ${termsList}.`;
```

ここでは、
・曖昧な表現や分かりにくい訳を行ってないか
・翻訳を行う言語の表現習慣に合っているかどうか
・後述する表記揺れ

などをレビューしてもらっています。

## 3. 表記揺れを防ぐグロッサリー

英語では “ログイン” → "Sign‑in" と "Log‑in" が混在しがち、など、AI は文脈に応じて語を変えます。そこで、固定訳したい用語リストを JSON で渡し、System プロンプトで "以下の用語は必ず置換表を用いて訳せ" と宣言します。

```json
{
  "Login": "Sign in",
  "ユーザー": "User",
  "ダッシュボード": "Dashboard"
}
```

グロッサリーの管理方法については後日別のメンバーから記事が出る予定なので乞うご期待です。

## まとめ

リッチテキストは単純な文章ではない分、翻訳となるといくつか気をつける事があります。しかし AI モデルの進化により日に日に精度が上がっているのも実感しています。将来的にはここの tips を用いなくとも十分な精度を出してくれる可能性も十分にあるので、参考程度にしていただけますと幸いです。

また、弊社では他にも翻訳に関して様々な取り組みを行なっております。そちらも併せてご覧いただけると、更に多言語対応に対する知見が深まると思います。

https://zenn.dev/articles/3cdcb2fa33463d/
