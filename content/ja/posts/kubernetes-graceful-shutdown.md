---
title: "アルパカでもわかる安全なPodの終了"
date: 2020-09-18
draft: false
tags: ["kubernetes"]
---


これはなに
---
KubernetesにおいてPodが終了するまでの動作を整理します。また、それを踏まえて、安全に（リクエストの欠損を極小化した）Podを終了する方法を考察します。

アプリケーションとしては、HTTPリクエストを受けてレスポンスを返却する、一般的なWebアプリケーションを想定します。

> 📘【note】
> この記事の内容は、@superbrothersさんによる[詳解 Pods の終了](https://qiita.com/superbrothers/items/3ac78daba3560ea406b2)と[公式ドキュメント](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)が元になっていますのでぜひそちらも参照ください。この記事は 2020/09/23 現在の最新版のKubernetes で内容を再確認するとともに、図を足したり、解説を増やしたりしています。


目次
---
- Podが終了する過程
- 安全なPodの終了のために注意すべきこと


Podが終了する過程
---
コンテナイメージの更新や `kubectl delete pod` の実行など、Podの終了のトリガーとなる事象が起きると、それまで起動していたPodに対する終了処理が開始されます。Podの終了処理の全体の流れは以下のとおりです。

1. Podの終了予定時刻をPodリソースに設定する
2. Podリソースをウォッチする複数のコンポーネントが、それぞれの終了処理を実行する
    - 2-a. kubeletによるプロセスのシャットダウン
    - 2-b. endpoints controllerとkube-proxyによるサービスアウト
    - 2-c. Ownerリソースによる管理からの除外

2.の3つの処理は、それぞれを担当するコンポーネントが独立して実行するため、例えば「サービスアウトしてからシャットダウンする」といったような互いに依存関係を持った制御は行われません。これは、本エントリーのテーマのひとつである、「Podの安全な終了」を考える上で重要なポイントになりますので、注意してください。

![](/images/kubernetes-graceful-shutdown-01.dio.svg)

以降は、上に挙げた各処理において具体的にどのような処理が行われているかを説明します。

### 1 . Podの終了予定時刻をPodリソースに設定する
削除対象しようとしているPodに対応するPodリソースに対して、 `.metadata.deletionTimestamp` と `.metadata.deletionGracePeriodSeconds` が設定されます。

- `.metadata.deletionTimestamp` :
    - 削除予定時刻。このフィールドの設定が行われる時刻に `.spec.terminationGracePeriodSeconds` （デフォルト: 30秒）を加算した値が設定される
- `.metadata.deletionGracePeriodSeconds` :
    - このフィールドの設定が行われる時点での `.spec.terminationGracePeriodSeconds` の値が設定される

これをきっかけに、Podリソースをウォッチしている各コンポーネントが後続の終了処理を開始します。

### 2-a. kubeletによるプロセスのシャットダウン
Podリソースに `.metadata.deletionTimestamp` が設定されたことをkubeletが検知すると、kubeletは以下のシャットダウンプロセスを開始します。

- 2-a-1. preStopフックを実行する
- 2-a-2. Dockerデーモンにコンテナの終了を依頼する

preStopフックは、プロセスの終了前に実行する事前処理です。`.spec.containers[].lifecycle.preStop` に処理内容を記述することができます。指定可能な処理は、任意のコマンドの実行、所定のエンドポイントへのHTTP GETリクエスト、TCPソケットのオープンの試行、の3種です。

preSropフックが終了するか、 `.metadata.deletionGracePeriodSeconds` の時間が経過した場合、kubeletがDockerデーモンにコンテナの終了を依頼します[^1] [^2]。このとき、終了処理のタイムアウト時間として、以下の値が渡されます。

- preStopフックが `.metadata.deletionGracePeriodSeconds` までに終了した場合:
    - `.metadata.deletionGracePeriodSeconds` からpreStopフックの所要時間で引いた値（2秒以下だった場合は2秒に切り上げ）
- preStopフックが `.metadata.deletionGracePeriodSeconds` までに終了しなかった場合
    - 2秒

コンテナの終了処理では、始めにコンテナにSIGTERMシグナルが送信されます。多くのアプリケーションでは、SIGTERMの受信を受けて終了処理を開始するように実装することが多いと思います(Gracefl Shutdown)。

SIGTERMの後、タイムアウト時間が経過してもコンテナが終了していない場合、SIGKILLシグナルが送信されます。ここで、コンテナが強制的にシャットダウンされます。

[^1]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/kubelet/kuberuntime/kuberuntime_container.go#L544-L550
[^2]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/kubelet/kuberuntime/kuberuntime_container.go#L635-L650

#### シャットダウン処理の時系列
シャットダウン処理の時系列と `.metadata.deletionGracePeriodSeconds` の関係を図に起こしてみると、以下のようになります。

##### preStopフックが `.metadata.deletionGracePeriodSeconds` までに終了した場合

![](/images/kubernetes-graceful-shutdown-02.dio.svg)

##### preStopフックが `.metadata.deletionGracePeriodSeconds` までに終了しなかった場合
この場合、preStopフックの終了を待たずにコンテナの終了処理に移行します。このときタイムアウトは固定で2秒ですので、SIGTERMの送信後2秒が経過するとSIGKILLが送信されます。

![](/images/kubernetes-graceful-shutdown-03.dio.svg)

### 2-b. endpoints controllerとkube-proxyによるサービスアウト
Podリソースに `metadata.deletionTimestamp` が設定されると、endpoints controllerがServiceリソースからPodのendpointを除外します[^3]（この処理は、endpointSliceを有効にしている場合はそちらで同等のことが行なわれます）。

Serviceリソースからendpointが除外されると、kube-proxyがトラフィックの配送ルールを更新（iptablesプロキシーモードの場合、Nodeのiptablesを更新[^4]）し、これによってPodに対して新規TCPコネクションが作成されないようになります（サービスアウト）。

![](/images/kubernetes-graceful-shutdown-04.dio.svg)

[^3]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/controller/endpoint/endpoints_controller.go#L398-L401
[^4]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/proxy/iptables/proxier.go#L569-L571

### 2-c. Ownerリソースによる管理からの除外
Ownerリソースは、あるリソースに対してそれを管理する関係にある上位のリソースです。Podリソースの場合、ReplicaSet、DaemonSetなどが該当します。ReplicaSetやDaemonSet、はたまたReplicaSetの更にOwnerリソースとなるDeploymentなどを `kubectl create` することでPodを起動している場合、そのPodはOwnerの管理下にあります。

![](/images/kubernetes-graceful-shutdown-05.dio.svg)

Podリソースに `.metadata.deletionTimestamp` が設定されると、Ownerリソースの管理下からPodが除外されます。

#### ReplicaSet配下のPodを削除した場合の挙動
ここでは、配下のPodが削除されたときのReplicaSetの動作を見てみます。

ReplicaSetは、管理下にあるPodを所定のReplica数に維持する機能を持るリソースです。
ReplicaSetのコントローラーは、突き合わせループの際に配下のPod数を毎回チェックしており、 `.metadata.deletionTimestamp` が設定されたPodはこのときのPod数にカウントされないようになっています[^5] [^6]。

これによって、配下のPod数がReplicaSetに設定されたReplica数より少ないと判定され、新たなPodの作成が実行されます[^7]。

![](/images/kubernetes-graceful-shutdown-06.dio.svg)

以上のことから、Podの削除が実行されると、コンテナの終了処理を待たずに新しいPodの作成が行なわれることになります。

[^5]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/controller/replicaset/replica_set.go#L685
[^6]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/controller/controller_utils.go#L910-L927
[^7]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/controller/replicaset/replica_set.go#L696


安全なPodの終了のために注意すべきこと
---
改めてPodが終了するまでの過程を時系列に整理すると、以下のようになります。

![](/images/kubernetes-graceful-shutdown-07.dio.svg)

Podリソースに `.metadata.deletionTimestamp` が設定されて以降、3種類の処理が走ることになりますが、ここで重要なのはそれらが互いに依存関係を持たず、独立して実行されるということです。
このため、サービスアウトが行なわれる前にコンテナがシャットダウン処理に入ってしまい、一部のトラフィックがエラーになってしまうということが起こりえます。

これを防止するには、以下のような対策が必要になります。

### 対策1: preStopフックでのsleep
preStopフックで十分な時間sleepし、サービスアウトが完了してからSIGTERMが送られるようにします。
SIGTERMをきっかけにGraceful Shutdownを開始し、その中で接続済みのコネクションの処理が終了してからプロセスを停止します。

![](/images/kubernetes-graceful-shutdown-08.dio.svg)

### 対策2: 最強のGraceful Shutdown
preStopフック、またはSIGTERMをきっかけにGraceful Shutdownを開始します。
Graceful Shutdownの処理では、新規コネクションを受け入れつつ全てのコネクションの処理が終了してからプロセスを停止します。

![](/images/kubernetes-graceful-shutdown-09.dio.svg)

### 対策案の比較
対策1はGraceful Shutdownの実装が比較的容易な一方、余裕を持ってsleep時間を設定するとPodの終了が遅くなるデメリットがあります。とはいえ、ReplicaSetの挙動で見たとおり、 `.metadata.deletionTimestamp` がPodに設定された時点で、OwnerリソースからはそのPodは終了したものとして扱われますので、実質的な害はあまりないかもしれません。

対策2はGraceful Shutdownの処理がやや複雑になりますが、sleep時間の設定を意識する必要はありません。そのため複雑さをアプリケーションに寄せた方法と言えます。
とはいえ、例えばSpring Bootには新規コネクションを受け入れつつシャットダウンを進めるようなGraceful Shutdownができない場合もある[^8]ようです。そういった場合はこちらを採用するのはハードルが高いでしょう。

[^8]: https://docs.spring.io/spring-boot/docs/2.3.4.RELEASE/reference/htmlsingle/#boot-features-graceful-shutdown


次回予告
---
この記事では机上の考察にとどまりましたが、実際に実験してみようと思います！
