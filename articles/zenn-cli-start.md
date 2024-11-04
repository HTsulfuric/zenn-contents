---
title: "Zenn cliの導入と、過去記事をGitHub連携した話"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Zenn", "GitHub"]
published: true
---

# はじめに

これまでZennのオンラインエディタを用いて記事を書いていたが、**Zenn cli**という便利そうなものがあることを知ったので(先に教えてくれよ)、そちらに環境を移行したときのメモ。
参照した記事が色々ばらついていたため、ここで纏めておく。

:::message alert

GitHub連携を行なうとオンラインエディタを使用することができなくなる(わぁ)。

:::

:::message

逆にスクラップはZenn cliで作成できないので、いままで通りオンラインエディタで作ることとなる

:::

# GitHub連携まで

https://zenn.dev/zenn/articles/connect-to-github

公式記事を参照する。

1. GitHubのレポジトリを作成する。

空のレポジトリを一つ準備する。PublicでもPrivateでもいいらしい。

2. GitHub連携を行う

[ここのリンク](https://zenn.dev/dashboard/deploys)から連携ページに飛べる(マイページ→GitHub連携でもよいはず)。
"リポジトリを連携する"を選択し、"Only select repositories"を選んで先程作ったレポジトリを登録する。

# Zenn cliの準備

https://zenn.dev/zenn/articles/install-zenn-cli

つぎはこの記事を参照。

0. Node.js 14以上のダウンロード

1. Zenn記事を保存しておきたいディレクトリに、先程のGitHubレポジトリをcloneする

```shell
$ git clone git@github.com:your-repository
```

2. クローンしたディレクトリへ移動し、zenn-cliのインストールをする

```shell
$ npm init --yes
$ npm install zenn-cli
```

3. Zenn cliのセットアップ

```shell
$ npx zenn init
```

これで以下のようなディレクトリが作成される。

```
your-repository
├── README.md
├── articles
├── books
├── node_modules
├── package-lock.json
└── package.json
```

# 過去記事のダウンロード

https://zenn.dev/zenn/articles/setup-zenn-github-with-export

ここを参照する。

1. 過去記事をエクスポート

[このリンク](https://zenn.dev/settings/export)から記事/本/スクラップをエクスポートできる。

2. .mdファイルをzenn用のディレクトリへと移動する

:::message
Q. エクスポートした記事をGitHubに投げても二重に投稿されない?

A. slugというもので記事が上書き対象か確認されます。

詳細は[公式記事](https://zenn.dev/zenn/articles/what-is-slug)の通りだが、簡単にいうとダウンロードしてきた.mdファイルの名前であるランダムな文字列によってファイルを識別してる。

:::

記事と本は以下のようにすればよい。

```shell
$ mv exported-zenn-files/articles/* your-repository/articles
$ mv exported-zenn-files/books/* your-repository/books
```

スクラップはファイルが自動で作成されていないため,自分で作ってあげる必要がある(デプロイはされないためあくまでローカルに保存するのみではある)。

```shell
$ mkdir your-repository/scraps
$ mv exported-zenn-files/scraps/* your-repository/scraps
```

あとはこの記事をGitHubにpushしてあげれば自動でデプロイしてくれる。

# Zenn cliの使い方

## 記事作成

https://zenn.dev/zenn/articles/zenn-cli-guide

ここを参照する。

1. 新規記事の作成

```shell
$ npx zenn new:article
```

でも作成できるが、スラグがランダムな文字列になってしまう。以下のようなオプションで指定できるため活用していきたい。個人的には、現在の作業環境だと絵文字が打ちずらいためここで指定してあげるとよさそう。

```shell
$ npx zenn new:article --slug your-slug --title your-title --type idea --emoji ✨
```

2. 記事編集

上のコマンドで対応するディレクトリ以下に.mdファイルが作成されているから、あとは好きなエディタで編集する。

## プレビュー

```shell
$ npx zenn preview
```

で`localhost:8000`でプレビューが立ち上がる。

## 画像のアップロード

[画像のアップロードページ](https://zenn.dev/dashboard/uploader)でリンクを取得できる

## 削除

記事削除はZennの[ダッシュボード](https://zenn.dev/dashboard)からしかできないため注意。

# 感想

好きなエディタでmdを書けるためとても便利。
僕にZennをお勧めした人へ、先にこっちのこと教えてくださいよ。

**以上!**
