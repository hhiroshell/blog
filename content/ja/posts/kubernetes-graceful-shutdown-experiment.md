---
title: "アルパカでもわかる安全なPodの終了 - 実験編"
date: 2020-12-17
draft: false
tags: ["kubernetes"]
---


これはなに
---
この記事は[アルパカでもわかる安全なPodの終了](https://zenn.dev/hhiroshell/articles/kubernetes-graceful-shutdown)の続編です。

前回は、Podの終了時の動作をKubernetesの各種コンポーネントの仕組みを踏まえつつ考察しました。
Deploymentのローリングアップデートを行うとPodの再起動を伴うことになりますが、このときリクエストを欠損なく処理するために、以下2つの対策が有効であることが分かりました。

- preStopフックでのスリープ
- アプリケーションのGraceful Shutdown

この記事では、Deploymentのローリングアップデートをオンラインで実際に行い、上記対策が本当に有効かどうかを確かめていきます。

> 📘【note】
> この記事は[Kubernetes2 Advent Calendar 2020](https://qiita.com/advent-calendar/2020/kubernetes2)の18日目です。
> 昨日は@south37さんの[手を動かして学ぶコンテナ標準 - Container Runtime 編](https://south37.hatenablog.com/entry/2020/12/11/%E6%89%8B%E3%82%92%E5%8B%95%E3%81%8B%E3%81%97%E3%81%A6%E5%AD%A6%E3%81%B6%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E6%A8%99%E6%BA%96_-_Container_Runtime_%E7%B7%A8)でした。


安全にPodを終了するための2つの対策
---
上述のとおり、Podが安全に終了するためにできることは大きく2つがあります。以下にそれぞれの概要を記します。
内部的な動作の詳細については、[前回の記事](https://zenn.dev/hhiroshell/articles/kubernetes-graceful-shutdown)を参照ください。

### preStopフックでのスリープ
preStopフックは、コンテナを停止する前に実行される前処理（フック）を定義する機能です。
preStopフックで一定時間の `sleep` を行うと、コンテナが停止される前に指定した時間だけ待機する動作になります。
これによって、Podへのトラフィックの配送が止まってから（サービスアウトしてから）終了処理に入るようにすることができます。

![](/images/kubernetes-graceful-shutdown-experiment-01.dio.svg)

ここで注意すべき点は、サービスアウトとpreStopフックに依存関係は持たせられず、サービスアウトを確認してからpreStopフックを抜けるといったような制御はできないことです。
このため、コンテナの終了処理をサービスアウトの後に行う、ということを保証することはできません。

### アプリケーションのGraceful Shutdown
Graceful Shutdownをアプリケーションに実装すると、アプリケーションのシャットダウンが開始されたとき、その時点で受け付けているリクエストが処理されてからプロセスを終了するということが保証できます。

![](/images/kubernetes-graceful-shutdown-experiment-02.dio.svg)

ただし、アプリケーションのシャットダウンが開始されて以降は、新たにリクエストを受け付けることはできないことに注意してください。

それでは、これら2つの対策がローリングアップデート中のエラーの抑制に役立つのか、実験して確かめていきたいと思います。


実験してみた！
---

### 実験の流れ
実験用のアプリケーションをKubernetesクラスターにデプロイしておき、一定量のトラフィックを送ります。
リクエストが送られている間に、Deploymentの再起動 (`kubectl rollout restart`) を行います。

![](/images/kubernetes-graceful-shutdown-experiment-03.dio.svg)

実際の運用場面では、再起動ではなくローリングアップデートが行われることが多いと思いますが、Podの停止・起動さえされれば検証の目的としては足りるため、 `kubectl rollout restart` で代替します。

### アプリケーション
実験用にサンプルアプリケーションを用意しています。アプリの概要は以下のとおりです。

- https://github.com/hhiroshell/cowweb-go/tree/v1.1.1
- Go製のサンプルアプリケーション
- 起動フラグで1リクエストの処理でかかるCPU負荷を調整できる[^1]
- 起動フラグで終了時にGraceful Shutdownを行うかどうかを指定することができる
- preStopフックはDeploymentのマニフェストに記述してデプロイする

[^1]: 内部処理でループする回数を変えるだけなので、millicoresなどの単位が指定できるわけではありません。

#### CPU負荷の調整方法

ランダム値を生成する処理を繰り返すことでCPU負荷がかかるようにしています。
ループ回数を起動フラグで `l=640` などとすることで指定できます。

- https://github.com/hhiroshell/cowweb-go/blob/823894c18cbdec4c796e6b91deab078034d75fb8/pkg/infrastructure/cowsay/slow_cowsay.go#L19-L23

```go
	// c.load が l フラグで指定した値となる
	for i := 0; i < c.load; i++ {
		for j := 0; j < c.load; j++ {
			rand.Intn(len(cows))
		}
	}
```

#### Graceful Shutdownの実装方法
Graceful ShutdownはGo標準の `http.Server.Shutdown()` を使って実装しています。
こちらも起動フラグで、Gracefulに終了するかどうかを指定できる仕掛けにしています。

- https://github.com/hhiroshell/cowweb-go/blob/v1.1.1/cmd/cowweb/serve.go#L53-L61

```go
		sig := make(chan os.Signal)
		defer close(sig)
		signal.Notify(sig, syscall.SIGTERM, os.Interrupt)
		<-sig
		if *shutdownGracefully {
			if err := server.Shutdown(context.Background()); err != nil {
				log.Print(err)
			}
		}
```

#### preStopスリープの指定方法
preStopフックによるスリープはDeploymentのmanifestに記述します。
以下は、preStopフックとして `sleep 5` を実行している例です。

- https://github.com/hhiroshell/k8s-rolling-update-experiment/blob/master/cowweb/overlay-go/prestop-sleep.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cowweb
spec:
  template:
    spec:
      containers:
        - name: cowweb
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
```


### やってみた！
それでは実際に、preStopスリープとGraceful Shutdownの効果を検証してみます。

#### 1. preStopスリープの効果確認
以下の3通りの条件での結果を比較してみます。
preStopスリープの時間を3通り(0s, 5s, 8s)に変え、それ以外の条件は同じにしてあります。

|#                          |条件1-a(0s)|条件1-b(5s)|条件1-c(8s)
|-                          |-          |-          |-
|**preStopスリープ(s)**     |**0**      |**5**      |**8**
|Graceful Shutdown          |true       |true       |true
|レプリカ数                 |8          |8          |8
|CPU負荷フラグ              |l=640      |l=640      |l=640
|最大秒間リクエスト数(rps)  |200        |200        |200

結果は以下のとおりです。

|#                      |条件1-a(0s)|条件1-b(5s)|条件1-c(8s)
|-                      |-          |-          |-
|総リクエスト数         |24060      |24060      |24060
|2xx 系レスポンス       |23008      |23900      |24060
|5xx 系レスポンス       |1052       |160        |0
|エラー率               |**4 %**    |**1 %**    |**0 %**
|平均レスポンス時間(ms) |23         |29         |29

preStopスリープを行わない(0s)のケースでは4%のリクエストがエラーとなっていますが、5sのスリープによって1%に、8sでは0%という結果になりました。
この結果を見る限り、preStopフックにエラーを抑制する効果があるように見えますが、実際にはKubernetesとアプリケーションにはどのような状況が起きているのでしょうか。

以下の図は、ローリングアップデートに伴うPodの終了時の動作を描いたもので、preStopスリープの長さが十分でないためにエラーが発生するケースを表しています。

![](/images/kubernetes-graceful-shutdown-experiment-04.dio.svg)

注目すべき点は、kube-proxyによりiptablesが更新される前の時点でpreStopスリープが終わり、コンテナにSIGTERMが送信されていることです。
これは、アプリケーションへのトラフィックへの配送がまだ続いている（サービスアウトしていない）にも関わらず、アプリケーションのシャットダウンが始まっていることを意味します。

実験用のGoアプリケーションは、（多くのプロダクションのアプリケーションも同様と思われますが）シャットダウン処理が始まると新規リクエストを受け付けることができず、レスポンスとしてはエラーが返却されます。
preStopスリープの時間を十分に取ることで、サービスアウトしてからコンテナのシャットダウンが開始されるようになり、8sのスリープの実験結果のように、エラーを抑制することができます。

#### 2. Graceful Shutdownの効果確認
次に、Graceful Shutdownの効果を調べるために、以下2通りの条件で実験してみます。
Graceful Shutdownをする/しないの条件以外は同じにしてあります（CPU負荷フラグ、最大リクエスト数の値が先程と異なりますが、これの理由は後述します）。

|#                          |条件2-f    |条件2-t
|-                          |-          |-
|preStopスリープ(s)         |8          |8
|**Graceful Shutdown**      |**false**  |**true**
|レプリカ数                 |8          |8
|CPU負荷フラグ              |l=1024     |l=1024
|最大秒間リクエスト数(rps)  |100        |100

結果は以下のとおりです。

|#                      |条件2-f    |条件2-t
|-                      |-          |-
|総リクエスト数         |12060      |12060
|2xx 系レスポンス       |11416      |12060
|5xx 系レスポンス       |644        |0
|エラー率               |**5 %**    |**0 %**
|平均レスポンス時間(ms) |47         |48

Graceful Shutdownをしない条件では5%ほどがエラー、する条件ではそのエラーが0%となっています。
どうやらGraceful Shutdownにもローリングアップデート時のエラーを抑制する効果があるようですが、このケースではどのようなことが起こっているのでしょうか。

以下の図は、Graceful Shutdownによってエラーを抑制可能なケースでの、Pod終了時の動作を表しています。

![](/images/kubernetes-graceful-shutdown-experiment-05.dio.svg)

この場合では、kube-proxyによりiptablesが更新されてから（サービスアウトしてから）コンテナの終了処理に入っています。
一見なにも問題ないように見えますが、そうとは言い切れません。
サービスアウトはされたとしても、それ以前にアプリケーションが受け付けたリクエストがまだ処理中かもしれないからです。
こういったリクエストがまだ残っている状態でGracefulでない終了を行ってしまうと、リクエストの処理が強制的に中断され、レスポンスはエラーとなってしまいます。

Graceful Shutdownを行うことによって、処理中のリクエストの処理が完了してからアプリケーションプロセスを終了する動作となり、上記の実験結果のようにエラーを抑制することができます。


考察 - 運用環境への適用に向けて
---
ここまでの実験で、preStopスリープ、Graceful Shutdonwのどちらもローリングアップデート時のエラーの抑制に効果があることが分かりました。
それでは、実際の運用環境で使っていくには、他にどのようなことを考慮する必要があるでしょうか。

### preStopスリープの長さ
preStopスリープの時間に関しては、サービスアウト処理が実際にどのくらいの速さで完了するかを考慮する必要があります。
サービスアウト処理は、endpoints-controllerによるReconcile処理、kube-proxyによるiptablesの更新が該当しますが、これらはクラスター規模（Serviceリソース数や配下のPod数、クラスターのNode数など）によって変わってくると予想されます。

それぞれの環境ごとに実際にサービスアウトにかかる時間を把握した上で、これを超えるようにpreStopスリープを設定してください。

### Graceful Shutdownの要否
Graceful Shutdownの要否はアプリケーションの特性によって判断していく必要があります。
例えば、リクエストの処理の中断によってデータ不整合などが起きうる場合では、しっかりとGraceful Shutdownする必要があるでしょう。

処理を中断しても問題ないことが明らかであったり、なるべく早く再起動したい特別な事情があるという場合に限り、Graceful Shutdownを行わない選択をとると考えるのが無難ではないでしょうか。

> 📘【note】
> preStopスリープを十分長く取って、処理中のリクエストさえなくなるまで待った上でシャットダウンに入る、という考え方もできそうですが、これは確実ではないと考えます。
> 外部サービスやDBに依存するようなアプリケーションでは、それら外部コンポーネントから予想以上の影響をうけることがあります。そうしてリクエストの処理に想定以上の時間がかかってしまい、レスポンスを返す前にシャットダウンが始まってしまうということがあり得るのではないでしょうか。


まとめ
---
本記事では、Deploymentのローリングアップデートに伴う再起動において、リクエストの欠損を防ぐために、以下の対策が有効であることが確認できました。

1. preStopフックでのスリープ
2. アプリケーションのGraceful Shutdown

これらを実際に運用環境に適用するに当たっては、1. preStopフックでのスリープ については実際のクラスターの規模を、2. Graceful Shutdown についてはアプリケーションの特性を考慮する必要があります。

以上です。最後まで読んでいただきありがとうございました！
