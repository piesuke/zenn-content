---
title: "貧弱ストレージに悩んでいる方々へ贈る、Macのストレージを空ける方法" # 記事のタイトル
emoji: "💥" # アイキャッチとして使われる絵文字（1文字だけ）
type: "idea" # tech: 技術記事 / idea: アイデア記事
topics: ["idea"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## これは何

お金をケチって貧弱なストレージの Mac を買ってしまったが故に日々ストレージが圧迫されている方々へ贈る、ストレージを空ける方法です。

## 免責事項

誤って必要なファイルを削除してしまう場合もあるので、実行は自己責任でお願いします。

## 不要なファイルを削除する

Downloads フォルダやデスクトップにある不要なファイルを削除しましょう。
特に、Downloads フォルダはアプリのインストーラーや PDF などが溜まっていることが多いので、定期的に確認して削除することをおすすめします。

```bash
# Downloads フォルダ内の不要なファイルを削除する
find ~/Downloads -type f -name "*.xip" -delete
find ~/Downloads -type f -name "*.dmg" -delete
find ~/Downloads -type f -name "*.pkg" -delete
find ~/Downloads -type f -name "*.zip" -delete

# 消していいファイルかどうか目視するときは echo で確認
find ~/Downloads -type f -name "*.xip" -exec echo {} \;
find ~/Downloads -type f -name "*.dmg" -exec echo {} \;
find ~/Downloads -type f -name "*.pkg" -exec echo {} \;
find ~/Downloads -type f -name "*.zip" -exec echo {} \;
```

デスクトップ内は私の場合スクリーンショットが溜まりがちなので、以下のコマンドで削除しています。

```bash
find ~/Desktop -type f -iname "*スクリーンショット*" -exec rm {} \;

# 消していいファイルかどうか目視するときは echo で確認
find ~/Desktop -type f -iname "*スクリーンショット*" -exec echo {} \;
```

「スクリーンショット」の部分を溜まりがちなファイル名に変更して実行してください。

## キャッシュを削除する

### ライブラリフォルダ内のキャッシュ

Mac の各種アプリケーションのキャッシュは
`~/Library/Caches` フォルダに保存されています。

```bash

```

## 結論

ケチらずにハイスペックの Mac を買いましょう。
