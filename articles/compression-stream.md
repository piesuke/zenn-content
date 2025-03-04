---
title: "CompressionStreamを使ってライブラリを導入せずにVercelのRoute Handlersのリミットを一時的に回避する" # 記事のタイトル
emoji: "🧊" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["nextjs", "javascript", "typescript", "approuter", "tech"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。
私たちは、マーダーミステリーを遊べることが出来るアプリ「ウズ」と、マーダーミステリーを制作してウズ上で遊べることが出来るアプリ「ウズスタジオ」を開発しています。
最近良かったマーダーミステリーは「[ABENDLAND -誰そ彼の地-](https://mdms.jp/scenarios/7491)」です。

## はじめに

私たちは運営する Web サイトにおいて Next.js を採用しており、データのやり取りの一部に Vercel の Route Handlers を利用しています。
簡単な API であれば問題ないのですが、リクエストボディ が 4.5MB を超えた場合は [Vercel の制限](https://vercel.com/docs/functions/runtimes#request-body-size)に引っかかり 413 エラーが返ってきてしまいます。
回避策としては Route Handlers に頼らずに自前でサーバーを用意するなどの解決策がありますが、Vercel のリミットを超えるリクエストは全体の 5%も満たないくらいと少なく、また素早く解決する必要があった為構成を変えずに一時的に解決する必要がありました。
これを回避する方法の一つとして、`CompressionStream` を使ってリクエストボディを圧縮して送信する方法を使ったので紹介します。

## CompressionStream とは

最近登場した組み込みのデータ圧縮・展開を行うための API です。
https://developer.mozilla.org/ja/docs/Web/API/CompressionStream

全てのブラウザでサポートされており、gzip 形式または deflate（または deflate-raw）形式を使用してデータストリームを圧縮することができます。

## 使い方

こんな感じで使います。

圧縮時

```ts
const compress = async (data: Record<string, unknown>): Promise<string> => {
  const compressedConfig = JSON.stringify(data);

  const compressedData = await new Response(
    new Blob([compressedConfig])
      .stream()
      .pipeThrough(new CompressionStream("gzip"))
  ).arrayBuffer();

  const bytes = new Uint8Array(compressedData);
  let binary = "";
  for (let i = 0; i < bytes.byteLength; i++) {
    binary += String.fromCharCode(bytes[i]);
  }
  return btoa(binary);
};

const data = { key: "value" };
const compressed = await compress(data);

// 送信
await fetch("https://example.com", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ compressed }),
});
```

解凍時

```ts
const decompress = async (data: string): Promise<string> => {
  // Base64 文字列をバイナリ文字列に変換
  const binaryString = atob(data);

  // バイナリ文字列から Uint8Array を生成
  const len = binaryString.length;
  const bytes = new Uint8Array(len);
  for (let i = 0; i < len; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }

  const decompressed = await new Response(
    new Blob([bytes]).stream().pipeThrough(new DecompressionStream("gzip"))
  ).arrayBuffer();
  const decoded = new TextDecoder().decode(decompressed);
  return decoded;
};

// 解凍
const decompressed = await decompress(compressed);
```

## よかったこと

- リクエストボディを圧縮したことで Vercel のリミットを一時的に回避できた
- サイズが大きいリクエストボディを送信するときのパフォーマンスが向上した

## まとめ

`CompressionStream`は使い勝手が良い組み込みの圧縮・展開 API なので、これからも積極的に利用していきたいと思います。
