---
title: "Hugo + GitHub Pages / Actionsでブログを公開する"
emoji: "🧐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [hugo, githubpages, githubactions]
published: true
---

Hugoでサイトを構築し、それをGitHub Pagesでホスティングし、ブログとして公開することにしました。
デプロイ作業は、GitHub Actionsを用いて自動化しています。

ここでは、Hugoのサイトを新規作成してから公開するまでの手順を記載しています。

## 事前準備

### HugoをMacにインストール

Homebrewを用いてHugoをインストールします。

```bash
▶ brew install hugo

▶ hugo version
Hugo Static Site Generator v0.79.1/extended darwin/amd64 BuildDate: unknown
```

## Huboでサイトを構築

公式のドキュメントに用意されている[Quick Start]( https://gohugo.io/getting-started/quick-start/ )に沿って、サイトの構築を進めます。

### Hugoサイトを新規作成

新しいHugoサイトを `hugo-blog` というフォルダに作成します。

```bash
▶ hugo new site hugo-blog
Congratulations! Your new Hugo site is created in /***/***/hugo-blog.

Just a few more steps and you're ready to go:

1. Download a theme into the same-named folder.
   Choose a theme from https://themes.gohugo.io/ or
   create your own with the "hugo new theme <THEMENAME>" command.
2. Perhaps you want to add some content. You can add single files
   with "hugo new <SECTIONNAME>/<FILENAME>.<FORMAT>".
3. Start the built-in live server via "hugo server".

Visit https://gohugo.io/ for quickstart guide and full documentation.
```

### テーマを適用

シンプルな[harbor]( https://github.com/matsuyoshi30/harbor/ )というテーマを今回は選びました。
サイトのthemesディレクトリにgitのsubmoduleとして追加します。

```bash
▶ cd hugo-blog
▶ git init
▶ git submodule add https://github.com/matsuyoshi30/harbor.git themes/harbor
Cloning into '/***/***/hugo-blog/themes/harbor'...
remote: Enumerating objects: 1329, done.
remote: Counting objects: 100% (136/136), done.
remote: Compressing objects: 100% (94/94), done.
remote: Total 1329 (delta 61), reused 86 (delta 39), pack-reused 1193
Receiving objects: 100% (1329/1329), 8.29 MiB | 9.70 MiB/s, done.
Resolving deltas: 100% (699/699), done.
```

### サイトを構築

テーマに用意されていたサンプルページをコピーして、サイトを構築します。

```bash
▶ cp -r themes/harbor/exampleSite/* .

▶ vim config.toml
- # REMOVE THIS
- themesDir = "themes/harbor"

# -Dのオプションを付けると、下書きとしてマークされたコンテンツも含みます
▶ hugo server -D
```

### リモートリポジトリを作成

構築したサイトをpushするリモートのリポジトリをGitHubに作成し、リモートリポジトリを設定する。

```bash
▶ git remote add origin git@github.com:bryutus/hugo-blog.git
```

## GitHub Actionsを用いて自動デプロイ

[Scheduling Hugo Builds on GitHub pages with GitHub Actions]( https://rmoff.net/2020/12/20/scheduling-hugo-builds-on-github-pages-with-github-actions/ )を参考に設定を行いました。

### SSHキーペアの作成と設定

デプロイ時に使用するSSHキーペアを作成します。

```bash
▶ ssh-keygen -t rsa -b 4096 -C "$(git config user.email)" -f gh-pages -N ""
```

GitHub Actionsを実行するリポジトリ（ `hugo-blog` ）のシークレットに秘密鍵（ `gh-pages` ）を登録します。
また、Actionsが静的サイトのコンテンツをプッシュする先のリポジトリ（ `bryutus.github.io` ）のデプロイキーに公開鍵（ `gh-pages.pub` ）を登録します。
この時に、書き込みアクセスの許可にチェックを付けることを忘れないようにします。

### GitHub ActionsでのHugoのビルドとデプロイの設定

- 先程シークレットに登録したデプロイキーを `deploy_key` に指定します。
- デプロイ先のリモートリポジトリを `external_repository` に指定します。

```yml
name: github pages

on:
  push:
    branches:
      - main  # Set a branch to deploy

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          external_repository: bryutus/bryutus.github.io
          publish_branch: main
```

### デプロイを実行

hugo-blogリポジトリのリモートリポジトリに対してmainブランチをpushします。

```bash
▶ git push origin main
```

GitHubのActionsでデプロイのworkflowが成功していることを確認します。

## 公開されたサイトを確認

[サイト]( https://bryutus.github.io/ )が公開されていることを確認して終わりです。

## おわりに

GitHub Pagesでブログを公開しましたが、ここで書くかは考え中です。
というのも、ブログのコンテンツよりも仕組みやデザイン等といったコンテンツ以外のことが気になり、手段が目的となってしまいそうだからです。
おとなしくブログのために提供されているプラットフォームで書くことの方が私には合ってそうです。
