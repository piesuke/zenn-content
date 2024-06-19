---
title: "AppRouter移行におけるuseRouterのハマりポイント" # 記事のタイトル
emoji: "💥" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["flutter", "firebase"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。
私たちは、マーダーミステリーを遊べることが出来るアプリ「ウズ」と、マーダーミステリーを制作してウズ上で遊べることが出来るアプリ「ウズスタジオ」を開発しています。
私の好きなマーダーミステリーは「[あなたの原罪](https://mdms.jp/scenarios/2577)」です。

私たちは運営する Web サイトにおいて Next.js を採用しています。今までは PageRouter を使用していましたが、様々な事情により最近 AppRouter に移行することになりました。その際、useRouter の仕様変更が地味に辛く、破壊的変更を行った Next.js への怒りがふつふつと湧いてきました。
なので、今回はその仕様変更と、なぜそのような仕様変更を行う必要があったのかについて書いていきます。

## 対象読者

- AppRouter への移行を検討している人
- useRouter の仕様変更について知りたい人
- なんとなく AppRouter の仕組みは分かっている人

## useRouter の変更点

公式ガイドが分かりやすいので、まずは公式ガイドを読んでみましょう。

### PageRouter の場合

PageRouter の場合の説明はこちら。
https://nextjs.org/docs/pages/api-reference/functions/use-router

> If you want to access the router object inside any function component in your app, you can use the useRouter hook, take a look at the following example:

```tsx
import { useRouter } from "next/router";

function ActiveLink({ children, href }) {
  const router = useRouter();
  const style = {
    marginRight: 10,
    color: router.asPath === href ? "red" : "black",
  };

  const handleClick = (e) => {
    e.preventDefault();
    router.push(href);
  };

  return (
    <a href={href} onClick={handleClick} style={style}>
      {children}
    </a>
  );
}

export default ActiveLink;
```

従来はこのように、useRouter を使って router オブジェクトを取得していました。router オブジェクトとは、pathname や query などの情報を持っているオブジェクトです。
また、push メソッドや replace メソッドなど、ページ遷移に関するメソッドも持っています。

https://nextjs.org/docs/pages/api-reference/functions/use-router#router-object

では続いて、AppRouter の場合を見ていきましょう。

### AppRouter の場合

AppRouter の場合の説明はこちら。

https://nextjs.org/docs/app/api-reference/functions/use-router

> The useRouter hook allows you to programmatically change routes inside Client Components.

```tsx
"use client";

import { useRouter } from "next/navigation";

export default function Page() {
  const router = useRouter();

  return (
    <button type="button" onClick={() => router.push("/dashboard")}>
      Dashboard
    </button>
  );
}
```

大きな変更点として、そもそもの import 先が変わっています。PageRouter の場合は `next/router` でしたが、AppRouter の場合は `next/navigation` に変更されています。
そして、PageRouter における `useRouter`の中でページ遷移に関するメソッドのみが提供されています。そのため、pathname や query などの情報は取得できなくなり、`usePathname`、`useSearchParams`といった新しいフックでそれぞれの情報を取得するようになりました。

## ハマったポイント

軽く変更点を紹介したところで、実際に変更を行う際にハマったポイントを紹介します。

### 1. 認知負荷

全く同名のフックの役割が縮小されていることによる認知負荷は地味に大きかったです。`next/router`は AppRouter では使えませんが Deprecated ではないため特に警告は出ず、実行してから `next/navigation`に変更するべきだったことを知る時がありました。
また、今まで取得できていた pathname や query などの情報を取得するためには新しいフックを使う必要があることになれるまで時間がかかりました。参考の為に主な変更点を以下にまとめます。

### pathname

`PageRouterの場合`

```tsx
import { useRouter } from "next/router";

const router = useRouter();
const pathname = router.pathname;
```

`AppRouterの場合`

```tsx
import { usePathname } from "next/navigation";
const pathname = usePathname();
```

### query Parameters

`PageRouterの場合`

```tsx
import { useRouter } from "next/router";

const router = useRouter();
const query = router.query;
const id = query.id;
```

`AppRouterの場合`

```tsx
import { useSearchParams } from "next/navigation";
const query = useSearchParams();
const id = query.get("id");
```

### Dynamic Parameters

`PageRouterの場合`

```tsx
import { useRouter } from "next/router";

const router = useRouter();
const { id } = router.query;
```

`AppRouterの場合`

```tsx
import { useParams } from "next/navigation";
const params = useParams<{ id: string }>();
const id = params.id;
```

### 2. router.events が使えなくなった

PageRouter の router オブジェクトは、router.events というイベントを提供していました。これはページ遷移時に発火するイベントで、ページ遷移時に何か処理を行いたい場合に便利でした。例えば自分たちは、ページ遷移時に Google Analytics にページ遷移を送信する処理を行っていました。

```tsx
import { useRouter } from "next/router";

const router = useRouter();

useEffect(() => {
  const handleRouteChange = (url: string) => {
    gtag.pageview(url);
  };

  router.events.on("routeChangeComplete", handleRouteChange);

  return () => {
    router.events.off("routeChangeComplete", handleRouteChange);
  };
}, []);
```

しかし AppRouter では router.events がなくなったのでこの方法は使えません。代わりに、useEffect の第二引数に pathname や query などの情報を指定して、その情報が変更された時に処理を行うようにする必要があります。

```tsx
import { usePathname } from "next/navigation";

const pathname = usePathname();
const query = useSearchParams();

useEffect(() => {
  const url = pathname + searchParams.toString();
  gtag.pageview(url);
}, [pathname, query]);
```

これはページの path が変わった時に都度発火するくらいの単純な処理なのでまだ大丈夫でした。しかし、ページ遷移時に何か処理を行いたい場合には、少し面倒になります。例えば、フォームの入力内容が未保存の場合にページ遷移をブロックする処理を行いたい場合などです。
元々 PageRouter では、next/router に依存するページ遷移の時は

```tsx
import { useRouter } from "next/router";

const router = useRouter();

useEffect(() => {
  const handleRouteChange = (url: string) => {
    if (!window.confirm("本当によろしいですか?")) {
      router.events.emit("routeChangeError");
      throw "routeChange aborted";
    }
  };

  router.events.on("routeChangeStart", handleRouteChange);

  return () => {
    router.events.off("routeChangeStart", handleRouteChange);
  };
}, []);
```

のように router.events というイベントを使ってページ遷移をブロックする処理を行うことができました。

また、ブラウザバックボタンの挙動を制御するには、beforePopState を使って以下のように実装することができました。

```tsx
import { useRouter } from "next/router";

const router = useRouter();

router.beforePopState(({ url, as, options }) => {
  if (!window.confirm("本当によろしいですか?")) {
    return false;
    // 省略
  }

  return true;
});
```

しかし、AppRouter では router.events や beforePopState がなくなったため、ページ遷移をブロックする処理を行なうことが難しくなりました。

https://github.com/vercel/next.js/discussions/41934

こちらの discussion にも同様の問題が挙がっており、今後の対応が注目されています。2024/06/19 時点では、Solution とし `window.addEventListener('beforeunload', showModal);`のイベントを使って `event.preventDefault();` を呼び出してページ遷移をブロックする方法が提案されていますが、ブラウザバックが対応してなかったり、`beforeunload`イベントは ios の Safari に対応してなかったと、まだまだ問題が残っているようです。
ですので、フォーム周りの挙動を制御することが重要なプロダクトの場合、AppRouter への移行は慎重に行う必要があります。

## 3. グローバルな router オブジェクトが使えなくなった

PageRouter の場合、useRouter で取得した router オブジェクトはグローバルなオブジェクトでした。そのため、どこからでも router オブジェクトを取得してページ遷移を行うことができました。例えば、React のライフサイクルの外でも

```ts
import Router from "next/router";

Router.router.push("/dashboard");
```

のように router オブジェクトを取得してページ遷移を行ったり、query や pathname を取得することができました。

しかし、AppRouter では router オブジェクトがグローバルなオブジェクトではなくなったため、このような使い方ができなくなりました。この場合、window.location.href や window.history.pushState などのブラウザの API を使ってページ遷移を行う必要があります。

## まとめ

AppRouter によってパフォーマンスが上がるなどの利点がある一方、useRouter 周りはかなり破壊的変更が行われており、移行がかなり面倒な印象でした。next/router を多用している PageRouter のプロジェクトの場合、それぞれのユースケースが AppRouter に対応しているかを確認してから移行を検討することが重要です。
