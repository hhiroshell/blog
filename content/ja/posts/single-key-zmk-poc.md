---
title: "スイッチ1個・ハンダ付けなしでZMKワイヤレスキーボードのPoCを作る"
date: 2026-09-18
draft: false
tags: ["zmk", "keyboard"]
---

これはなに
---
現在愛用中の自作キーボードは、Pro Micro + QMK Firmwareという定番構成の有線キーボードなのですが、これの完全無線化バージョンを作りたいと思い立ちまして、手始めにXIAO nRF52840と[ZMK](https://zmk.dev/)に触ってみることにしました。

まずは最小構成で「ZMKの設定リポジトリ作成 → ビルド → 書き込み → Bluetooth接続」までの一連の流れを体験してみたので、この記事ではその内容を共有します。

用意したのはスイッチ1個だけです。ハンダ付けもブレッドボードも使わず、マイコンのパッドにICテストフックを挟むだけの配線にしました。PoCが終わったあとも、パーツを元の状態に戻しやすくするためです。


ハードウェア構成
---
- マイコン: [Seeed Studio XIAO nRF52840](https://wiki.seeedstudio.com/XIAO_BLE/)
- スイッチ: Cherry MX互換のキースイッチ1個
- 配線: XIAO本体のパッドにICテストフック（グラバー）を直接クランプし、スイッチの端子へ2本だけ配線
    - `D0`（`P0.02`） → スイッチの一方の端子
    - `GND` → スイッチのもう一方の端子
- 外付けのプルアップ抵抗はなし。GPIOの内部プルアップに依存
- 電源: USB-C給電（モバイルバッテリー）

配線はこんな感じです。

![XIAO nRF52840とキースイッチの配線](/images/single-key-zmk-poc-01.jpg)

ハンダ付けなしで気軽に試せるように、モバイルバッテリーからUSB-C給電しつつ動かしました。キーキャップには実際に送信するキーコードと同じ「A」を貼っています。

![モバイルバッテリー給電でのセットアップ全体](/images/single-key-zmk-poc-02.jpg)


ZMK設定リポジトリを作る
---
ZMKには公式の[ZMK CLI](https://github.com/zmkfirmware/zmk-cli)（`uv tool install zmk`→`zmk init`）が用意されていて、対話的にGitHubリポジトリの作成までできるようです。ただ今回は、生成される構成をきちんと理解したかったので、ZMK公式の[unified-zmk-config-template](https://github.com/zmkfirmware/unified-zmk-config-template)をベースに、必要なファイルを手で組み立てました。

最終的にできたリポジトリはこちらです。

- [hhiroshell/zmk-config](https://github.com/hhiroshell/zmk-config)

ディレクトリ構成は次のようになりました。

```
zmk-config/
├── .github/workflows/build.yml   # ZMK公式の再利用可能workflowを呼ぶだけ
├── boards/shields/single_key/    # 自作シールド本体
│   ├── Kconfig.shield
│   ├── Kconfig.defconfig
│   ├── single_key.overlay        # ハードウェア定義（kscan / GPIO）
│   ├── single_key.keymap         # キーコード
│   └── single_key.conf
├── build.yaml                    # ビルドマトリクス（board + shield の組み合わせ）
├── config/west.yml               # ZMK本体を取得するmanifest
└── zephyr/module.yml             # このリポジトリをboard/shieldの検索パスに登録
```

ポイントは`zephyr/module.yml`です。

```yaml
build:
  settings:
    board_root: .
```

これでリポジトリ直下がboard/shieldの検索パスとして登録され、`boards/shields/single_key/`に置いた自作シールドをZMKが見つけられるようになります。


kscan-gpio-directでスイッチ1個を認識させる
---
本題のハードウェア定義は`single_key.overlay`に書きます。

```dts
#include <dt-bindings/zmk/matrix_transform.h>

/ {
    chosen {
        zmk,kscan = &kscan0;
        zmk,matrix-transform = &default_transform;
    };

    kscan0: kscan_0 {
        compatible = "zmk,kscan-gpio-direct";
        wakeup-source;
        input-gpios = <&xiao_d 0 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
    };

    default_transform: keymap_transform_0 {
        compatible = "zmk,matrix-transform";
        columns = <1>;
        rows = <1>;
        map = <RC(0,0)>;
    };
};
```

いくつか設計判断のポイントがあります。

**`zmk,kscan-gpio-direct`を使う**
スイッチ1個だけなら、GPIOを直接読みに行く`zmk,kscan-gpio-direct`で十分で、今回はその設定を使います。実用的なキーボードのキー数に対応したい場合は、複数のスイッチをマトリクス配線で読み取る行列スキャン(`zmk,kscan-gpio-matrix`)を利用します。

**`&xiao_d 0`でD0ピンを指定する**
XIAO nRF52840は`seeed_xiao`という「インターコネクト」定義を公開していて、そのノードラベル`xiao_d`経由でピン番号を指定できます。[ZMKの設計ガイドライン](https://github.com/zmkfirmware/zmk/blob/main/app/boards/interconnects/seeed_xiao/seeed_xiao.zmk.yml#L16-L17)に「D0を参照するには`&xiao_d 0`を使う」と明記されており、`&gpio0 2`のように自分でP0.02を計算する必要はありません。

**`GPIO_ACTIVE_LOW | GPIO_PULL_UP`**
外付けプルアップ抵抗を使わず、スイッチはGND側でONになる配線なので、「内部プルアップ有効化」＋「Lowアクティブ」の組み合わせにします。

**matrix-transformは1x1でも一応残した**
スイッチ1個なら`zmk,kscan`だけをchosenに指定して`matrix-transform`を省略しても問題なくビルドは通ります。ただ、今後スイッチを増やしていくときのひな形として、他の複数キーシールドと同じ構造に揃えておきたかったので、あえて1x1の`matrix-transform`を残しています。


つまずいたポイント：ボード名がドキュメントと違う
---
ZMKの公式ドキュメントやGitHubリポジトリの`main`ブランチを見ながら作業していたところ、XIAO nRF52840のボード名は`xiao_ble/nrf52840/zmk`と表記されていました。調べてみると、ZMKは[2025年12月のZephyr 4.1移行](https://zmk.dev/blog/2025/12/09/zephyr-4-1)にあわせてZephyrの「Hardware Model v2」という新しいボード定義の仕組みを取り入れており、XIAO nRF52840のボード定義がこの新形式に切り替わった[PR](https://github.com/zmkfirmware/zmk/pull/3145)がマージされたのは2026年2月と、実はこのPoCのわずか数ヶ月前のことでした。その影響で、ボードの識別子が新しい形式にリネームされていたのです。

そのまま`build.yaml`にこの名前を書いてpushしたところ、GitHub Actionsのビルドが以下のエラーで失敗しました。

```
No board named 'xiao_ble/nrf52840/zmk' found.
```

原因を調べたところ、`config/west.yml`が実際に取得しているのはZMKの`v0.3`安定ブランチで、そちらはまだ移行前の**`seeeduino_xiao_ble`**という旧名称のままだったのです。つまり、`main`ブランチ（開発版）のドキュメントを鵜呑みにしてしまい、実際にビルドで使われる安定版との差分に気づかなかったのが原因でした。

```yaml
# build.yaml
include:
  - board: seeeduino_xiao_ble  # xiao_ble/nrf52840/zmk ではなくこちら
    shield: single_key
```

`&xiao_d 0`のようなデバイスツリーレベルの記法は`main`でも`v0.3`でも変わっていなかったので、影響を受けたのはトップレベルのボード識別子だけでした。とはいえ、ZMKのハードウェア調査をするときは、**参照しているブランチ・タグが、実際に自分の`west.yml`が指しているリビジョンと一致しているか**を確認しておいたほうが良さそうです。


GitHub Actionsでビルド、UF2を書き込み
---
`build.yaml`を直したあとは、pushするだけでGitHub Actionsが自動でファームウェアをビルドしてくれます。ローカルにZephyr SDKを入れる必要はありません。

```yaml
# .github/workflows/build.yml
name: Build ZMK firmware
on: [push, pull_request, workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3
```

ビルドが成功すると、Actionsの実行結果に`.uf2`ファイルがArtifactとして出てきます。ボードへの書き込み手順は以下の通りです。

1. XIAOのリセットボタンを素早く2回押してブートローダーモードに入る（USBマスストレージとして認識される）
2. 出てきたドライブに`.uf2`ファイルをドラッグ&ドロップ（コピー）する
3. 自動的にボードがリセットされ、書き込んだファームウェアで起動する


動作確認
---
書き込み後、PCとBluetoothでペアリングしてスイッチを押すと、期待通り`A`が入力されました。単一のスイッチとはいえ、実際にワイヤレスでキー入力が飛んでくると、思っていたよりも感動があります。


学びまとめ
---
- ZMKの設定リポジトリは、`board_root`を登録した自作シールドを`build.yaml`で指定するだけで、ローカル環境なしにGitHub Actions上でビルドできる
- `zmk,kscan-gpio-direct` + インターコネクトのピンラベル（`&xiao_d N`など）を使えば、GPIO番号を自分で計算せずに配線を指定できる
- ZMKはドキュメント・`main`ブランチと、実際にリポジトリが参照している安定ブランチ（`v0.3`など）でボード識別子が異なることがある。調査するときは参照ブランチを揃えるべし
- ハンダ付けなしのICクリップ配線でも、内部プルアップさえ設定すれば問題なく動作する

このPoCでXIAO nRF52840とZMK Firmwareの使用感が理解できたので、無線キーボードの設計に進んでいこうと思います。
