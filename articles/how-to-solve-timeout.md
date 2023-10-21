---
title: "api routesの処理がタイムアウトする問題を解決するために行ったこと" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["next.js", "vercel"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## TL;DR

api routes の処理がタイムアウトする問題を解決するために、以下のことを行った。

- 非同期処理を並列で行う。
- api routes の maxDuration を延長する。
- vercel の serverless functions の Region を変更する。

## 直面した課題

Next.js でサービスの開発を行っているが、api routes を使って Google Cloud Storage に保存した画像の URL を取得する処理がタイムアウトしてしまう問題に直面した。
取得すべき画像の数は 500 件程度で、それぞれの画像の URL を取得する処理は非同期で行っている。

## 行ったこと

### ① 非同期処理を並列で行う。

Promise.all を使って非同期処理を並列で行うようにした。

コード例

```ts
const urls = await Promise.all(
  images.map(async (image) => {
    const url = await storage.bucket(bucketName).file(image).getSignedUrl({
      action: "read",
      expires: "03-09-2491",
    });
    return url[0];
  })
);
```

しかし、500 件の画像 URL を一度に取得しようとすると 500 エラーになってしまったので、任意の件数に分割して取得するようにした。

コード例(ほぼ chatGPT パイセンが書いてくれた。こういうよくありそうな処理を一発で書いてくれるのでありがたい。)

```ts
async function limitConcurrencyWithInterval<T>(
  tasks: Promise<T>[],
  limit: number,
  interval: number,
): Promise<T[]> {
  const results: T[] = [];
  const executing: Promise<void>[] = [];

  for (const task of tasks) {
    const p = task.then((r) => {
      results.push(r);
      const index = executing.indexOf(p);
      if (index !== -1) {
        executing.splice(index, 1);
      }
    });

    executing.push(p);

    if (executing.length >= limit) {
      try {
        await Promise.all(executing);
        await sleep(interval); // インターバルを追加
      } catch (error) {
        console.error('Promise.race error:', error);
        throw new Error('Promise.race error');
      }
    }
  }

  const result = await limitConcurrencyWithInterval(tasks, limit, interval);

```

これでいけるか！？と思ったが状況は変わらず。ということで vercel の設定を見直してみることにした。

### ② api routes の maxDuration を延長する。

どうやら api routes の maxDuration は 15 秒に設定されているらしい。これを延長することで解決できるかもしれないと思い、延長した。page Router と App Router で設定の方法は違うらしいが、今回は Page Router なので、以下のように設定した。

api routes のファイルの上部に以下のように記述する。

```ts
export const config = {
  maxDuration: 〇〇,
};
```

しかしまだタイムアウトが発生したので、以下の対策を行った。

### ③ vercel の serverless functions の Region を変更する。

どうやら vercel の serverless functions の Region は気軽に変更できるらしい。
https://vercel.com/docs/functions/serverless-functions/regions

Google Cloud Storage の置き場が東京だったので、vercel の方も東京に変更した。

これで無事に解決した。

## まとめ

そろそろ api routes の限界を感じ始めているので、きちんとサーバーを立てていく必要があるかもしれない。
