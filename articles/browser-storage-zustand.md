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

Zustand は、React のためのシンプルで高速な状態管理ライブラリです。「状態」を意味するドイツ語から名付けられました。Redux のような複雑なボイラープレートを必要とせず、React のフックをベースにした直感的な API でグローバルな状態を管理できるのが大きな特徴です。

Zustand は、同じく状態管理ライブラリである Jotai や Valtio を開発した Poimandres というコミュニティによって作られました。これらのライブラリは、それぞれ異なるアプローチで状態管理の問題を解決しようとしています。

- **Zustand**: Redux にインスパイアされた Flux ライクなアプローチを取ります。単一のストア（または複数のストア）を作成し、その中の状態をコンポーネントからフックで購読します。シンプルさと学習の容易さが魅力です。
- **Jotai**: 「アトム」と呼ばれる小さな状態の断片を組み合わせて状態を構築する、アトミックなアプローチを取ります。コンポーネントの再レンダリングを最小限に抑えたい場合に特に強力です。
- **Valtio**: JavaScript の Proxy を利用して、状態オブジェクトへの変更を自動的に追跡します。まるで通常の JavaScript オブジェクトを直接書き換えるかのように状態を更新できるため、非常に直感的に扱えます。

これらの選択肢の中で、Zustand は「シンプルさは欲しいけれど、ある程度の構造化も保ちたい」という場合に適した、バランスの取れたライブラリと言えるでしょう。

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

## 実用的なコード例

次に、もう少し実用的な例として、アプリケーションの設定を管理するストアを見てみましょう。
この例では、`version`と`merge`オプションを使って、後から新しい設定項目（フォントサイズ）を追加するシナリオを扱います。

```typescript
import { create } from "zustand";
import { devtools, persist } from "zustand/middleware";

type SettingsState = {
  theme: "light" | "dark";
  // このプロパティは後から追加されたと仮定します
  fontSize: number;
  setTheme: (theme: "light" | "dark") => void;
  setFontSize: (size: number) => void;
};

const useSettingsStore = create<SettingsState>()(
  devtools(
    persist(
      (set) => ({
        theme: "light",
        fontSize: 16, // 新しいプロパティのデフォルト値
        setTheme: (theme) => set({ theme }),
        setFontSize: (size) => set({ fontSize: size }),
      }),
      {
        name: "app-settings-storage",
        version: 1, // fontSize追加時に0から1に上げた、などのシナリオを想定
        // 新しいプロパティを安全に追加するためのカスタムマージ戦略
        merge: (persistedState, currentState) => {
          // `currentState` は新しいデフォルト値を含むコード上の最新の状態
          // `persistedState` はストレージにある古い状態（fontSizeを含まない可能性がある）
          // これらをマージすることで、既存の `theme` 設定を維持しつつ、新しい `fontSize` を安全に追加します。
          return {
            ...currentState,
            ...(persistedState as object),
          };
        },
      }
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

`version` は、永続化されるデータのバージョンを管理するためのオプションです。
ストアのデータ構造に破壊的変更（例：プロパティ名の変更、型の変更）があった場合に、このバージョン番号をインクリメントします。

バージョンが古いままのデータがストレージに残っていると、アプリケーションが予期せぬエラーを起こす可能性があります。`persist` は、コードで指定された `version` とストレージに保存されている `version` が異なる場合、デフォルトではストレージのデータを破棄して初期状態に戻してくれます。これにより、安全にデータ構造の変更を行うことができます。

#### `merge`

`merge` は、ストレージから復元された永続化データ (`persisted`) と、現在のコード上の初期状態 (`current`) をどのようにマージするかをカスタマイズするための関数です。

デフォルトの `merge` は単純な `Object.assign` のような動作をしますが、これでは不十分な場合があります。

上記のコード例では、`merge`関数が重要な役割を果たします。例えば、ストアの`version`を`0`から`1`に上げ、新たに`fontSize`という設定項目を追加したとします。このとき、ユーザーの`localStorage`にはまだ`fontSize`を含まない古い状態が保存されています。

`merge`関数は、ストレージから読み込んだ古い状態（`persistedState`）と、コード上の新しい初期状態（`currentState`）をどのように組み合わせるかを定義します。この例の`{ ...currentState, ...(persistedState as object) }`という実装により、ユーザーが以前設定した`theme`はそのままに、新しい`fontSize`のデフォルト値（16）が安全に追加されます。これにより、破壊的変更を避けつつ、スムーズに状態のスキーマを更新できます。

## まとめ

Zustandの `persist` ミドルウェアは、単に状態を永続化するだけでなく、`version` や `merge` といった高度なオプションを使うことで、より複雑な要件にも対応できる非常に強力な機能です。

状態管理と永続化のロジックをストア内にきれいにカプセル化できるため、コンポーネントのコードをクリーンに保つことにも繋がります。ぜひ活用してみてください。
