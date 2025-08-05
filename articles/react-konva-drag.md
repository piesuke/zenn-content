---
title: "【React-konva】Shapeをホバーした時にDevモードのFigmaっぽく表示する" # 記事のタイトル
emoji: "🧊" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["tech", "react", "konva"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。
私たちは、マーダーミステリーを遊べることが出来るアプリ「ウズ」と、マーダーミステリーを制作してウズ上で遊べることが出来るアプリ「ウズスタジオ」を開発しています。
最近良かったマーダーミステリーは「[新世界のユキサキ](https://mdms.jp/scenarios/8205)」です。

今回はかなりニッチですが、React-Konvaを使ってShapeをホバーした時にDevモードのFigmaっぽく表示する実装の解説を行いたいと思います。

## 前提

- React-Konvaの基本的な知識

## DevモードのFigmaっぽくとは

こんな感じ。
![](/images/CleanShot 2025-08-05 at 10.49.42.gif)

### ポイント

- ホバーした際にオブジェクトの上部にラベルが表示される
- ズームイン・ズームアウトによってラベルのサイズが調整される

## 実装

```tsx
import { Group, Layer, Rect, Stage, Tag, Text } from "react-konva";
import { useState, useEffect, useMemo, useRef } from "react";
import Konva from "konva";

export default function App() {
  const [dimensions, setDimensions] = useState({ width: 800, height: 600 });
  const [scale, setScale] = useState({ x: 1, y: 1 });
  // 現在のScaleを取得するためにStageのrefを使用
  const stageRef = useRef<Konva.Stage>(null);

  useEffect(() => {
    setDimensions({
      width: window.innerWidth,
      height: window.innerHeight,
    });
  }, []);

  useEffect(() => {
    if (stageRef.current) {
      console.log(stageRef.current.scaleX(), stageRef.current.scaleY());
      setScale({ x: stageRef.current.scaleX(), y: stageRef.current.scaleY() });
    }
  }, [stageRef.current]);

  const TEXT = "サンプルRect";

  // 描画に必要な値を定数で指定
  const IMAGE_SIZE = 300;
  const IMAGE_X = dimensions.width / 2 - IMAGE_SIZE / 2;
  const IMAGE_Y = dimensions.height / 2;
  const IMAGE_PADDING = 10;
  const RECT_PADDING = 4;
  const FONT_SIZE = 12;
  const LINE_HEIGHT = 1.2;
  const TEXT_MAX_WIDTH = 100;
  const TAG_Y_PADDING = 10;
  const TAG_HEIGHT = 24;

  // widthを取得するために一時的にKonva.Textを使用
  const textWidth = useMemo(() => {
    const tempText = new Konva.Text({
      text: TEXT,
      fontSize: FONT_SIZE,
      padding: IMAGE_PADDING,
      lineHeight: LINE_HEIGHT,
      x: IMAGE_X + IMAGE_SIZE,
      letterSpacing: 1.2,
      y: 0,
      ellipsis: true,
    });
    const width = tempText.width();
    tempText.destroy();
    return Math.min(width, TEXT_MAX_WIDTH);
  }, []);

  // onWheel時にマウスポインターの位置を中心にScaleの値を変更する
  const onHandleScale = (e: Konva.KonvaEventObject<WheelEvent>) => {
    const stage = e.target.getStage();
    if (!stage) return;
    
    const oldScale = stage.scaleX();
    const pointer = stage.getPointerPosition();
    if (!pointer) return;

    const mousePointTo = {
      x: (pointer.x - stage.x()) / oldScale,
      y: (pointer.y - stage.y()) / oldScale,
    };

    const direction = e.evt.deltaY > 0 ? -1 : 1;
    const scaleBy = 1.05;
    const newScale = direction > 0 ? oldScale * scaleBy : oldScale / scaleBy;

    stage.scale({ x: newScale, y: newScale });

    const newPos = {
      x: pointer.x - mousePointTo.x * newScale,
      y: pointer.y - mousePointTo.y * newScale,
    };
    stage.position(newPos);

    setScale({ x: newScale, y: newScale });
  }

  const tagWidth = useMemo(() => textWidth + RECT_PADDING, [textWidth]);

  const [isHover, setIsHover] = useState(false);


  return (
    <Stage
      width={dimensions.width}
      height={dimensions.height}
      ref={stageRef}
      onWheel={(e) => {
        e.evt.preventDefault();
        // ホイール中はラベルを非表示にするというFigmaの仕様を再現
        setIsHover(false);
        
        onHandleScale(e);
      }}
    >
      <Layer>
        <Rect
          x={IMAGE_X}
          y={IMAGE_Y}
          width={IMAGE_SIZE}
          height={IMAGE_SIZE}
          fill="red"
          shadowBlur={10}
          draggable
          onMouseEnter={() => setIsHover(true)}
          onMouseMove={() => {
            setIsHover(true);
          }}
          onMouseLeave={() => setIsHover(false)}
        />
        {isHover && stageRef.current && (
          <Group
            x={IMAGE_X}
            y={IMAGE_Y - (TAG_HEIGHT / scale.y) - (TAG_Y_PADDING / scale.y)}
            // スケールサイズごとにラベルのサイズを調整
            scaleX={1 / scale.x}
            scaleY={1 / scale.y}
          >
            <Tag
              width={tagWidth}
              height={TAG_HEIGHT}
              fill="#4A46D4"
              cornerRadius={99}
            />
            <Text
              text={TEXT}
              fill={"white"}
              fontSize={FONT_SIZE}
              x={RECT_PADDING / 2}
              y={TAG_HEIGHT / 2}
              offsetY={6}
              align="center"
              width={tagWidth - RECT_PADDING}
            />
          </Group>
        )}
      </Layer>
    </Stage>
  );
}
```

## 実際の表示

![](/images/CleanShot 2025-08-05 at 11.36.02.gif)

## 解説

### ホバー時のラベル表示機能

ホバーした際にオブジェクトの上部にラベルが表示される機能は、以下の実装で実現されています：

```tsx
const [isHover, setIsHover] = useState(false);

 // widthを取得するために一時的にKonva.Textを使用
  const textWidth = useMemo(() => {
    const tempText = new Konva.Text({
      text: TEXT,
      fontSize: FONT_SIZE,
      padding: IMAGE_PADDING,
      lineHeight: LINE_HEIGHT,
      x: IMAGE_X + IMAGE_SIZE,
      letterSpacing: 1.2,
      y: 0,
      ellipsis: true,
    });
    const width = tempText.width();
    tempText.destroy();
    return Math.min(width, TEXT_MAX_WIDTH);
  }, []);

<Rect
  // ... 他のprops
  onMouseEnter={() => setIsHover(true)}
  onMouseMove={() => {
    setIsHover(true);
  }}
  onMouseLeave={() => setIsHover(false)}
/>
{isHover && stageRef.current && (
  <Group
    x={IMAGE_X}
    y={IMAGE_Y - (TAG_HEIGHT / scale.y) - (TAG_Y_PADDING / scale.y)}
    // ...
  >
    <Tag
      width={tagWidth}
      height={TAG_HEIGHT}
      fill="#4A46D4"
      cornerRadius={99}
    />
    <Text
      text={TEXT}
      fill={"white"}
      fontSize={FONT_SIZE}
      // ...
    />
  </Group>
)}
```

**実装のポイント：**

- `isHover`ステートでホバー状態を管理
- `onMouseEnter`、`onMouseMove`、`onMouseLeave`でホバー状態を切り替え
- 条件付きレンダリング`{isHover && ...}`でラベルの表示を制御
- ラベルは`Group`内の`Tag`（背景）と`Text`（テキスト）で構成
- ラベルの位置は`y={IMAGE_Y - (TAG_HEIGHT / scale.y) - (TAG_Y_PADDING / scale.y)}`でオブジェクトの上部に配置
- ラベルの幅を確定するため、一度Konva.Textを作りTextのサイズを計算し、適切な値をTagのwidthに設定している

### ズーム対応のラベルサイズ調整機能

ズームイン・ズームアウトによってラベルのサイズが調整される機能は、以下の実装で実現されています：

```tsx
const [scale, setScale] = useState({ x: 1, y: 1 });

// ズーム時のスケール更新
const onHandleScale = (e: Konva.KonvaEventObject<WheelEvent>) => {
  // ... ズーム処理
  const newScale = direction > 0 ? oldScale * scaleBy : oldScale / scaleBy;
  stage.scale({ x: newScale, y: newScale });
  setScale({ x: newScale, y: newScale });
}

<Group
  x={IMAGE_X}
  y={IMAGE_Y - (TAG_HEIGHT / scale.y) - (TAG_Y_PADDING / scale.y)}
  // スケールサイズごとにラベルのサイズを調整
  scaleX={1 / scale.x}
  scaleY={1 / scale.y}
>
```

**実装のポイント：**

- `scale`ステートで現在のズーム倍率を管理
- `onHandleScale`関数でホイールイベントからズーム処理を実行し、`scale`を更新
- ラベルの`Group`に`scaleX={1 / scale.x}`、`scaleY={1 / scale.y}`を設定
  - これにより、キャンバスがズームしてもラベルは常に一定サイズを保持
- ラベルのY座標も `Group`の`y={IMAGE_Y - (TAG_HEIGHT / scale.y) - (TAG_Y_PADDING / scale.y)}`でスケールに応じて調整され、正確な位置に表示

また、Groupのy座標もScaleが考慮されてない場合、以下のように拡大・縮小したときにラベルがずれてしまいます。
![](/images/CleanShot 2025-08-05 at 11.45.43.gif)


## まとめ

React-KonvaはCanvasの実装を宣言的に書けるので重宝しています。
使いこなすことによってはFigmaのような難易度が高そうなプロダクトも(見た目上は)作れるかもしれないのでもっと学んでいきたいです。