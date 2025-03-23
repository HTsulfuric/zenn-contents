---
title: "NeovimでTypstを書くための環境準備"
emoji: "🥳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Typst, Neovim]
published: true
---

# はじめに

最近個人的に気にいっている組版であるTypstであったが、`tinymist`を入れてNeovim上で編集していると`callback.lua`(だったはず)によるアラートメッセージの嵐になってしまい、まともに作業できないという現象に見舞われていた。どうも日本語入力が邪魔しているようだったが、流石に日本語が打てない状態でレポートを書きたい訳もなく、泣く泣くVScodeを起動していた。

そんな時、友人からNeovimの方で更新があり問題が解消した(以下のpull request参考)との噂を聞いたので、Neovim v0.10.3を試してみよう、というのがこの記事である。

https://github.com/neovim/neovim/pull/30747

[2025/03/17追記]
nvim v0.10.4がstableになったので、この記事の **Neovim nightly** の項目は不要になりました。

# Neovim nightlyの準備

## Neovimのアンインストール

Neovim v0.10.3はこの記事を執筆時点(2024/11/11)ではnightly(pre-releaseぐらいのニュアンス?)なので、一度手元のneovimをアンインストールしてあげる必要がある。筆者はHomebrewでダウンロードしていたので、

```shell
$ brew uninstall neovim
```

## NeovimをGitHubからダウンロード

ここを参考にするとよい。
https://github.com/neovim/neovim/blob/release-0.10/BUILD.md

0. ビルドに必要なものをダウンロード
   筆者はmacOSなので`xcode`と`ninja` `cmake` `gettext` `curl`を入れる

```shell
$ xcode-select --install
$ brew install ninja cmake gettext curl
```

1. NeovimをGitHubからダウンロード

```shell
$ git clone git@github.com:neovim/neovim.git
```

2. ディレクトリを移動し、`CMAKE`

まずは作業ディレクトリに入る

```shell
$ cd neovim
```

今回はNeovim v0.10.3が目的なので、0.10のブランチへとチェックアウトする(執筆時点のv0.11.0は破壊的変更が沢山で大変だった)

```shell
$ git checkout release-0.10
```

ブランチを切り替えたら

```shell
$ make CMAKE_BUILD_TYPE=RelWithDebInfo
```

でビルドが開始する。

3.  インストール

```shell
$ sudo make install
```

で `/usr/local/bin`にインストールされる。もしパスが通ってなければ通し、バージョン確認。

```shell
$ vim --version
NVIM v0.10.3-dev-25+gf8ee92feec
Build type: RelWithDebInfo
LuaJIT 2.1.1713484068
Run "nvim -V1 -v" for more info
```

# Typst.vim, tinymistの準備

筆者は`Lazy.nvim`でプラグイン管理しているため、以下のようなluaファイルを用意して`lua/plugins`以下に置く。

```lua:~/.config/nvim/lua/plugins/typst.lua
return {
  {
    "kaarmu/typst.vim",
    ft = "typst",
    lazy = false,
    config = function() vim.g.typst_pdf_viewer = "skim" end, -- ここは好きなpdf viewerを入れるとよい
  },
}
```

また、LSPは`Mason`で管理しているため。一度Neovimを起動して、`:MasonInstall tinymist`としてあげる。

# 結果

無事環境設定に成功。`:TypstWatch`から`skim`を起動をして、Typstの魅力であるリアルタイムなコンパイルもskimの自動更新によって見られる。ようやくNvimでTypstが書けて嬉しい。

# 余談

先日ターミナル上でPDFを見ることのできるプログラム(しかも自動更新!)をみつけたので、これを組みあわせればこんな作業環境にもできる。

![](https://storage.googleapis.com/zenn-user-upload/cd34022acd25-20241111.png)
_おしゃれ!_

https://github.com/itsjunetime/tdf
