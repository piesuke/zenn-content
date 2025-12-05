---
title: "Reactにおける共通コンポーネント設計のアンチパターン" # 記事のタイトル
emoji: "🧊" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["nextjs", "javascript", "typescript", "tech"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。
弊社では、マーダーミステリーをプレイできるアプリ「ウズ」と、マーダーミステリーを制作しウズ上で利用できるアプリ「ウズスタジオ」を開発しています。
最近良かったマーダーミステリーは「[青いクジラは沈まない](https://mdms.jp/scenarios/5257)」です。

複数のWebサービスを展開する弊社では、開発効率の向上と車輪の再発明防止のため、頻繁に使用されるコンポーネントを共通コンポーネントとして管理しています。
しかし、共通コンポーネントは多岐にわたる箇所での利用を想定する必要があり、設計上の考慮が不足すると、結果として使いづらいコンポーネントとなることがあります。

本稿では、共通コンポーネントの作成・運用経験から得られたアンチパターンを紹介します。

## 前提：弊社のコンポーネントの共通化の運用
- turborepoを使ったモノレポで運用している
- デザイン時に何度も使われるコンポーネントを切り出し、packages/uiに共通コンポーネントとして実装している
- それをapps/配下の各サービスで呼び出している

## アンチパターン①：HTML 標準の型から逸脱した props を作る

共通コンポーネントの設計においては、「素のHTMLが持つ振る舞いを損なわない」ことが前提となります。例えば`button`要素であれば、`type`、`disabled`、`aria-*`属性、`onClick`イベントハンドラなど、HTML標準の属性や型に準拠させるべきです。

### よくある NG 実装

`onClick`イベントハンドラに独自の`value`のみを渡すような実装は、保守性や拡張性を著しく低下させます。HTML標準の属性やイベント型から逸脱すると、将来的な拡張において問題が生じます。

```tsx
// NG: HTML の属性が拾えない/型が逸脱しているボタン
type Props = {
  label: string;
  // 本来は React.MouseEventHandler<HTMLButtonElement>
  onClick?: (value: string) => void;
  value?: string;
};

export const BadButton: React.FC<Props> = ({ label, onClick, className, value }) => {
  return (
    <button
      // type/disabled/aria-* などを拾えない
      className="inline-flex items-center rounded-md bg-blue-600 px-4 py-2 text-white"
      onClick={() => onClick?.(value ?? label)}
    >
      <span>{label}</span>
    </button>
  );
};
```

この設計では、`type="submit"`や`aria-label`の付与といった基本的なHTML属性の利用が困難になります。また、`onclick`時にイベントの伝播を止める場合がある時などに実現が困難になります。

### 改善例

HTMLの属性をそのまま受け渡し、イベントハンドラは標準の型を使用します。

```tsx
import React from "react";

type ButtonProps = React.ButtonHTMLAttributes<HTMLButtonElement> & {
  variant?: "primary" | "secondary";
};

export const Button: React.FC<ButtonProps> = ({ className, variant = "primary", ...rest }) => {
  const base = "inline-flex items-center justify-center rounded-md px-4 py-2 text-sm font-medium focus:outline-none focus:ring-2 focus:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50";
  const style =
    variant === "primary"
      ? "bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-600"
      : "bg-gray-200 text-gray-900 hover:bg-gray-300 focus:ring-gray-400";
  return (
    <button className={[base, style, className].filter(Boolean).join(" ")} {...rest} />
  );
};

// 使い方: event から value 等を取りたければ標準の方法で取れる
<Button type="submit" value="hello" onClick={(e) => console.log(e.currentTarget.value)} />;
```
`Button` や `Input`などは、出来るだけHTMLの属性をそのまま受け渡した方が後々追加の要件が発生した時にも対応できます。

---

## アンチパターン②：props が抽象的すぎる（結果として繰り返し実装が生じる）

「汎用性を持たせることであらゆる状況に対応できる」という考え方は、共通コンポーネントにおいては予期せぬ問題を引き起こすことがあります。抽象度を過度に高めると、コンポーネントの呼び出し側で定型的な実装が繰り返され、結果としてコードの重複が増加します。

### よくある NG 実装

アクションを配列で受け取り、`onAction`に種別文字列を返却する汎用的なダイアログは、利用側で常にOK/Cancelなどのアクションを再構築する必要が生じます。

```tsx
type Action = { label: string; type: string };

type DialogProps = {
  title: string;
  isOpen: boolean;
  onClose: () => void;
  actions: Action[]; // 何でも入る
  onAction: (type: string) => void; // 何でも返す
};

export const AnyDialog: React.FC<React.PropsWithChildren<DialogProps>> = ({
  title,
  isOpen,
  onClose,
  actions,
  onAction,
  children,
}) => {
  if (!isOpen) return null;
  return (
    <div className="fixed inset-0 z-50">
      <div className="fixed inset-0 bg-black/50" onClick={onClose} />
      <div className="fixed inset-0 flex items-center justify-center p-4">
        <div role="dialog" aria-modal className="w-full max-w-md rounded-lg bg-white shadow-xl">
          <div className="border-b p-4 text-lg font-semibold">{title}</div>
          <div className="p-4">{children}</div>
          <div className="flex justify-end gap-2 border-t p-3">
            {actions.map((a) => (
              <button
                key={a.type}
                className="rounded-md bg-gray-200 px-3 py-2 text-sm hover:bg-gray-300"
                onClick={() => onAction(a.type)}
              >
                {a.label}
              </button>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
};

// 利用のたびに類似の実装が増加する例
<AnyDialog
  title="削除しますか？"
  isOpen={open}
  onClose={() => setOpen(false)}
  actions={[
    { label: "キャンセル", type: "cancel" },
    { label: "削除する", type: "confirm" },
  ]}
  onAction={(t) => (t === "confirm" ? doDelete() : setOpen(false))}
>
  この操作は取り消せません。
</AnyDialog>;
```

### 改善例

用途が固定されている場合は、明確な用途別コンポーネント（または明確なvariant）として切り出すべきです。これにより、具体的な型を用いることで、呼び出し側での実装の重複を排除できます。

```tsx
type ConfirmDialogProps = {
  title: string;
  isOpen: boolean;
  confirmText?: string;
  cancelText?: string;
  destructive?: boolean;
  onConfirm: () => void;
  onCancel: () => void;
};

export const ConfirmDialog: React.FC<React.PropsWithChildren<ConfirmDialogProps>> = ({
  title,
  isOpen,
  confirmText = "OK",
  cancelText = "キャンセル",
  destructive,
  onConfirm,
  onCancel,
  children,
}) => {
  if (!isOpen) return null;
  return (
    <div className="fixed inset-0 z-50">
      <div className="fixed inset-0 bg-black/50" onClick={onCancel} />
      <div className="fixed inset-0 flex items-center justify-center p-4">
        <div role="dialog" aria-modal className="w-full max-w-md rounded-lg bg-white shadow-xl" onClick={(e) => e.stopPropagation()}>
          <div className="border-b p-4 text-lg font-semibold">{title}</div>
          <div className="p-4">{children}</div>
          <div className="flex justify-end gap-2 border-t p-3">
            <button className="rounded-md bg-gray-200 px-3 py-2 text-sm hover:bg-gray-300" onClick={onCancel}>
              {cancelText}
            </button>
            <button
              className={["rounded-md px-3 py-2 text-sm text-white", destructive ? "bg-red-600 hover:bg-red-700" : "bg-blue-600 hover:bg-blue-700"].join(" ")}
              onClick={onConfirm}
            >
              {confirmText}
            </button>
          </div>
        </div>
      </div>
    </div>
  );
};

// コードの重複が解消され、意図が明確な利用例
<ConfirmDialog
  title="このアイテムを削除"
  isOpen={open}
  destructive
  onConfirm={doDelete}
  onCancel={() => setOpen(false)}
>
  この操作は取り消せません。
</ConfirmDialog>;
```

抽象化は「必要最小限の範囲で」行うのが原則です。用途が限定される場合はpropsを積極的に具体化し、可読性と一貫性の向上を図るべきです。

---

## アンチパターン③：共通コンポーネントで細かい CSS を盛り込みすぎる

共通コンポーネント内でマージン、パディング、見出しサイズといった「文脈依存のスタイル」を決定してしまうと、利用側の意図に合致しないデザインとなる可能性があります。レイアウトに関する責務は、可能な限りコンポーネントの呼び出し側に委譲すべきです。

### よくある NG 実装

```tsx
// NG: 文脈依存の余白やネストスタイルを内包してしまうカード
export const BadCard: React.FC<{ className?: string }> = ({ className, children }) => (
  <section
    className={[
      "w-full rounded-md bg-white p-6 shadow-md",
      "mt-8", // 呼び出し側のレイアウトを勝手に決める
      "[&_h3]:mt-6",
      "[&_footer>button+button]:ml-3",
      className,
    ]
      .filter(Boolean)
      .join(" ")}
  >
    {children}
  </section>
);
```

`margin`や内部要素のスタイルが固定されると、「カードを横並びに配置する」「親要素で`gap`プロパティを利用する」といった異なるレイアウトコンテキストにおいて、予期せぬ表示崩れが発生します。

### 改善例

コンポーネントは、背景色、枠線、角丸などの最小限の視覚的表現に留めるべきです。外部のレイアウトは呼び出し側が決定できるように、`grid`や`flex`といったレイアウトシステムを介して構成します。

```tsx
type CardProps = React.HTMLAttributes<HTMLElement> & {
  tone?: "neutral" | "elevated";
};

export const Card: React.FC<CardProps> = ({ className, tone = "neutral", ...rest }) => {
  const base = "rounded-md bg-white";
  const elevation = tone === "elevated" ? "shadow-md" : "shadow-none";
  return <section className={[base, elevation, className].filter(Boolean).join(" ")} {...rest} />;
};

// 利用例: 余白や配置は呼び出し側の責務
<div className="grid grid-cols-2 gap-4">
  <Card tone="elevated" className="p-6 space-y-3">
    <h3 className="text-lg font-bold">タイトル</h3>
    <p>本文</p>
    <footer className="flex gap-2">
      <Button>キャンセル</Button>
      <Button variant="primary">保存</Button>
    </footer>
  </Card>
  <Card className="p-6">任意の余白は className で</Card>
</div>
```

「コンポーネントは中立性を保ち、文脈は呼び出し側が決定する」という責務分離を遵守することで、予期せぬ表示崩れを抑制し、堅牢な共通化を実現できます。

## まとめ

これらの原則を守ることで、再利用性が高く、堅牢で保守しやすい共通コンポーネントを設計することができます。

共通コンポーネントは適切に作成することで大幅に開発の生産性を上げることができるので、ぜひ適切に作成してみてください。


