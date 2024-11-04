---
title: "ArchLinuxインストール奮闘録"
emoji: "🫠"
type: "tech"
topics:
  - "初心者"
  - "初心者向け"
  - "archlinux"
  - "インストール"
published: true
published_at: "2024-09-24 16:45"
---



## はじめに

久々にArchLinuxを起動したら色々壊れてしまっていたためクリーンインストールを決行したが、2回目なのに関らず幾度となく失敗した記録。犯したミスを含めて全体の流れを振り返っておこうと思う。

(情報は執筆時2024/09/24のものである)
### 作業環境

- CPU: AMD Ryzen 7 5700x
- GPU: NVIDIA Geforce RTX 3050

## インストールまで

### ISOイメージの準備

1. [ダウンロードページ](https://archlinux.org/download/) からISOイメージをダウンロード
:::message alert
**ミス1：遅いところからダウンロードした** 
Reflectorの結果をみると、執筆時期においてはcat.netが一番ダウンロードが早く終りそう。なんとなくjaistからにしたせいで結構時間がかかった。
:::

3. USBに焼く
手元にあったのがMacOSだったので[balenaEtcher](https://etcher.balena.io/)で焼いた。

### 起動・キーボード・時計等

1. USBを刺してbootメニューから起動
今回は2枚刺しのSSDの片方に導入する予定だったので、安全のためにもう片方のSSDを抜いておいた。
:::message alert
**ミス2：ネジがどっかにいった** 
SSDを固定してたネジがないなった。ヒートシンク的なもので上から抑えてるのでどうにかなっているけど......
:::

2. 起動モードの確認

とりあえずJISキーボードにする。
```shellsession
$ loadkeys jp106
```
そうしたら`efivars`ディレクトリの存在を確認。
```shellsession
$ ls /sys/firmware/efi/efivars
```

:::message alert
**ミス3：ここで起動モードの確認をサボった**
後のgrubのセットアップに失敗した。stackoverflowかどこかに

> ArchLinuxのFancyなロゴが出てたらUEFIで開けてるさ!HAHAHA!

みたいなことが書いてあったけど普通に**嘘**だった。このせいで1日潰したので許せない。
:::

3. ネットワーク、システムクロック設定

pingでネットワークの確認。
```shellsession
$ ping archlinux.org
```
今回は有線なのでこれ以上やることなし。

システムクロックを更新する
```shellsessioon
$ timedatectl set-ntp true
```

## パーティション割り、マウント
1. パーティション割り
   
まずは現状を把握するため
```shellsession
$ lsblk
```
今回はM2SSDにインストールするため`nvme0n1`を割っていく。以下のようにGPTで割った。

| Partition | Type | Size |
| --------- | ---- | ---- |
| nvme0n1p1 | EFI Partition | 512MiB |
| nvme0n1p2 | Linux Filesystem(root) | rest |
| nvme0n1p3 | swap Partition | 3GB |

`cgdisk` が視覚的に分かり易かった。
```shellsession
$ cgdisk /dev/nvme0n1/
```
とりあえず不要なテーブルを削除した後、[パーティションタイプ](https://wiki.archlinux.jp/index.php/GPT_fdisk)を確認しながら、
- EFI パーティション： +512M確保し、ef00
- root: -3G(これで3G残して全部確保できる)、8300
- swap: +3G、8200

と割ってあげる。

2. フォーマット、マウント

EFIパーティションは`fat32`、rootは`ext4`、swapは`mkswap`でフォーマットしてあげる。

```shellsession
$ mkfs.fat -F 32 /dev/nvme0n1p1
$ mkfs.ext4 /dev/nvme0n1p2
$ mkswap /dev/nvme0n1p3
```

次にrootを`/mnt`に、EFIパーティションを`/mnt/boot/`にマウントする。

```shellsession
$ mount /dev/nvme0n1p2 /mnt
$ mount --mkdir /dev/nvme0n1p1 /mnt
```

swapは`swapon`で有効化する。
```shellsession
$ swapon /mnt/nvme0n1p3
```

## インストール
ダウンロードを早くすませるために`/etc/pacman.d/mirrorlist/`のミラーリストを更新してあげる。[先駆者さま](https://zenn.dev/ytjvdcm/articles/0efb9112468de3)のありがたいコマンドをお借りして、
```shellsession
$ cat /etc/pacman.d/mirrorlist | cat <(curl -s "https://archlinux.org/mirrorlist/?country=JP" | sed -e 's/^#Server/Server/') - > /etc/pacman.d/mirrorlist
```
をすると更新してくれる。vimで`/etc/pacman.d/mirrorlist/`を覗いて、コメントアウトされてないかを確認する。

ここで、色々パッケージをインストールする。 今回は

- base
- base-devel
- linux
- linux-firmware
- linux-headers
- vim

をとりあえずいれた。
```shellsession
$ pacstarp /mnt base base-devel linux linux-firmware linux-headers vim
```

## ch-rootで色々
### fstab,chroot
1. ここでfstabを生成しておく。
```shellsession
$ genfstab -U /mnt >> /mnt/etc/fstab
```
2. `arch-chroot`し、rootに入る
```shellsession
$ arch-chroot
```

### タイムゾーン、ローカリゼーション
1. タイムゾーン設定

日本にすんでる人なら
```shellsession
$ ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
```
BIOSをシステム時間とシンクロさせる
```shellsession
$ hwclock --synctohc
```

2. ローカライゼーション

`/etc/locale.gen`にある`en_US.UTF-8 UTF-8`と`ja_JP.UTF-8 UTF-8`をコメントアウト。その後

```shellsession
$ locale-gen
```

をすることでlocaleを作成できる。そうしたら、LANG変数とkeymappingを設定する。

```shellsession
$ echo LANG=en_US.UTF-8 > /etc/locale.conf
$ echo KEYMAP=jp106 > /etc/vconsole.conf
```

### ネットワーク設定
1. ホスト名の設定

まずは`/etc/hostname/`に好きなホスト名(今回はmyhostnameとする)を書きこみ、`/etc/hosts`を編集する。
```shellsession
$ echo myhostname > /etc/hostname
$ vim /etc/hosts
```
`hosts`は下のように記述する。
```clike:/etc/hosts
127.0.0.1    localhost
::1    localhost
127.0.1.1    myhostname.localdomain myhostname
```

2. networkmanagerの導入

`networkmanager`と周辺パッケージを色々いれる。
```shellsession
$ pacman -S networkmanager wpa_supplicant dialog iw iwd dhcpcd netctl
```
後は`networkmanager`を起動してあげるだけ。
```shellsession
$ systemctl enable NetworkManager
```

:::message alert
**ミス3：dhcpcdだけで頑張ろうとした**
ミスというよりかはskill issueだが、どこかに

> 有線接続ならdhcpcdだけのほうがいいよ

みたいなことを書いてある記事をみつけ、また先駆者様達のインストールガイドでは`dhcpcd`だけで**何故か**インターネット接続できているため、一日頑張ってしまった。しかし、うまくいかなかったため`networkmanager`を使ってみたら一発で接続できて微妙な気持ちになった。
:::

### GRUBの設定
1. まずは必要なものを色々インストール

```Shellsession
$ pacman -S grub efibootmgr dosfstools os-prober mtools
```
2. GRUBをEFIパーティションをマウントした場所にインストール

```Shellsession
$ grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB_UEFI
```
3.忘れないうちにconfigを作る

```Shellsession
$ grub-mkconfig -o /boot/grub/grub.cfg
```

### マイクロコード、rEFInd (Optional)
ここはやってもやらなくてもどうにかなる(rEFIndは便利だった)
1. マイクロコード適応
amdを使っているので`amd-ucode`をいれる
```Shellsession
$ pacman -S amd-ucode
```
これだけで勝手にやってくれる。

2.rEFIndのインストール
rEFInd:便利なbootローダー

```Shellsession
$ pacman -S refind
$ refind-install
```

### パスワード設定とユーザー追加
1. パスワード設定

```Shellsession
$ passwd
```
でrootにパスワードを設定する。

:::message alert
**ミス4:パスワードを設定しないで再起動**
パスワード無しで入れるとおもいました(一敗)。 結局一から入れなおしになったので本当に気をつけたい。
:::

2. ユーザー追加
いつまでもrootで作業していると危ないので通常ユーザーを作る(ユーザー名はuserとする)。

```Shellsession
$ useradd -m -g users -G wheel -s /bin/bash archuser
$ passwd archuser
```

3. `sudo`の導入

通常ユーザーでも管理者権限を使えるようにするために`sudo`を入れる。
```Shellsession
$ pacman -S sudo
```
としたあと、`/etc/sudoers`を編集して、wheelグループにsudoをあたえる。

```Clike:/etc/sudoers
## uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL:ALL) ALL
```


ここで`ch-root`でやることは終りなので`exit`で抜けたあと`reboot`する。

## GUI設定まで
### ログイン
1. rEFIndでbootを選ぶ
再起動すると、2種類のArchアイコンがみつかると思うので、`EFI/GRUB_UEFI/grubx64.efi`と書いてあるほうを起動する。
:::message alert
**ミス5:違うほうを起動する**
何もできなくなった。画面をちゃんと見て作業しよう。
:::
2. 設定したユーザー名とパスワードでログインする。
### ネットワーク設定とアップデート
1. NetworkManagerの設定
```Shellsession
$ nmtui
```
からNetworkManagerの設定画面が開ける。ここで使いたいネットワークを選択すれば接続できる。

2. パッケージのアップデート
```Shellsession
$ sudo pacman -Syu
```
で全てアップデートできる。

### ARUヘルパーのインストール
1. AURヘルパーとは？
[wiki](https://wiki.archlinux.jp/index.php/AUR_%E3%83%98%E3%83%AB%E3%83%91%E3%83%BC)曰く、AUR(Arch User Repository)からのパッケージのダウンロード、ビルドの自動化をしてくれるものらしい。

2. インストール準備
`yay`と`paru`の2巨頭があるらしいが、今回は`paru`をインストールしていく。`paru`はrust製なのでまずrustの準備をする。

```Shellsession
$ sudo pacman -S --needed git base-devel rustup
$ rustup default stable
```
3. インストール
gitからクローンしてくる

```Shellsession
$ git clone https://aur.archlinux.org/paru.git
$ cd paru
$ makepkg -si
```
### GUIのインストール(GNOME on Xorg)
1. Xorgのインストール
waylandにしてもよかったが、まだ問題が多い(？)らしいので今回は無難にX.orgを使用する
```Shellsession
pacman -S xorg-server xorg-server-utils xorg-xinit xterm
```

2. ドライバのインストール
まず自分のGPUを確認するために以下を実行。

```Shellsession
$ lspci -k | grep -A 2 -E VGA
```
今回はNVIDIAのTuring以降のモデルを使用したため`nvidia-open`をダウンロード([Wiki](https://wiki.archlinux.jp/index.php/NVIDIA#Xorg_.E8.A8.AD.E5.AE.9A)参考)。その後`xorg.conf`を自動設定するコマンドを使う。
```Shellsession
$ paru -S nvidia-open nvidia-xconfig
$ nvidia-xconfig
```
この後もう一度`lspci`をしてドライバが認識されているか確認する。
:::message alert
**ミス6：再起動をしなかった**
特にどこにも書いてなかったけど、手元環境では再起動をしないとドライバーが認識されなかった。
:::

3. GNOMEのインストール
とりあえず必要パッケージをダウンロード
```Shellsession
$ paru -S gnome gnome-tweak-tool
```
次に`~/.xinitrc`を以下のように編集する。
```clike:~/.xinitrc
export XDG_SESSION_TYPE=x11
export GDK_BACKEND=x11
exec gnome-session
```
そうしたら、`startx`でGNOMEに入れることを確認する。
```Shellsession
$ startx
```

## その他
### 日本語環境
1. 入力環境
今回はSKKにした。GNOMEだとibusがデフォルトで入っているようなのでインストールは不要だった。`/etc/environment`の環境変数を変更しておく。
```clike:/etc/environment
GTK_IM_MODULE=ibus
QT_IM_MODULE=ibus
XMODIFIERS=@im=ibus
```
ログインするときにibusを起動するため自動起動エントリを設定する。
```Shellsession
$ ibus-daemon -drxR
```
そうしたら`ibus-skk`と`skk-jisyo`をインストールし、設定>キーボードでskkを設定して完了。
```Shellsession
$ paru -S ibus-skk skk-jisyo
```
2. 日本語フォント
取り急ぎ`noto-fonts-cjk`だけ入れた。[Wiki](https://wiki.archlinux.jp/index.php/%E3%83%AD%E3%83%BC%E3%82%AB%E3%83%AA%E3%82%BC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3)を参照しつつ追加していくのもいいかもしれない。
```Shellsession
$ paru -S noto-fonts-cjk
```

### ターミナル周り
1. ターミナルの選択
標準のターミナルでも満足だが、MacOSで使えるiterm2のホットキーが好きなので、似たことのできるGuakeを入れた。
```Shellsession
$ paru -S guake
```
Guake Preferenceのアプリを開いて、ログイン時の自動起動設定とサイズ調整だけした。

2. シェルの選択 
個人的にfishシェルが好きなので変更する。
```Shellsession
$ paru -S fish
$ sudo chsh -s /usr/bin/fish
```

**以上!**

## 最後に
これで次インストールするときに困らない(と信じたい)。


## 参考資料
https://wiki.archlinux.jp/index.php/%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%82%AC%E3%82%A4%E3%83%89
https://qiita.com/Hayatann/items/09c2fee81fcb88d365c8
https://zenn.dev/ytjvdcm/articles/0efb9112468de3
https://qiita.com/k0kubun/items/e60fae90c688ac960ab7
https://wiki.archlinux.jp/index.php/GNOME#.E8.B5.B7.E5.8B.95
https://wiki.archlinux.jp/index.php/IBus
https://japan.zdnet.com/article/35208378/
https://zenn.dev/sawao/articles/0b40e80d151d6a