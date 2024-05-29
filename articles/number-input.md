---
title: "数値形式のinputをコンマ表示付きで作成する" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["git"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

VSCode でファイルのコミット・プッシュを行っていると次のようなエラーが発生してプッシュが出来なくなった。

```
This repository is configured for Git LFS but 'git-lfs' was not found on your path. If you no longer wish to use Git LFS, remove this hook by deleting the 'pre-push' file in the hooks directory (set by 'core.hookspath'; usually '.git/hooks').
```

この問題が発生しているのは特定のリポジトリのみ。また、VSCode 以外のターミナルからは問題なくプッシュが出来ている。
一旦 VSCode を再起動しても解決しなかった。

git-lfs がインストールされていないというエラーなので、git-lfs があるか確認してみる。

```
git-lfs
```

全然あるな...。brew でインストールしてある。なぜか VSCode では見つからない。
