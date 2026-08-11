---
title: "kubeletのAPIを調べてみた"
date: 2022-12-19
draft: false
tags: ["kubernetes"]
---


これはなに
---
ひょんなことからkubeletのAPIを使った開発をしてみたくなりまして、そのための調査をしたので共有したいと思います。

自分は今までこれといって触る機会がなく、コンテナのメトリクスを取るのに使われているということで、たまーに「ああ、kubeletさんのAPIは今日も頑張ってくれているのだなぁ」と思いを馳せるくらいの温度感でした。Kubernetesエンジニアの多くが、同じような感想を持たれているのではないでしょうか。

そんな縁の下の力持ち、kubeletのAPIさん、この記事をきっかけに、今までより少しだけ身近に感じてくれたらいいなって、そんなふうに思います♪

> 📘【note】
> この記事は[Kubernetes Advent Calendar 2022](https://qiita.com/advent-calendar/2022/kubernetes)の19日目です。


どんなAPIがあるの
---
kubeletのAPIの詳細を記したドキュメントが見つけられなかったので、以下の箇所を起点にして、コードを追って調べていきます。

- https://github.com/kubernetes/kubernetes/blob/v1.25.5/pkg/kubelet/server/server.go

以下、調査の結果わかったもの。

#### Kubernetesリソースの取得

- `/pods`
    - kubeletと同一Node上の、Podリソースのリストを取得できる

#### メトリクスの取得

- `/metrics`
    - kubelet自身の各種メトリクスがを取得できる
- `/metrics/cadvisor`
    - kubeletと同一Node上の、コンテナのCPU使用量などの各種メトリクスを取得できる
    - kubeletにはcAdvisorが組み込まれており、それが提供するメトリクスを取得する。そのあたりの詳しい話は[@ryysudさんの記事](https://qiita.com/ryysud/items/23eab7110de7337a8bf3)を参照ください
- `/metrics/probes`
    - kubeletと同一Node上の、コンテナのProbe(Readiness/Liveness/StartUp)の成否を集計したメトリクスを取得できる
- `/metrics/resource`
    - kubeletと同一Node上の、コンテナ単位、Pod単位それぞれのCPU、メモリ使用量を取得できる
    - v0.6.0以降のMetrics Serverはこのエンドポイントからメトリクスを収集している[^1]
- `/stats/summary`
    - kubeletがあるNodeと、そのNode上のPodのCPU、メモリ、ネットワーク、ディスク関連のメトリクスを取得できる
    - 他はPrometheusのExporterの形式だが、このエンドポイントはJSONで値が取得できる
    - v0.5.x以前のMetrics Serverはこのエンドポイントからメトリクスを収集している[^1]

[^1]: https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/#summary-api-source


#### kubectlコマンドの機能で内部的に利用されるもの
以下は、kubectlのサブコマンドを実行したときに使われているAPIと推測されます。

- `/run`
- `/exec`
- `/attach`
- `/portForward`
- `/logs`

パスの名前から察するに `run`, `exec`, `attach`, `port-forward`, `logs` の各コマンドで使われているものだと思われます。

#### 他

- `/checkpoint`
    - Kubernetes v1.25からalpha機能として提供されているCheckpoint Restoreで使われるAPI
    - `ContainerCheckpoint` フィーチャーゲートを有効にしつつ、コンテナランタイムとしてこれに対応するものを利用していると利用可能になる
    - Checkpoint Restore機能については、[KubeCon NA 2021のセッション](https://www.youtube.com/watch?v=0RUDoTi-Lw4)があります
- `/debug/pprof`
    - pprofプロファイラのエンドポイント
- `/debug/flags/v`
    - kubeletで利用可能な起動フラグの一覧を取得できる


認証、認可の仕組み
---
認証、認可については以下のドキュメントがあります。

- https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/

ポイントをピックアップすると、以下のような感じです。

- 認証
    - 認証はデフォルトではOFF
    - X.509クライアント証明書認証とAPI Bearer Token認証をかけることができる
- 認可
    - Role/ClusterRoleで `nodes/[sub-resource]` に対する権限を与えるとアクセスが許可される
    - APIのパスに応じて、対応するsub-resource名が異なってくる（具体的な対応関係は上記のリンクを参照）


実際にアクセスしてみる
---
というわけで、API Bearer Tokenによる認証がかかったkubeletのAPIにアクセスしてみます。
ここでは簡単のためにKubernetes上にデプロイしたPodから、kubeletのAPIにアクセスすることにします。

まずは、以下のようなmanifestを用意します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubelet-api-experiment
spec:
  containers:
  - name: centos
    image: centos7
    command: ["/bin/sh", "-c"]
    args:
      - |
        tail -f /dev/null
  serviceAccount: kubelet-api-experiment
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubelet-api-experiment
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kubelet-api-experiment
rules:
- apiGroups:
  - ""
  resources:
  - nodes/log
  - nodes/metrics
  - nodes/proxy
  - nodes/stats
  verbs:
  - get
  - create
  - update
  - patch
  - delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubelet-api-experiment
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubelet-api-experiment
subjects:
- kind: ServiceAccount
  name: kubelet-api-experiment
  namespace: default
```

ClusterRoleでkubeletのAPIに対する全権限がつくようにしています。

次にこれをクラスタにapplyします。

```console
$ kubectl apply -f kubelet-api-experiment.yaml
```

後で使うので適当なNodeのIPアドレスを控えておきます。

```console
$ kubectl get $(kubectl get node -o name | head -1) -o jsonpath='{.status.addresses[?(@.type == "InternalIP")].address}'
```

デプロイしたPodのコンソールに入ります。

```console
$ kubectl exec -it kubelet-api-experiment -- /bin/sh
```

APIにアクセスする前の準備として、上で控えておいたNodeのIPと、ServiceAccountのトークンを環境変数に設定しておきます。

```console
sh-4.2# export NODE_IP=[上で控えたIP]

sh-4.2# export TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

以下のようにcurlを実行すると、指定したNode上のkubeletのAPIにアクセスできます。

```console
sh-4.2# curl -H "Authorization: Bearer $TOKEN" "https://${NODE_IP}:10250/pods"
```


結局何につかうの
---
というわけで一応APIを呼び出すところまでできはしたわけですが、一体どんな使いみちがあるのでしょうか。

冒頭に書いた「kubelet APIを使った開発」では、DaemonSetとして各Nodeに配置するエージェントで、自分がいるNode上のPod一覧を `/pods` から取得しようと考えています。

DaemonSetでデプロイするとNodeを足す毎にエージェントの数が増えていくので、kube-apiserverからPod情報を取得していると、そこにかかる負荷がどんどん大きくなってします。
そこで、Node上のkubeletからPod情報を取得して負荷をオフロードするというアイデアです。

```
  Node A
+--------------------------------------------------------------------------+
|                                                  Get Pods on             |
|                                   +------------+  this Node  +---------+ |
|  Do Something Super Awesome! <----+ Daemon Pod +-------------> kubelet | |
|                                   +------------+             +---------+ |
|                                                                          |
+--------------------------------------------------------------------------+
  Node B
+--------------------------------------------------------------------------+
|                                                  Get Pods on             |
|                                   +------------+  this Node  +---------+ |
|  Do Something Super Awesome! <----+ Daemon Pod +-------------> kubelet | |
|                                   +------------+             +---------+ |
|                                                                          |
+--------------------------------------------------------------------------+
  Node C
  ...
```

エージェントは自分がデプロイされたNode内で仕事をするだけの情報が手に入ればいいので、そういう意味でもkubeletのAPIは都合が良いですし。

あと、`/metrics/probes` のメトリクスは監視に仕込んでおくと使い所がありそうです。
Podが予期せぬシャットダウンを起こしたとき、livenessProbeの失敗が原因であればこちらのメトリクスから補足できます（Prometheus Operatorを使ってると最初からこれを集めるようになっていたりするのかな？）。

これを呼んで頂いている方からも、なにかいいアイデアがあったら教えていただけると嬉しいです！


以上です。
最後まで読んでいただきありがとうございました。メリークリスマス！
