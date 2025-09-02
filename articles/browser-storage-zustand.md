---
title: "ブラウザストレージの状態管理をzustandで行うと簡単かつ高機能で便利" # 記事のタイトル
emoji: "🐻" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["react", "nextjs", "typescript", "zustand", "localstorage", "tech"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。

私たちは、マーダーミステリーを遊べることが出来るアプリ「ウズ」と、マーダーミステリーを制作してウズ上で遊べることが出来るアプリ「ウズスタジオ」を開発しています。
私が最近やって面白かったマーダーミステリーは [プライト・フライト](https://mdms.jp/scenarios/9762) です。

弊社は状態管理ライブラリに zustand を使用しています。今回は zustand を使ってブラウザストレージへの状態の永続化を簡単かつ高機能に実現する方法を解説します。

## zustand とは

Zustand は、React のためのシンプルで高速な状態管理ライブラリです。「状態」を意味するドイツ語から名付けられました。なぜドイツ語なのかは分かりませんがとてもセンスが良いですね。Redux のような複雑なボイラープレートを必要とせず、React のフックをベースにした直感的な API でグローバルな状態を管理できるのが大きな特徴です。

Zustand は、同じく状態管理ライブラリである Jotai や Valtio を開発した Poimandres というコミュニティによって作られました。これらのライブラリは、それぞれ異なるアプローチで状態管理の問題を解決しようとしています。

- **Zustand**: Redux にインスパイアされた Flux ライクなアプローチを取ります。単一のストア（または複数のストア）を作成し、その中の状態をコンポーネントからフックで購読します。シンプルさと学習の容易さが魅力です。
- **Jotai**: 「アトム」と呼ばれる小さな状態の断片を組み合わせて状態を構築する、アトミックなアプローチを取ります。コンポーネントの再レンダリングを最小限に抑えたい場合に特に強力です。
- **Valtio**: JavaScript の Proxy を利用して、状態オブジェクトへの変更を自動的に追跡します。まるで通常の JavaScript オブジェクトを直接書き換えるかのように状態を更新できるため、非常に直感的に扱えます。

(この三つのOSS全てに関わっている[Daishi さんのインタビュー](https://levtech.jp/media/article/focus/detail_685/)はとても面白いのでぜひ)

これらの選択肢の中で、Zustand は「シンプルさは欲しいけれど、ある程度の構造化も保ちたい」という場合に適した、バランスの取れたライブラリと言えるでしょう。また、zustand は圧倒的にバンドルサイが小さことも魅力的です。

## persist ミドルウェア

そんな zustand はいくつかのミドルウェアを用意しています。その中の一つである`persist` は、Zustand のストアの状態を自動的にストレージ（デフォルトでは `localStorage`）に保存し、ページのリロード後も状態を復元してくれるミドルウェアです。これにより、ユーザーの設定や UI の状態などを簡単に保持することができます。

基本的な使い方は非常にシンプルです。

```typescript
import { create } from "zustand";
import { persist } from "zustand/middleware";

type Store = {
  count: number;
  increase: () => void;
};

const useStore = create<Store>()(
  persist(
    (set) => ({
      count: 0,
      increase: () => set((state) => ({ count: state.count + 1 })),
    }),
    {
      name: "count-storage", // ストレージに保存される際のキー
    }
  )
);
```

これだけで、`count` の状態が `localStorage` に `count-storage` というキーで保存されます。

## 実用的なコード例：`migrate` を使ったスキーマの更新

次に、より実用的な例として、アプリケーションの設定を管理するストアを見てみましょう。
この例では、`version`と`migrate`オプションを使って、後から新しい設定項目（`fontSize`）を追加する、という破壊的変更に安全に対応する方法を扱います。

```typescript
import { create } from "zustand";
import { devtools, persist, PersistOptions } from "zustand/middleware";

// 最新のStateの型
type SettingsState = {
  theme: "light" | "dark";
  fontSize: number;
  setTheme: (theme: "light" | "dark") => void;
  setFontSize: (size: number) => void;
};

// バージョン0の古いStateの型（マイグレーション用）
type OldSettingsState = {
  theme: "light" | "dark";
  // fontSizeが存在しない
};

const persistOptions: PersistOptions<SettingsState> = {
  name: "app-settings-storage",
  version: 1, // 新しいバージョン番号
  migrate: (persistedState, version) => {
    // version 0 から 1 へのマイグレーション
    if (version === 0) {
      const oldState = persistedState as OldSettingsState;
      // 古いStateに新しいプロパティを追加して返す
      return {
        theme: oldState.theme,
        fontSize: 16, // デフォルト値
      };
    }
    return persistedState as SettingsState;
  },
};

const useSettingsStore = create<SettingsState>()(
  devtools(
    persist(
      (set) => ({
        theme: "light",
        fontSize: 16,
        setTheme: (theme) => set({ theme }),
        setFontSize: (size) => set({ fontSize: size }),
      }),
      persistOptions
    ),
    {
      name: "SettingsStore",
    }
  )
);
```

この例では、`persist` の第二引数にいくつかの高度なオプションが指定されています。

### 高度なオプション

#### `version`

`version` は、永続化されるデータのバージョンを管理するためのオプションです。ストアのデータ構造に破壊的変更（例：プロパティ名の変更、型の変更）があった場合に、このバージョン番号をインクリメントします。

`persist`ミドルウェアは、ストレージに保存されているバージョンとコード上で指定された`version`を比較し、異なっていた場合に`migrate`関数を実行します。

#### `migrate`

`migrate`は、`version`の不一致が検出されたときに実行される、スキーマ変更を吸収するための関数です。

上記のコード例では、`version: 1` と指定しています。もしユーザーのストレージに `version: 0` のデータが保存されていた場合、`migrate`関数が呼び出されます。

1.  `migrate`関数は引数として、永続化されていた古い状態 (`persistedState`) とそのバージョン (`version`) を受け取ります。
2.  関数内では、バージョン番号をチェックし（`if (version === 0)`）、どのバージョンからの移行処理なのかを判断します。
3.  古い状態の型 (`OldSettingsState`) に基づいてデータにアクセスし、新しい状態の型 (`SettingsState`) に合うように、不足している`fontSize`プロパティにデフォルト値を追加して返します。

この仕組みにより、ユーザーが過去に設定した`theme`の値を保持したまま、安全に新しい`fontSize`プロパティを追加できます。`migrate`関数が提供されていない場合、`version`が異なるとストレージのデータは破棄され、ストアはコード上の初期値でリセットされます。

## まとめ

Zustand の `persist` ミドルウェアは、単に状態を永続化するだけでなく、`version` や `migrate` といった高度なオプションを使うことで、より複雑な要件にも対応できる非常に強力な機能です。

状態管理と永続化のロジックをストア内にきれいにカプセル化できるため、コンポーネントのコードをクリーンに保つことにも繋がります。ぜひ活用してみてください。
