---
title: "Typstのテンプレートファイルを作業ディレクトリに移すのを、fishのオートロード関数で簡単に"
emoji: "🐟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typst, shell, fish]
published: true
---

# はじめに

[前に書いた記事](https://zenn.dev/htsulfuric/articles/typst_and_nvim)のとおり、現在TypstをTypst.vim+tinymistで編集しているが、Typst.vimがレポートのTypstファイルと同じディレクトリにある`template.typ`しか認識してくれないのでちょっと面倒だった。なので今回、今いるディレクトリから親ディレクトリを再帰的に探索して、テンプレートファイルをcpしてくるようなコマンドをfishのオートロード関数を使って実装した。

# やりたいこと

1.  現在のディレクトリから親ディレクトリを再帰的に探索して`template.typ`を見つける
2.  見つかった`template.typ`を現在のディレクトリにコピーする
3.  引数で指定したファイル名のTypstファイルを作成する
4.  あらかじめ設定したheaderをTypstファイルに追加する
5.  Typstファイルをエディタで開く

# 実装

以下のコードを`~/.config/fish/functions/typinit.fish`に保存する。

```shell:~/.config/fish/functions/typinit.fish
set header '#import "template.typ" : *

#show:report.with(
    title: "",
    author: "",
)
'

# 関数名とファイル名は同じにする
function typinit
    # 引数が指定されているか確認
    if test (count $argv) -ne 1
        echo "Usage: mkreptyp <filename.typ>"
        return 1
    end

    # ファイル名を取得し、正規化 (.typ をつける)
    set typname (string replace -r '(\.typ)?$' '' -- $argv[1]).typ

    # ファイルがすでに存在する場合はエラー
    if test -e $typname
        echo "Error: File '$typname' already exists."
        return 1
    end

    # 現在のディレクトリから再帰的に親ディレクトリを探索して template.typ を見つける
    set current_dir (pwd)
    while test "$current_dir" != /
        if test -e "$current_dir/template.typ"
            echo "Found 'template.typ' in '$current_dir'. Copying it to the current directory."
            cp "$current_dir/template.typ" ./template.typ
            break
        end
        set current_dir (dirname $current_dir)
    end

    # template.typ が見つからなかった場合はエラー
    if not test -e ./template.typ
        echo "Error: 'template.typ' not found in the current or any parent directory."
        return 1
    end

    # 新しい Typst ファイルを作成し、Header を書き込む
    touch $typname
    if test -n "$header"
        echo -e "$header" >$typname
    end

    # 作成したファイルを開く
    nvim $typname
end
```

# 使い方

ファイルを作りたいディレクトリに移動して、以下のコマンドを実行する。

```shell
typinit <filename.typ>
```

# コード詳解

- `set typname (string replace -r '(\.typ)?$' '' -- $argv[1]).typ`
  引数で指定されたファイル名を取得し、`.typ`をつける。-rオプションで正規表現を使っている。?は.typが0回か1回出現することを表し、$は行末を表す。そのため、ファイル名の末尾に.typがついていればそれを削除し、.typをつける。
- `set current_dir (dirname $current_dir)`
  現在のディレクトリから親ディレクトリを取得する。`dirname`は引数で指定されたファイルのディレクトリを返す。
