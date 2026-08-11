---
title: "Digging into the kubelet API"
date: 2022-12-19
draft: false
tags: ["kubernetes"]
---

Overview
---
I found myself wanting to build something using the kubelet API, so I did some research to figure out how, and I'd like to share what I learned.

Up until now I'd never really had a reason to touch it directly — I knew it was used for collecting container metrics, and every so often I'd think "ah, the kubelet API is out there working hard today too," and that was about the extent of my relationship with it. I suspect a lot of Kubernetes engineers feel roughly the same way.

So, unsung hero that it is, I hope this post makes the kubelet API feel just a little more familiar to you than before. ♪

> 📘【note】
> This post is day 19 of [Kubernetes Advent Calendar 2022](https://qiita.com/advent-calendar/2022/kubernetes).


What APIs are there?
---
I couldn't find documentation that spells out the kubelet API in detail, so I dug through the code starting from here:

- https://github.com/kubernetes/kubernetes/blob/v1.25.5/pkg/kubelet/server/server.go

Here's what I found.

#### Fetching Kubernetes resources

- `/pods`
    - Returns the list of Pod resources on the same Node as the kubelet.

#### Fetching metrics

- `/metrics`
    - Returns the kubelet's own assorted metrics.
- `/metrics/cadvisor`
    - Returns assorted metrics — like container CPU usage — for containers on the same Node as the kubelet.
    - The kubelet has cAdvisor built in, and this exposes the metrics it provides. For more detail on that, see [@ryysud's post](https://qiita.com/ryysud/items/23eab7110de7337a8bf3).
- `/metrics/probes`
    - Returns aggregated metrics on the pass/fail results of container probes (readiness/liveness/startup) for containers on the same Node as the kubelet.
- `/metrics/resource`
    - Returns CPU and memory usage per container and per Pod for containers on the same Node as the kubelet.
    - Metrics Server v0.6.0 and later collects metrics from this endpoint[^1].
- `/stats/summary`
    - Returns CPU, memory, network, and disk-related metrics for the Node the kubelet is running on, and for the Pods on that Node.
    - Unlike the other endpoints, which use the Prometheus exporter format, this one returns values as JSON.
    - Metrics Server v0.5.x and earlier collects metrics from this endpoint[^1].

[^1]: https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/#summary-api-source


#### Used internally by kubectl subcommands
The following are, as far as I can tell, the APIs used under the hood when you run various kubectl subcommands.

- `/run`
- `/exec`
- `/attach`
- `/portForward`
- `/logs`

Judging from the path names, these look like they're used by the `run`, `exec`, `attach`, `port-forward`, and `logs` commands respectively.

#### Others

- `/checkpoint`
    - The API used by Checkpoint Restore, available as an alpha feature since Kubernetes v1.25.
    - Available when the `ContainerCheckpoint` feature gate is enabled and you're using a container runtime that supports it.
    - There's a [KubeCon NA 2021 session](https://www.youtube.com/watch?v=0RUDoTi-Lw4) on the Checkpoint Restore feature.
- `/debug/pprof`
    - The pprof profiler endpoint.
- `/debug/flags/v`
    - Returns the list of startup flags available on the kubelet.


Authentication and authorization
---
There's official documentation on authentication and authorization:

- https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/

The key points, in brief:

- Authentication
    - Authentication is off by default.
    - You can enable X.509 client certificate authentication and API bearer token authentication.
- Authorization
    - Access is granted by giving permissions on `nodes/[sub-resource]` via a Role/ClusterRole.
    - Which sub-resource name applies depends on the API path (see the link above for the exact mapping).


Let's actually try accessing it
---
So, let's try accessing the kubelet API with bearer-token authentication turned on.
For simplicity, we'll access the kubelet API from a Pod deployed on the Kubernetes cluster itself.

First, prepare a manifest like this:

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

The ClusterRole grants full permissions on the kubelet API.

Next, apply this to the cluster.

```console
$ kubectl apply -f kubelet-api-experiment.yaml
```

Note down the IP address of some Node, since we'll need it later.

```console
$ kubectl get $(kubectl get node -o name | head -1) -o jsonpath='{.status.addresses[?(@.type == "InternalIP")].address}'
```

Get a shell into the Pod we just deployed.

```console
$ kubectl exec -it kubelet-api-experiment -- /bin/sh
```

Before we access the API, set the Node IP we noted down and the ServiceAccount token as environment variables.

```console
sh-4.2# export NODE_IP=[the IP you noted down above]

sh-4.2# export TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

Running curl like this lets you access the kubelet API on the specified Node.

```console
sh-4.2# curl -H "Authorization: Bearer $TOKEN" "https://${NODE_IP}:10250/pods"
```


So what's this actually for?
---
Okay, so we've managed to call the API — but what's it actually good for?

For the "development using the kubelet API" I mentioned at the top, my idea is to have an agent deployed as a DaemonSet on every Node fetch the list of Pods on its own Node from `/pods`.

With a DaemonSet, the number of agents grows every time you add a Node, so if each one fetched Pod info from kube-apiserver, the load on the API server would keep growing along with the cluster.
So the idea is to offload that load by fetching Pod info from the local kubelet instead.

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

Since the agent only needs enough information to do its job within the Node it's deployed on, the kubelet API is a good fit for that reason too.

Also, the `/metrics/probes` metrics seem worth wiring into monitoring.
When a Pod shuts down unexpectedly because of a failed livenessProbe, you can catch that from these metrics (if you're using the Prometheus Operator, maybe it collects these out of the box already?).

If any of you reading this have other good ideas for what this API is useful for, I'd love to hear them!


That's all.
Thanks for reading to the end. Merry Christmas!
