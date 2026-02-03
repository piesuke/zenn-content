---
title: "Git Worktreeの使いづらいポイントを改善した" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["git"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
publication_name: "uzu_tech"
---

こんにちは。[株式会社 Sally](https://sally-inc.jp/) エンジニアの [@piesuke](https://x.com/piesuke27)です。

手前味噌ですが、最近弊社のプロダクション事業部で体験型のイベントを開催しています！私も一部テストプレイに参加しましたがとても面白かったので興味のある方はぜひ！

https://sh1-ken.uzu-app.com/

弊社ではマーダーミステリーを遊べるアプリ「ウズ」と、マーダーミステリーを制作してウズ上で遊べるアプリ「ウズスタジオ」を開発しています。
絶賛エンジニア募集中です！詳しくは [こちら](https://sally-inc.super.site/) をご覧ください。

## 背景

Claude Codeを使って並行で様々なタスクをこなす際、Git Worktree を使うことが増えました。

基本的に便利ですが、　使い続けていくうちにいくつかの課題が見えてきました。

### 課題1: `git worktree add` を毎回手打ちするのが面倒

Worktree を追加する際、毎回以下のようなコマンドを手打ちする必要があります。

```bash
git worktree add ../worktrees/feature-login feature/login
```

パスの指定やブランチ名の入力が必要で、特にブランチ名にスラッシュが含まれる場合はディレクトリ名を別途考える必要があり、地味に手間がかかります。

### 課題2: 毎回環境構築を手動で行うのが面倒

Worktree を作成した後、`npm install` や `pnpm install` などの依存関係のインストールを毎回手動で行う必要があります。これを忘れてビルドエラーになることもしばしば...。

### 課題3: ワークツリーを起点として別のワークツリーを作成してしまう

Worktree のディレクトリで作業していると、そこを起点として別の Worktree を作成してしまうことがあります。本来はメインのリポジトリから作成したいのに、意図せず入れ子構造になってしまい混乱の原因になります。

## 解決策

これらの課題を解決するために、Worktree の作成から環境構築までを一発で行うシェルスクリプトを作成しました。

### 使い方

エイリアスとして登録しておくことで、以下のようなシンプルなコマンドで Worktree の作成と環境構築が完了します。

```bash
add-wt frontend feature/login
```

これだけで以下の処理が自動で行われます：

1. メインリポジトリに移動（どこにいても大丈夫）
2. `git fetch --all` で最新情報を取得
3. ブランチの存在確認（ローカル/リモート/新規作成を自動判定）
4. Worktree の作成
5. プロジェクトに応じた環境構築コマンドの実行

### スクリプトの実装

まず、設定ファイル `worktree-config.sh` でプロジェクトごとの設定を定義します。

```bash
#!/bin/zsh

# Git Worktree 共通設定ファイル
# worktree.sh と remove-worktree.sh から読み込まれる

typeset -A REPO_PATHS
typeset -A WORKTREE_DIRS
typeset -A EXEC_PATHS
typeset -A SETUP_COMMANDS

# 例: frontend プロジェクト
REPO_PATHS[frontend]="$HOME/repos/frontend-main"
WORKTREE_DIRS[frontend]="$HOME/worktrees/frontend"
SETUP_COMMANDS[frontend]="npm install"

# 例: backend プロジェクト
REPO_PATHS[backend]="$HOME/repos/backend-main"
WORKTREE_DIRS[backend]="$HOME/worktrees/backend"
SETUP_COMMANDS[backend]="bundle install"

# 例: monorepo プロジェクト（サブディレクトリで実行する場合）
REPO_PATHS[monorepo]="$HOME/repos/monorepo-main"
WORKTREE_DIRS[monorepo]="$HOME/worktrees/monorepo"
EXEC_PATHS[monorepo]="packages/web"  # worktree内のサブディレクトリ
SETUP_COMMANDS[monorepo]="pnpm install"

# ここに必要なプロジェクトを追加
# REPO_PATHS[your_project]="リポジトリのパス"
# WORKTREE_DIRS[your_project]="worktree作成先のディレクトリ"
# EXEC_PATHS[your_project]="worktree内のサブディレクトリ（オプション）"
# SETUP_COMMANDS[your_project]="セットアップコマンド（オプション）"
```

次に、メインのスクリプト `worktree.sh` です。

```bash
#!/bin/zsh

set -e

# スクリプトのディレクトリを取得
SCRIPT_DIR="${0:A:h}"

# 共通設定を読み込み
source "$SCRIPT_DIR/worktree-config.sh"

# デフォルトのメインブランチ
MAIN_BRANCH="main"

# === 関数 ===
usage() {
    echo "Usage: $0 <project_type> <branch_name>"
    echo ""
    echo "Available project types:"
    for key in ${(k)REPO_PATHS}; do
        echo "  - $key"
    done
    echo ""
    echo "Examples:"
    echo "  $0 frontend feature/login"
    echo "  $0 backend fix/api-bug"
    exit 1
}

# === メイン処理 ===

# 引数チェック
if [[ $# -lt 2 ]]; then
    echo "Error: 引数が不足しています"
    usage
fi

PROJECT_TYPE="$1"
BRANCH_NAME="$2"

# プロジェクトタイプの存在確認
if [[ -z "${REPO_PATHS[$PROJECT_TYPE]}" ]]; then
    echo "Error: 不明なプロジェクトタイプ: $PROJECT_TYPE"
    usage
fi

REPO_PATH="${REPO_PATHS[$PROJECT_TYPE]}"
WORKTREE_BASE="${WORKTREE_DIRS[$PROJECT_TYPE]}"
EXEC_SUBDIR="${EXEC_PATHS[$PROJECT_TYPE]}"

# リポジトリの存在確認
if [[ ! -d "$REPO_PATH" ]]; then
    echo "Error: リポジトリが見つかりません: $REPO_PATH"
    exit 1
fi

# worktreeディレクトリの作成（存在しない場合）
if [[ ! -d "$WORKTREE_BASE" ]]; then
    echo "Creating worktree directory: $WORKTREE_BASE"
    mkdir -p "$WORKTREE_BASE"
fi

# ブランチ名からworktreeのディレクトリ名を生成（スラッシュをハイフンに変換）
WORKTREE_NAME="${BRANCH_NAME//\//-}"
WORKTREE_PATH="$WORKTREE_BASE/$WORKTREE_NAME"

# worktreeが既に存在する場合はスキップ
if [[ -d "$WORKTREE_PATH" ]]; then
    echo "Worktree already exists: $WORKTREE_PATH"
    echo "移動: cd $WORKTREE_PATH"
    exit 0
fi

# メインリポジトリに移動
cd "$REPO_PATH"

# 最新の情報を取得
echo "Fetching latest from remote..."
git fetch

# ブランチが存在するか確認
if git show-ref --verify --quiet "refs/heads/$BRANCH_NAME"; then
    # ローカルブランチが存在する場合
    echo "Creating worktree from existing local branch: $BRANCH_NAME"
    git worktree add "$WORKTREE_PATH" "$BRANCH_NAME"
    # upstreamが未設定の場合は設定
    cd "$WORKTREE_PATH"
    if ! git rev-parse --abbrev-ref --symbolic-full-name "@{u}" &>/dev/null; then
        git push -u origin "$BRANCH_NAME" 2>/dev/null || true
    fi
    cd "$REPO_PATH"
elif git show-ref --verify --quiet "refs/remotes/origin/$BRANCH_NAME"; then
    # リモートブランチが存在する場合
    echo "Creating worktree from remote branch: origin/$BRANCH_NAME"
    git worktree add "$WORKTREE_PATH" -b "$BRANCH_NAME" "origin/$BRANCH_NAME"
else
    # 新規ブランチを作成（origin/mainをベースに）
    echo "Creating worktree with new branch: $BRANCH_NAME (based on origin/$MAIN_BRANCH)"
    git worktree add -b "$BRANCH_NAME" "$WORKTREE_PATH" "origin/$MAIN_BRANCH"
    # 新規ブランチのupstreamを設定
    cd "$WORKTREE_PATH"
    git push -u origin "$BRANCH_NAME" --no-verify 2>/dev/null || true
    cd "$REPO_PATH"
fi

echo ""
echo "Worktree created successfully!"
echo "  Path: $WORKTREE_PATH"
echo "  Branch: $BRANCH_NAME"

# セットアップコマンドが定義されている場合は実行
if [[ -n "${SETUP_COMMANDS[$PROJECT_TYPE]}" ]]; then
    # worktree内のサブディレクトリでセットアップを実行
    if [[ -n "$EXEC_SUBDIR" ]]; then
        EXEC_PATH="$WORKTREE_PATH/$EXEC_SUBDIR"
    else
        EXEC_PATH="$WORKTREE_PATH"
    fi
    echo ""
    echo "Running setup command: ${SETUP_COMMANDS[$PROJECT_TYPE]}"
    echo "  in: $EXEC_PATH"
    echo "----------------------------------------"
    cd "$EXEC_PATH"
    eval "${SETUP_COMMANDS[$PROJECT_TYPE]}"
    echo "----------------------------------------"
    echo "Setup completed!"
fi

# 移動先パスの表示
if [[ -n "$EXEC_SUBDIR" ]]; then
    echo ""
    echo "移動コマンド: cd $WORKTREE_PATH/$EXEC_SUBDIR"
else
    echo ""
    echo "移動コマンド: cd $WORKTREE_PATH"
fi
```

### エイリアスの登録

`.zshrc` に以下を追加してエイリアスとして登録します。

```bash
alias add-wt="/path/to/worktree.sh"
```

### ポイント

いくつかのポイントを解説します。

**常にメインリポジトリから作成**

スクリプト内で `cd "$REPO_PATH"` として必ずメインリポジトリに移動してから Worktree を作成するため、どのディレクトリにいても正しく Worktree を作成できます。

**ブランチの自動判定**

ブランチの状態を自動で判定し、適切な方法で Worktree を作成します。

- ローカルブランチが存在する → そのブランチをチェックアウト
- リモートブランチのみ存在する → ローカルブランチを作成してチェックアウト
- どちらも存在しない → 新規ブランチを作成

**スラッシュをハイフンに変換**

`feature/login` のようなブランチ名は `feature-login` というディレクトリ名に変換されます。これによりディレクトリ構造がシンプルになります。

**環境構築の自動化**

プロジェクトごとに `SETUP_COMMANDS` を定義しておくことで、Worktree 作成後に自動で依存関係のインストールなどが実行されます。

## まとめ

Git Worktree は便利な機能ですが、そのまま使うといくつかの煩わしい点があります。今回紹介したスクリプトを使うことで、Worktree の作成から環境構築までをワンコマンドで完了できるようになりました。

特に複数のプロジェクトを並行して開発している場合に重宝します。ぜひ自分の環境に合わせてカスタマイズして使ってみてください。