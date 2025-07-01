---
title: "貧弱ストレージに悩んでいる方々へ贈る、Macのストレージを空ける方法" # 記事のタイトル
emoji: "💥" # アイキャッチとして使われる絵文字（1文字だけ）
type: "idea" # tech: 技術記事 / idea: アイデア記事
topics: ["idea", "mac"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## これは何

お金をケチって貧弱なストレージの Mac を買ってしまったが故に日々ストレージが圧迫されている方々へ贈る、ストレージを空ける方法です。

## 免責事項

誤って必要なファイルを削除してしまう場合もあるので、実行は自己責任でお願いします。

## 【非エンジニア・エンジニア共通】不要なファイル、フォルダを削除する 

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

### ライブラリ内のキャッシュを削除する

Mac の各種アプリケーションのキャッシュは
`~/Library/Caches` フォルダに保存されています。放っておくと特にブラウザアプリなどはかなりキャッシュが溜まるので、定期的に消しましょう。

```bash
sudo rm -rf ~/Library/Caches
```

## 不要なアプリケーションを削除する
初期から入っているGarageBandやiMovieは、使わない人からしたらただの容量圧迫でしかないので速やかに消しましょう。
他にも、軽い気持ちでインストールしたけどよくよく考えると使ってないアプリケーションがあると思います。定期的に確認して削除しましょう。

## 【エンジニア用】各環境の不要なファイルを削除する

### npmのキャッシュを削除
npmのキャッシュフォルダからキャッシュデータを削除してくれます。

```bash
npm cache clean --force
```

### 不要なnode_modulesを一括削除

以下のコマンドで、node_modulesを持っているフォルダを一括検索します。
```bash
find . -name 'node_modules' -type d -prune 
```

あとは、しばらく使わないプロジェクトのnode_modulesを削除してあげるだけです。

### Flutter環境のキャッシュを削除

Flutterの各種キャッシュやビルド成果物を削除します。

```bash
# Flutterのキャッシュを削除
flutter clean

# pub-cacheを削除（全てのパッケージが再ダウンロードされるので注意）
rm -rf ~/.pub-cache

# 各プロジェクトのビルドフォルダを一括削除
find . -name "build" -type d -path "*/[flutterプロジェクトの名前]/*" -prune -exec rm -rf {} \;

# iOSビルドキャッシュを削除
rm -rf ~/Library/Developer/Xcode/DerivedData/*

# CoreSimulatorのキャッシュ削除
rm -rf ~/Library/Developer/CoreSimulator/Caches/dyld/

# 利用できないシミュレータを削除
xcrun simctl delete unavailable
```

### Go環境のキャッシュを削除

Goのモジュールキャッシュやビルドキャッシュを削除します。

```bash
# モジュールキャッシュを削除
go clean -modcache

# ビルドキャッシュを削除
go clean -cache

# テストキャッシュを削除
go clean -testcache
```

### Docker環境の不要なデータを削除

Dockerイメージやコンテナ、ボリュームなどを削除します。

```bash
# 停止中のコンテナを全て削除
docker container prune -f

# 使用されていないイメージを削除
docker image prune -a -f

# 使用されていないボリュームを削除
docker volume prune -f

# 使用されていないネットワークを削除
docker network prune -f

# 全ての不要なデータを一括削除（慎重に実行してください）
docker system prune -a --volumes -f
```

### その他の容量削減方法

```bash
# Homebrewのキャッシュを削除
brew cleanup -s
brew autoremove

# pipのキャッシュを削除
pip cache purge

# Gradleのキャッシュを削除（Android開発者向け）
rm -rf ~/.gradle/caches

# Cocoapodsのキャッシュを削除（iOS開発者向け）
pod cache clean --all

# ゴミ箱を空にする
rm -rf ~/.Trash/*

# システムログを削除（管理者権限が必要）
sudo rm -rf /var/log/*
sudo rm -rf /Library/Logs/*
```

## 結論
ケチらずにハイスペックの Mac を買いましょう。
