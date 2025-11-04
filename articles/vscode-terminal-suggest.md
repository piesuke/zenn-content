---
title: "VSCode のターミナルを快適に使いたい" # 記事のタイトル
emoji: "👩‍💻" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["vscode", "terminal"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。
私たちは、マーダーミステリーを遊べるアプリ「ウズ」と、マーダーミステリーを制作してウズ上で配信できるアプリ「ウズスタジオ」を開発しています。
最近良かったマーダーミステリーは「[深い赤を愛する](https://mdms.jp/scenarios/7800)」です。

Claude Code CLI や Codex CLI を本格的に使い始めてから、ターミナルアプリではなく VSCode のターミナルを使う機会がぐっと増えました。ただ、既存のターミナルツールと比べると補完が弱く、細かいオプションを思い出せずに手が止まる場面もありました。そこで、VSCode のターミナルを快適に使うために行っている設定と活用法を整理します。

## 1. VSCode ターミナルで補完を有効化する
VSCode のターミナルを使い始めてまず困ったのが、補完がほとんど効かないことでした。Warp などでは補完を頼りにコマンドを覚えていたので、同じ感覚で作業すると細かいオプションを調べ直す羽目になります。ところが VSCode 1.98 でターミナル補完が大幅に改善され、今では十分実用的になりました。
[リリースノート](https://code.visualstudio.com/updates/v1_98#_terminal-intellisense-preview) でも詳しく紹介されています。

設定自体はシンプルで、`terminal.integrated.suggest.enabled` を `true` にするだけです。`Cmd + ,` で設定を開き、検索ボックスに "suggest" と入力すれば該当設定が見つかります。コマンドパレットから「Preferences: Open Settings (JSON)」を開いて直接書き込んでも構いません。

![VSCode のターミナル補完設定](</images/CleanShot2025-11-04at10-45-11.png>)

設定後は コマンドを打つと候補が表示され、`Tab` を打つと選択中の候補が確定されます。


### 補完機能でできること

例えば `git`と打ち、その後一文字入力すると `git`関連のサブコマンドが表示されるようになります。
![git コマンドの補完例](</images/CleanShot2025-11-04at11-10-47.png>)

パッケージマネージャーで登録しているスクリプトも候補に並びます。

![pnpm スクリプトの補完例](</images/CleanShot2025-11-04at11-09-35.png>)

嬉しいのは、`.zshrc` や `.config/fish/config.fish` で定義している自作エイリアスもしっかり候補に含まれる点です。よく打つ複雑な `ssh` コマンドやデプロイスクリプトをエイリアス化しておけば、VSCode のターミナルでも迷わず呼び出せるようになります。

![自作エイリアスの補完例](</images/CleanShot2025-11-04at_111109.png>)


## 2. Copilot Chat を呼び出す
VSCode のターミナルにフォーカスがある状態で `Cmd + I` を押すと、インラインの Copilot Chat を開けます。Copilot Chat にはターミナル関連のコンテキストを渡せるので、実行したコマンドをそのまま調査に活用できます。

- `#terminalLastCommand` … ターミナルで最後に実行したコマンド
- `#terminalSelection` … ターミナルでの選択範囲

コマンド実行後に怪しいログやエラーが出たときも、そのまま Copilot Chat に貼り付けて原因を分析してもらえるので、調査の初動が早くなります。

## まとめ
VSCode のターミナルを整えると、ツールを行き来することなくその場で試行錯誤できるようになります。今回紹介した補完設定や Copilot Chat を組み合わせて、日々の開発をさらに快適にしてみてください。
