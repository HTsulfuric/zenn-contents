---
title: "研究室のGPU 1枚でDeepSeek-V4.1-Flashを4並列ホストしている話（Project Maya）"
emoji: "🧵"
type: "tech"
topics:
  - "llm"
  - "gpu"
  - "ローカルllm"
# publication_name: "kadomoto_lab"
published: false
---

## tl;dr

- 研では [Project Maya](https://github.com/mw00/project-maya) を使って、DeepSeek-V4.1-Flash-1017 を GPU 1枚・4並列でホストしています
- 同じ GPU で SGLang + KTransformers も試しましたが、Maya のほうが1本で約10倍、4本同時で約6倍速かったので Maya にしました
- この記事は、その経緯と構成、設定値、その値にした理由のまとめです。立て直すときや、自分の GPU で同じことをするときの参考にしてください
- 途中で fair slice に「待ちが1本なのに、走っている4本が全部切られる」バグを見つけたので、上流で直しておきました（v1.1.59 / #883）

## はじめに

研究室で鯖管をしています。その仕事のひとつが、GPU を1枚占有して LLM をホストすることです。研のメンバーは OpenAI 互換 API として、各自のクライアントやコーディングエージェントから使っています。

構成と運用の細かいところはサーバ上の README にまとめてあるのですが、「なぜこうなっているか」は散らばりがちです。次に立て直す人（たぶん未来の自分）のために、ここに整理しておきます。

## Maya を選ぶまで

### 最初は Qwen3.8-Flash-Next でした

もともとは llama.cpp で Qwen3.8-27B を動かしていて、10 月頭に流行りの Qwen3.8-Flash-Next（125B の MoE、NVIDIA 公式の NVFP4）を vLLM で動かす構成に切り替えました。速度は文句なしで、1本だけなら日本語で 114〜126 tok/s、4本同時の合計は 298 tok/s 出ていました。

ただ、実際に使うと、thinking を low にしても長々と考え込んだり、回答の中で言語が混ざったりして、普段使いにはつらいものでした。速さを多少犠牲にしても、もう少し賢いモデルを載せたい、というのが乗り換えの動機です。

### SGLang + KTransformers では実用になりませんでした

次に試したのが DeepSeek-V4.1-Flash-1017 です。748B なので GPU 1枚には当然載らず、エキスパートを CPU 側に逃がす必要があります。定番の組み合わせは [SGLang](https://github.com/sgl-project/sglang) + [KTransformers](https://github.com/kvcache-ai/ktransformers) なので、まずはこれで unsloth の UD-Q2_K_XL を動かしました。

VRAM に載せるエキスパート数（層ごと 384 個中）を変えながら、あとで Maya を測ったのと同じベンチ（入力約 6,000 文字、出力 512 トークン、thinking なし）を流した結果です。

| 構成 | 1本 (tok/s) | 4本の合計 (tok/s) | 最初のトークン (1本 / 4本) |
| --- | ---: | ---: | --- |
| SGLang + KT、VRAM 0 / 384 | 4.7 | 5.6 | 45 s / 130 s |
| SGLang + KT、VRAM 64 / 384 | 5.2 | 7.5 | 35 s / 102 s |
| SGLang + KT、VRAM 128 / 384 | 5.8 | 10.7 | 27 s / 78 s |
| SGLang + KT、VRAM 160 / 384、RAM キャッシュ 160 GiB | 6.4 | 15.7 | 17 s / 45 s |
| **Maya v1.1.36、Maya-S** | **67.3** | **96.0** | |

SGLang 側は、高速化のオプション（CUDA Graph、DSpark、KV の貸し出しなど）を切った互換モードで、コンテキストも 4K です。デコーダの CUDA Graph を有効にした試行は、最初の要求でサーバが落ちました。なので、これが SGLang + KTransformers の実力の上限というわけではありません。それでも、同じ GPU と同じベンチで1本なら約10倍、4本同時でも約6倍の差があり、最初のトークンに数十秒かかるのでは、研で共用するには厳しいと判断しました。

Maya は同じモデルを、262,144 トークンのコンテキスト・4並列のままこの速度で動かせたので、そのまま本番にしました。約 262K トークンの入力を 4 本同時に流しても、落ちずに答えが返ることも確かめてあります。

## Project Maya について

[Project Maya](https://github.com/mw00/project-maya) は、GLM-5.3（744B）や DeepSeek-V4.1-Flash-1017（748B）のような、普通ならデータセンターが要るサイズの MoE モデルを、手元の GPU で動かすためのオープンソースのプロジェクトです。作者は Peasant Smith（[@mw00](https://github.com/mw00)）です。

特徴は、エキスパートをメモリ階層に分けて置くところです。よく使われるエキスパートは VRAM に、残りは RAM（CPU で計算）や SSD に置き、会話に合わせて入れ替えます。起動時に GPU・CPU・RAM・SSD を測って自分でチューニングもしてくれます。OpenAI 互換 / Anthropic 互換の API、ブラウザのチャット、ライブモニター、モデルのダウンロードまで同梱されていて、ひととおり揃っています。

## 研の構成

| 項目 | 内容 |
| --- | --- |
| GPU | RTX PRO 6000 Blackwell Max-Q 1枚（97,887 MiB、電力上限 300 W） |
| CPU | 32 スレッドを CPU レーンに使用 |
| モデル | DeepSeek-V4.1-Flash-1017 の Maya-S（約 340 GB） |
| エキスパートの置き場所 | 15,744 個のうち約 7,200 個が VRAM、残りが RAM 上の CPU レーン |
| コンテキスト | 262,144 トークン / 会話 |
| 同時実行 | 4 会話（`--parallel 4`） |
| 速度の目安 | 1本だけなら 80〜110 tok/s、4本同時だと1本あたり 30〜50 tok/s |

Maya-S は Maya 独自の量子化で、元のモデルと同じ次トークンを選ぶ割合は 92.6% です。普通のチャットやコーディングの補助なら、速度・品質ともに十分です。

エンジンの起動引数はこうなっています。

```sh
maya-ds4 \
  --pack .../DeepSeek-V4.1-Flash-1017-GGUF/Maya-S/maya-ds4 \
  --max-context 262144 --tp 1 --threads on --dspark on \
  --parallel 4 --cpu-threads 32 --vram-experts 1.0 --reserve-mib 3584
```

- `--dspark on`: 投機的デコードです。起動時の計測で、有効だと 82.8 tok/s、無効だと 44.7 tok/s でした
- `--reserve-mib 3584`: VRAM の予約です（最初は 600 でした）。<!-- TODO: 増やした理由 -->
- `--parallel 4`: Maya は最大 8 並列まで扱えますが、GPU 1 枚なので 4 にしています

サーバは systemd の `llm.service` として常駐させていて、個人のログインセッションには依存しません。リリースは `releases/vX.Y.Z/` に1つずつビルドし、`current` のシンボリックリンクを付け替えて切り替えます。新しいリリースが出たらビルドまでを自動で済ませてメールで知らせ、切り替えは人が確認してから行います。

## 設定値と、その理由

Maya の設定ファイル（`maya-ds4.json`）の `env` で、会話の保存と fair slice を調整しています。

| 設定 | 初期値 | 現在 | 理由 |
| --- | --- | --- | --- |
| `MAYA_DS4_SLOT_KEEP` | 1 | 1 | 会話の推論状態を保存し、同じ会話の続きで再計算を減らす |
| `MAYA_DS4_SLOT_CAP` | 4 | 8 | 4レーン分に加えて、fair slice で待機中の会話も RAM に置けるようにする |
| `MAYA_DS4_SLOT_RAM_GB` | 2 | 4 | 8会話が全部満杯でも約 2.5 GB。それに復元点の分を足した保険 |
| `MAYA_DS4_SLOT_GB` | 4 | 16 | 4 GB ではディスク側が上限に張り付き（107会話・4.29 GB）、起動のたびに古い会話が消えていた |
| `MAYA_DS4_SLOT_DAYS` | 1 | 1 | 未使用の会話の保存期限（日） |
| `MAYA_FAIR_SLICE_S` | なし | 30 | 後述 |

保存されるのはチャット履歴ではなく、推論の途中状態です。この状態はかなり小さく、1位置あたり約 1.1〜1.2 KB でした。実測では 82,733 トークンの会話で 0.10 GB です。262,144 トークンの会話が満杯になっても約 0.3 GB なので、会話の上限数は気前よく取って大丈夫です。

:::message
v1.1.59 で、環境変数の接頭辞が `STRATA_` から `MAYA_` に変わりました。古い `STRATA_` の名前も `MAYA_` の名前として読まれるので、そのままでも動きます。
:::

### fair slice

コーディングエージェント経由で使われると、1回の依頼の裏でタイトル生成や会話要約のような短い要求がいくつも飛びます。4レーンが長い回答で埋まっていると、こういう短い要求はどれかの回答が終わるまで待たされます。ログで見ると、10分以上返ってこないこともありました。

fair slice は、回答を一定秒数書き続けたら、待っている要求にエンジンを譲る仕組みです。譲った回答は待ち行列の後ろに並び直し、順番が来たら続きから書きます。待っている要求がなければ何も起きません。

研では 30 秒にしています。4本同時で 1本あたり 37 tok/s 前後なので、約 1,100 トークン書くごとに1回譲る計算です。短い要求が待たされるのは、最悪でもおよそ 30 秒になります。10 秒も試しましたが、切り替えのたびに読み直すコスト（次の段落）が目立ったので 30 秒にしました。

ここで注意が必要なのは、DeepSeek 用のエンジン（ds4）が「状態を保ったまま一時停止する」機能（pause）を持たないことです。GLM のエンジンは pause を持っていて、状態を残したまま次の要求に譲れます。ds4 では、いったん STOP してレーンを譲り、再開時に「プロンプト＋ここまでの出力」を読み直します。会話の保存（`SLOT_KEEP`）が効いていれば、読み直しは差分の数百トークン、数秒で済みます。

効果を確かめるには、ログに次の2行が出ているかを見ます。

```
[maya] fair slice: 28798 tokens written, the engine goes to the next request (1 waiting); this one continues after it
[maya] sessions: took back a ... conversation ... ; N new tokens to read
```

`N new tokens to read` が大きい場合は、状態が再利用されていません。そのときは fair slice を外したほうが速いです。

## ついでに直したバグ: 待ち1本で4本全部が切られる

fair slice を入れた直後に本番ログを見たところ、妙なことが起きていました。

```
[maya] fair slice: 29696 tokens written, the engine goes to the next request (1 waiting); this one continues after it
[maya] fair slice: 28798 tokens written, the engine goes to the next request (1 waiting); this one continues after it
[maya] fair slice: 32337 tokens written, the engine goes to the next request (1 waiting); this one continues after it
```

待っているのは1本なのに、走っていた4本が全部切られています。1人が待っているだけなのに、4人分の読み直しが発生していました。

原因は `serve/server.py` のスライス判定です。

```python
# serve/server.py（v1.1.56、修正前）
if slice_s > 0 and not pausing and not emb and not self.stopping and len(produced) < max_new \
        and self.fifo.waiting() and time.monotonic() - t0 >= slice_s:
```

`waiting()` は待ち行列の長さを返すだけです。この判定は4本の走者がそれぞれ独立に評価するので、待ちが1本でもいれば、30 秒を越えた走者は全員が譲ってしまいます。

そこで、「譲る本数」をロックの中で数えるように直しました（`claim_yield()`）。

```python
def claim_yield(self, slice_s: float) -> bool:
    now = time.monotonic()
    with self._cv:
        need = sum(min(e[1], self.capacity) for e in self._queue
                   if not (len(e) > 3 and e[3]) or now - e[2] >= slice_s)
        if need - (self.capacity - self._used) - self._yields > 0:
            self._yields += 1
            return True
        return False
```

「待っている側が必要とする場所 − 空いている場所 − すでに譲ると決まった本数」が正のときだけ譲ります。これで、待ちが1本なら譲るのも1本になります。あわせて、場所を空けることと列に並ぶことを1ステップにまとめ（`yield_place()`）、結局切らずに済んだときはこの予約を取り消せる（`unclaim_yield()`）ようにしました。

### 実測

GPU 1枚、`--parallel 4`、v1.1.56 と修正版の比較です（違いは `serve/server.py` だけ）。長い回答4本を流している途中で短い要求（`What is 1+1?`）を1本投げ、スライスは効果を見やすくするため 10 秒にしました。各3回です。

| 項目 | v1.1.56 | 修正版 |
| --- | --- | --- |
| 短い要求の最初のトークン (秒) | 0.26 / 0.52 / 0.51 | 0.32 / 0.32 / 0.53 |
| 短い要求の完了 (秒) | 1.67 / 2.02 / 1.97 | 0.44 / 0.45 / 0.61 |
| 切られた本数 | 4 / 4 / 4 | 1 / 1 / 1 |
| 読み直し | 4本、534〜724トークンを1.4〜4.8秒ずつ | 1本、586〜725トークンを1.4〜1.6秒 |

短い要求の待ち時間はほぼ変わらず、犠牲になる回答が4本から1本に減りました。

既存のテスト（`serve/test_fair.py`）は容量1の FIFO でしか確かめていなかったので、このバグは表に出ませんでした。容量1なら、正しく動いていても譲るのは1本だけだからです。4レーンで「何本切られたか」を数えるテストも足しています。

修正は v1.1.59 で上流に取り込まれ、CHANGELOG に `thanks to @HTsulfuric, #883` と書いてもらいました。研の本番もすでに v1.1.59 以降で動いているので、今は気にしなくて大丈夫です。

## 立て直すときのメモ

- **バージョン**: v1.1.59 以降を使ってください。v1.1.58 以前で fair slice を有効にすると、上のバグを踏みます
- **校正**: エンジンは「VRAM に載らないエキスパートを CPU で計算するか、PCIe で流すか」を起動時に測り、結果を校正ファイルに保存します。この保存先のディレクトリにサービスユーザーが書き込めないと、保存に失敗しても何も表示されず、遅い配置のまま動き続けます。起動ログで VRAM に載ったエキスパートの数（約 7,200 個）を確認してください
- **`MAYA_DS4_LANE_PARITY=1`**: この構成（Maya-S、GPU 1枚、4レーン）では `std::length_error` で起動に失敗しました（v1.1.56）。使わないでください
- **fair slice の読み直し**: ds4 が pause を持つようになれば読み直しはなくなります。上流のロードマップ（#437）に載っています
- **1人の大量投入**: fair slice は待ち時間を公平にするだけで、流し続ける利用者を止めるわけではありません。必要になったら、per-IP の同時数制限や `max_tokens` の上限を別途入れます

## 付録: 環境

- GPU: RTX PRO 6000 Blackwell Max-Q 1枚（97,887 MiB）
- CUDA 13.1.1、ビルドは `-DCMAKE_CUDA_ARCHITECTURES=120 -DSTRATA_ENABLE_CUDA=ON -DSTRATA_NATIVE_EXPERTS=ON`
- パック: DeepSeek-V4.1-Flash-1017 / Maya-S
- 本番: `llm.service`（`/srv/llm/current` → `releases/v1.1.60`）
