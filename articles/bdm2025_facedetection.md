---
title: "ESP32-S3を用いたメガネ装着型カメラモジュール"
emoji: "👓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ESP32", "顔認証", "Python"]
published: true
---

# はじめに

この記事は2024年に開催された電気情報機器学(BDM)の最終課題作成の備忘録です。また、2025年五月祭のEEIC企画で展示されていた同作品の詳細記事でもあります。

# やりたかったこと

**人の顔が覚えられない!** これは、人の顔を見て話せない私の問題でしかないのだが、せっかくなので私の人間的成長以外の手段で解決してみたくなった。
そこで、人の顔を覚える、認識する、という行為を外部に委託すればいいのではないか、というモチベーションで作成された。
最終的には、ESP32-S3を用いて、メガネに装着できるカメラモジュールを作成し、PC側で顔認証を行い、LINE Notifyで通知する、というものを作成した。

# 使用したもの

## XIAO ESP32-S3 Sense

装着するにあたって小型軽量のマイコンを使用する必要があったため、ESP32を選択した。XIAO ESP32-S3 SenseとESP32-CAMなどが標準でカメラ(OV2640CMOS等)を搭載するポートがついてるため、候補にあがった。スペック比較は海外の方がYouTubeに上げてくださっていたので、そちらを参照した。

https://youtu.be/gYQDGoVRObA?si=RuicvLGegs6WKI7H

結論として、どちらもESP32-S3を搭載しているが、XIAO ESP32-S3 Sence側には8MBのフラッシュが搭載されているのに対し、ESP32-CAM側は4MBのフラッシュしか搭載されていないため、XIAO ESP32-S3 Senceを選択した。ただし、XIAO ESP32-S3 Senceは小型である分発熱がかなり大きく、また付属のアンテナ以外で技適を取得しているか不明なため、その点を考慮しながら使用する必要があった。

:::message
以降簡略のため、XIAO ESP32-S3 SenseをESP32と呼ぶことにする。
:::

## MacBook Air(M1)

顔認証をして、それをLINE Notifyで通知したかったために、PC側でPythonを使用して顔認証を行うこととした。 顔認証は `face_recognition` というライブラリを使用した。これを使用することで、顔認証のための学習や、顔認識を行うことができる。

## 3Dプリンタ

顔に装着するためにESP32をメガネに装着しようとなり、3Dプリンタでメガネにひっかける籠のようなものを作成した。3Dプリンタは、学内の3Dプリンタを使用した。

## その他

- Gemini

最近無料になっていたのでコードのリファクタリングで使ってみた(BDMはデスマーチだったためかなり汚いコードになっていた)。

# コード

https://github.com/HTsulfuric/face_recognition

`firmware/`以下にESP32のコードが、その他はPython用のコードが書かれている。 以下それぞれのコードの軽い説明を行う。

## ESP32側のコード

基本的にはESP32のお試しライブラリのなかにあるCameraWebServerを参考にする。

https://github.com/RuiSantosdotme/arduino-esp32-CameraWebServer

変更点としては、

- XIAO ESP32 以外のボード用のコードを削除(Face_detection.ino, camera_pins.h)
- HTTPではなくWebSocketをもちいることで双方向での通信が容易になった(app_https.cpp)

などがある。

WebSocketの関数などをArudiono IDEのlibraryでAsync WebServerから持ってくることができたため、ESP32側のコードはあまり難しくなかった。

## Python側のコード

GUI(Tkinter)、Logger、Websocket、Face_recognitionの4つを行なうモジュールを作成した。

# 大変だった点

## ESP32側

## Python側

# 課題点

## ESP32側

- 発熱の問題

5min程度の稼動でもESP32が熱くなりすぎてWiFIが切断されることがあった。また、httpサーバーの起動はそのままに、カメラを休止させようとしたがそれに必要なピン(PWDN端子)がGPIOに割り当てられていなかったため、カメラを休止させることができなかった。

詳しくは以下参照
https://forum.seeedstudio.com/t/xiao-esp32s3-sense-camera-sleep-current/271258/5?page=4

- アンテナの問題

Xiao ESP32 Senceは技適の都合で付属のアンテナ以外で使用することができないため、ESP32-CAMを使用した場合と比べて通信距離が短くなってしまった。これにより、PC側からの通信が途切れやすくなってしまった。

:::details 現実的でない解決方法
電波法に基づくと、電波暗室内においては技適のないアンテナを使用しても問題ない。実際、筆者が電波暗室においてアンテナに鉄でできたクリップをつけて使用したところ、通信が大幅に改善された。しかし、電波暗室内で相手の顔がわからなくなるケースがあるとは思えないため、実用的ではない。
:::

## Python側

- デザインの問題
  Tkinterダサい。
