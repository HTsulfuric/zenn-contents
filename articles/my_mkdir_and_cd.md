---
title: "fishでmkdir+cdをひとつのコマンドに"
emoji: "💽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [fish, shell]
published: true
---

# はじめに

`mkdir`と`cd`、同時にやりたいよね……
bashやzshなんかだと毎回

```shell
sh hoge.sh
```

みたいなことをする必要がありそうで結構大変そう。一方fishにはオートロード関数といって、`~/.config/fish/functions/`以下に`.fish`の拡張子でfish functionを登録しておけば、簡単に書けてかつどこからでも呼びだせるらしい。とはいえ、ここまでは色んな人から過去記事があがっているため、エラー処理回りを少し丁寧にして実装してみた。

# 実装

関数名は過去の人々に倣って`mkcd`としてみた

```fish: ~/.config/fish/funtions/mkcd.fish
#function名とファイル名は一致している必要がある
function mkcd
    # 引数が1つ以外のときエラー
    if  test (count $argv) -ne 1
        echo "Usage: mkcd <directory>"
        return 1
    end

    # 同名ファイルが既に存在していればエラー
    if test -d $argv[1]
        echo "Directory '$argv[1]' already exists"
        return 1
    end

    # $statusは前の関数の返り値を保持する
    # mkdirが失敗していたらエラー
    mkdir -p $argv[1]
    if test $status -ne 0
        echo "Failed to create directory '$argv[1]'"
        return 1
    end

    cd $argv[1]
end
```
