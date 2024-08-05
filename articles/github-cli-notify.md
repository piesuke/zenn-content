---
title: "Github Actionsの処理結果を見逃さないようにデスクトップ通知をする" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["flutter"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。

私たちは、マーダーミステリーを遊べることが出来るアプリ「ウズ」と、マーダーミステリーを制作してウズ上で遊べることが出来るアプリ「ウズスタジオ」を開発しています。
私が最近やって面白かったマーダーミステリーは [ヒガンノ饗宴](https://uzu-app.com/scenario/4366) です。

今回は、Github Actions の処理結果を見逃さないように通知をする方法を紹介します。

## 背景

コミットをプッシュした後、CI/CD の Github Actions が完了するのを待っている間他の作業をして、気づいたら CI/CD の完了を見逃してしまうこと、ありますよね。

私はコミットをプッシュしてプレビューを確認しようと CI/CD の完了を待っている間に、暇つぶしとして X を眺めて気づいたら 30 分経っていたことがあります。あな恐ろしや...

そうした悲劇が起こらないよう、Github Actions の処理が完了したら通知を飛ばして気づけるようにしました。

## 準備

使うのは Github CLI です。

```bash
    brew install gh
```

デスクトップ通知については、自分は Mac ユーザーなので terminal-notifier を使います。

```bash
    brew install terminal-notifier
```

Linux なら `notify-send` などが使えると思います。

## Github CLI の設定

Github CLI では様々な機能が提供されていますが、今回は Github Actions の実行状況を監視するために `gh run watch` を使います。

`gh run watch` は、コマンドを実行した時にそのリポジトリで走っている Github Actions のジョブの状況を監視することができるコマンドです。

これを通知と一緒に使うには、以下のようなコマンドを実行します。

```bash
    gh run watch -i1 | terminal-notifier -title "Github Actions" -message "Finished"
```

上記コマンドを実行すると、今走っているアクションの一覧が表示されるので、その中で完了通知を飛ばしたいアクションを選択します。

選択したアクションが完了したら、以下のような通知が飛びます。

![](/images/notify.png)

これで Github Actions の完了を見逃さないようになりますね！

## 応用

普段は、push と同時に `gh run watch`を走らせて、CI/CD の完了を待っている間に他の作業をしています。
こんな感じのエイリアスです。

```bash
alias gh-w="gh run watch -i1  && terminal-notifier -title "🍏" -message 'run is done!' -sound Crystal"

alias gp-w="git push && sleep 5 && gh-w"
```

push と同時に `gh run watch` を走らせると、まだ Github Actions が開始してない状況なので処理中のワークフローを取得することができません。なので sleep を入れています。

## まとめ

今回は、Github Actions の完了を見逃さないように通知を飛ばす方法を紹介しました。
特に CI/CD の完了を待っている間についつい X を眺めたり他の作業をしてしまう人に有効です！
