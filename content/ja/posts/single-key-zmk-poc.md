---
title: "スイッチ1個・ハンダ付けなしでZMKワイヤレスキーボードのPoCを作った"
date: 2026-09-18
draft: false
tags: ["zmk", "keyboard"]
---

これはなに
---
自作キーボードのファームウェアとして定番の[ZMK](https://zmk.dev/)を、まだ一度も触ったことがありませんでした。とはいえ、いきなり何十個もスイッチが並んだ本格的なキーボードを組むのはハードルが高いので、まずは最小構成で「ZMKの設定リポジトリ作成 → ビルド → 書き込み → Bluetooth接続」までの一連の流れを体験してみることにしました。

用意したのはスイッチ1個だけ。ハンダ付けもブレッドボードも使わず、マイコンのパッドにICクリップを挟むだけの配線です。とにかく元に戻しやすい状態を保ちながら、ZMKの勘所を掴むのが目的のPoCです。

> 📘【note】
> この記事はClaude Codeと一緒に進めた作業のログを元にしています。ZMKの仕様調査やファイル作成、GitHub Actionsでのビルド確認、実機への書き込みまでを一緒にやってもらいました。


ハードウェア構成
---
- マイコン: [Seeed Studio XIAO nRF52840](https://wiki.seeedstudio.com/XIAO_BLE/)
- スイッチ: モーメンタリ・ノーマルオープンのキースイッチ1個
- 配線: XIAO本体のキャスタレーションパッド（基板端の半円形の切り欠き）にSMD ICテストフック（グラバー）を直接クランプし、スイッチの端子へ2本だけ配線
    - `D0`（`P0.02`） → スイッチの一方の端子
    - `GND` → スイッチのもう一方の端子
- 外付けのプルアップ抵抗はなし。GPIOの内部プルアップに依存
- 電源: USB-C給電（モバイルバッテリー）

配線はこんな感じです。

![XIAO nRF52840とキースイッチの配線](/images/single-key-zmk-poc-01.jpg)

はんだ付けなしで気軽に試せるように、モバイルバッテリーからUSB-C給電しつつ動かしました。キーキャップには実際に送信するキーコードと同じ「A」を貼っています。

![モバイルバッテリー給電でのセットアップ全体](/images/single-key-zmk-poc-02.jpg)


ZMK設定リポジトリを作る
---
ZMKには公式の[ZMK CLI](https://github.com/zmkfirmware/zmk-cli)（`uv tool install zmk`→`zmk init`）が用意されていて、対話的にGitHubリポジトリの作成までできるようです。ただ今回は、生成される構成をきちんと理解したかったのと、CLIの対話フローを自動でなぞらせるのも大変だったので、ZMK公式の[unified-zmk-config-template](https://github.com/zmkfirmware/unified-zmk-config-template)をベースに、必要なファイルを手で組み立てました。

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
行列スキャン(`zmk,kscan-gpio-matrix`)は複数のスイッチをマトリクス配線で効率よく読み取るための仕組みなので、スイッチ1個だけならオーバースペックです。GPIOを直接読みに行く`zmk,kscan-gpio-direct`で十分でした。

**`&xiao_d 0`でD0ピンを指定する**
XIAO nRF52840は`seeed_xiao`という「インターコネクト」定義を公開していて、そのノードラベル`xiao_d`経由でピン番号を指定できます。ZMKの設計ガイドラインに「D0を参照するには`&xiao_d 0`を使う」と明記されており、`&gpio0 2`のように自分でP0.02を計算する必要がありませんでした。これは正直ちょっと感動しました。

**`GPIO_ACTIVE_LOW | GPIO_PULL_UP`**
外付けプルアップ抵抗を使わず、スイッチはGND側でONになる配線なので、「内部プルアップ有効化」＋「Lowアクティブ」の組み合わせが正解です。

**matrix-transformは1x1でも一応残した**
スイッチ1個なら`zmk,kscan`だけをchosenに指定して`matrix-transform`を省略しても問題なくビルドは通ります。ただ、今後スイッチを増やしていくときのひな形として、他の複数キーシールドと同じ構造に揃えておきたかったので、あえて1x1の`matrix-transform`を残しています。


つまずいたポイント：ボード名がドキュメントと違う
---
ここが今回のPoCで一番の学びでした。

ZMKの公式ドキュメントやGitHubリポジトリの`main`ブランチを見ながら作業していたところ、XIAO nRF52840のボード名は`xiao_ble/nrf52840/zmk`と表記されていました。ZMKが2024年以降、Zephyrの「Hardware Model v2」という新しいボード定義の仕組みに移行した影響で、ボードの識別子が新しい形式にリネームされていたのです。

言われた通りに`build.yaml`にこの名前を書いてpushしたところ、GitHub Actionsのビルドが以下のエラーで失敗しました。

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

`&xiao_d 0`のようなデバイスツリーレベルの記法は`main`でも`v0.3`でも変わっていなかったので、影響を受けたのはトップレベルのボード識別子だけでした。とはいえ、ZMKのハードウェア調査をするときは、**参照しているブランチ・タグが、実際に自分の`west.yml`が指しているリビジョンと一致しているか**を確認する癖をつけておいたほうが良さそうです。


GitHub Actionsでビルド、UF2を書き込み
---
`build.yaml`を直したあとは、pushするだけで[GitHub Actionsが自動でファームウェアをビルド](https://github.com/hhiroshell/zmk-config/actions)してくれます。ローカルにZephyr SDKを入れる必要はありません。

```yaml
# .github/workflows/build.yml
name: Build ZMK firmware
on: [push, pull_request, workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3
```

ビルドが成功すると、Actionsの実行結果に`.uf2`ファイルがArtifactとして出てきます。書き込み手順は以下の通りです。

1. XIAOのリセットボタンを素早く2回押してブートローダーモードに入る（USBマスストレージとして認識される）
2. 出てきたドライブに`.uf2`ファイルをドラッグ&ドロップ（コピー）する
3. 自動的にボードがリセットされ、書き込んだファームウェアで起動する

今回はドラッグ&ドロップの代わりに、マウントされたドライブに`cp`コマンドでコピーするだけで書き込めました。GUI操作なしで完結するので、この部分は地味に便利でした。


動作確認
---
書き込み後、PCとBluetoothでペアリングしてスイッチを押すと、期待通り`A`が入力されました。単一のスイッチとはいえ、実際にワイヤレスでキー入力が飛んでくると、思っていたよりも感動があります。


学びまとめ
---
- ZMKの設定リポジトリは、`board_root`を登録した自作シールドを`build.yaml`で指定するだけで、ローカル環境なしにGitHub Actions上でビルドできる
- `zmk,kscan-gpio-direct` + インターコネクトのピンラベル（`&xiao_d N`など）を使えば、GPIO番号を自分で計算せずに配線を指定できる
- ZMKはドキュメント・`main`ブランチと、実際にリポジトリが参照している安定ブランチ（`v0.3`など）でボード識別子が異なることがある。調査するときは参照ブランチを揃えるべし
- ハンダ付けなしのICクリップ配線でも、内部プルアップさえ設定すれば普通に安定して動く

次はスイッチを増やして、実際にタイピングできる小さなキーボードを組んでみようと思います。
