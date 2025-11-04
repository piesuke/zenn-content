---
title: "VSCodeのターミナルで補完を効かせて快適にコマンドを打とう" # 記事のタイトル
emoji: "👩‍💻" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["vscode", "terminal"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。
私たちは、マーダーミステリーを遊べるアプリ「ウズ」と、マーダーミステリーを制作してウズ上で配信できるアプリ「ウズスタジオ」を開発しています。
最近良かったマーダーミステリーは「[深い赤を愛する](https://mdms.jp/scenarios/7800)」です。

Claude Code を本格的に使い始めてから、VSCode のターミナルでプロンプトを調整したり、自動生成された差分を確認したりする機会が一気に増えました。これまでは Warp をメインターミナルにしていたので、Tab 補完前提で指が覚えているコマンドも多く、VSCode のターミナルだけ補完が効かないのは小さなストレスに感じていました。
しかし、実はVSCodeのターミナルでも補完機能が追加されていましたのでご紹介します。

## VSCode ターミナルで補完を有効化する

設定自体はとてもシンプルで、`terminal.integrated.suggest.enabled` を `true` にするだけです。`Cmd + ,` で設定を開き、検索ボックスに “suggest” と打てばすぐに該当設定が見つかります。コマンドパレットから「Preferences: Open Settings (JSON)」を開いて直接書き込んでも構いません。

![VSCode のターミナル補完設定](</images/CleanShot2025-11-04at10-45-11.png>)


## 補完を活かすワークフロー

普段よく使う `git` や `pnpm` のサブコマンドが自動的に学習され、複雑なサブコマンド入力も数文字で呼び出せるようになります。以下のスクリーンショットでは、`git` の履歴から branch 名まで含めて提案してくれています。

![git コマンドの補完例](</images/CleanShot2025-11-04at11-10-47.png>)

パッケージマネージャーで登録しているスクリプトも候補に並ぶので、`pnpm dev` や `pnpm lint` のようなルーティンコマンドを即座に実行できます。

![pnpm スクリプトの補完例](</images/CleanShot2025-11-04at11-09-35.png>)

嬉しいのは、`.zshrc` や `.config/fish/config.fish` で定義している自作エイリアスもしっかり候補に含まれる点です。よく打つ複雑な `ssh` コマンドやデプロイスクリプトをエイリアス化しておけば、VSCode のターミナルでも迷わず呼び出せるようになります。

![自作エイリアスの補完例](</images/CleanShot2025-11-04at_111109>)


