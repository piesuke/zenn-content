---
title: "Modal系のライブラリはどう作られているのか調べてみた" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["react", "chakraui", "css"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。

最近面白かったマーダーミステリーは、[カモメのピースが揃う時](https://mdms.jp/scenarios/2909) です。

弊社ではマーダーミステリーを遊べるアプリ「ウズ」と、マーダーミステリーを制作してウズ上で遊べるアプリ「ウズスタジオ」を開発しています。
絶賛エンジニア募集中です！詳しくは [こちら](https://sally-inc.super.site/) をご覧ください。

## はじめに

弊社では可能な限りサードパーティーのライブラリを使用せず、共通コンポーネントを自作する方針を採っていますが、モーダルダイアログのライブラリは自前で一から実装するのは大変だと思い、サードパーティーのライブラリを使っています。

モーダルライブラリは適切な props を渡すだけで簡単に利用できるため、その実装方法について調べてみました。
ちなみに弊社では [Chakra UI](https://chakra-ui.com/) を使っているので、Chakra UI の Modal を例に調べてみました。

## Chakra UI の Modal

仕様は [こちら](https://v2.chakra-ui.com/docs/components/modal) に記載されています。
わざわざ仕様のページを見に行くのが面倒なあなたの為に基本的なコードを引用すると、

```tsx
function BasicUsage() {
  const { isOpen, onOpen, onClose } = useDisclosure();
  return (
    <>
      <Button onClick={onOpen}>Open Modal</Button>

      <Modal isOpen={isOpen} onClose={onClose}>
        <ModalOverlay />
        <ModalContent>
          <ModalHeader>Modal Title</ModalHeader>
          <ModalCloseButton />
          <ModalBody>
            <Lorem count={2} />
          </ModalBody>

          <ModalFooter>
            <Button colorScheme="blue" mr={3} onClick={onClose}>
              Close
            </Button>
            <Button variant="ghost">Secondary Action</Button>
          </ModalFooter>
        </ModalContent>
      </Modal>
    </>
  );
}
```

`Modal`でラップして、`ModalOverlay`、`ModalContent`、`ModalHeader`、`ModalCloseButton`、`ModalBody`、`ModalFooter` などのコンポーネントを使ってモーダルを作成します。

### Modal の実装

`Modal`のソースコードは以下のようになっています。
https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/modal/modal.tsx

`Modal`はラッパーとして機能しつつ、子コンポーネントにコンテキストやイベントハンドリング機能を提供しています。
https://github.com/chakra-ui/chakra-ui/blob/969fe57c97540cafa736ff64cfdff970c948c810/packages/components/src/modal/modal.tsx#L190

context は、`Modal` コンポーネントの Props として渡される値と、 `useModal`というカスタムフックで生成された値をマージして作成されています。
どうやら `useModal` が `Modal` のロジックを担っているようです。次は `useModal` のソースコードを見てみましょう。

### useModal の実装

`useModal`のソースコードは以下のようになっています。
https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/modal/use-modal.ts

`useModal`がやってることは、大まかに以下の通りです。

- `useIds`を利用して、コンポーネント内で必要な複数のユニークな ID を一括生成し、他の要素と衝突しないようにする
- モーダルが開いている時にそれ以外の属性に `aria-hidden` を付与したり、`aria-labelledby`、`aria-describedby`といった https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/ の仕様に従って属性を付与
- `useModalManager`を呼び出してダイアログに index を付与
- オーバーレイクリック時や、キーボード、マウスイベントのハンドリング

この中で面白いなーと思ったのは ARIA の対応と modalManager の使い方です。
ARIA の対応は、モーダルが開いている時に他の要素に `aria-hidden` を付与することで、スクリーンリーダーがそれらの要素を無視するようにし、ユーザーがモーダル外の要素にフォーカスしないようにします。
また、ヘッダーに `aria-labelledby`、ボディに `aria-describedby` といった属性を付与することで、スクリーンリーダーがモーダルの内容を読み上げる際にどの要素を読み上げるかを指定しています。これを props の値を元に簡単なロジックで実現しているのが面白いなと思いました。

また、`useModalManager`は、モーダルの index を管理するためのカスタムフックです。複数のモーダルが同時に開かれている場合に、それぞれのモーダルのインデックスを管理するために使われています。
こちらも、シンプルながら汎用性の高い使い方をしているなと感じました。

### 全画面表示を行う為に DOM ツリーの外に描画する Portal

Modal の便利な点として、コンポーネントの中のどこに書いても全画面表示になるという点があります。これは、`Portal`というコンポーネントを使って実現されています。
ソースのどこに書いても全画面表示になるように、 `Portal`を使っています。
https://github.com/chakra-ui/chakra-ui/blob/v2/packages/components/src/portal/portal.tsx

例えば `DefaultPortal`を見てみると、
https://github.com/chakra-ui/chakra-ui/blob/969fe57c97540cafa736ff64cfdff970c948c810/packages/components/src/portal/portal.tsx#L37

`useSafeLayoutEffect` を用いて、レイアウトの副作用を安全に処理し、`createPortal` によって Portal の中身を document.body に追加しています。

https://github.com/chakra-ui/chakra-ui/blob/969fe57c97540cafa736ff64cfdff970c948c810/packages/components/src/portal/portal.tsx#L51

この処理によって、どこで書いても DOM ツリーの外に描画されるようになり、全画面表示を行うことができるようになっています。

## まとめ

ここまで複雑な処理を行っているとなると、自作で実装するのは大変そうですね。全ての UI ライブラリに感謝したいです。

a11y の対応や、Portal の使い方など、実装の参考になる点が多かったです。今後もライブラリのソースコードを読んで、学びを得ていきたいと思います。