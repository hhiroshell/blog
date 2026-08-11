---
title: "Safe Pod Termination — Even an Alpaca Can Understand It"
date: 2020-09-18
draft: false
tags: ["kubernetes"]
---

Overview
---
This post organizes what actually happens from the moment a Pod starts terminating in Kubernetes until it's gone. Building on that, it looks at how to terminate Pods safely — that is, with minimal loss of in-flight requests.

The application model assumed here is a typical web application that receives HTTP requests and returns responses.

> 📘【note】
> This post is based on [@superbrothers's "詳解 Pods の終了" (A Detailed Look at Pod Termination)](https://qiita.com/superbrothers/items/3ac78daba3560ea406b2) and the [official documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) — please check those out too. This post re-verifies the content against the latest Kubernetes as of 2020/09/23, and adds diagrams and extra explanation.


Table of Contents
---
- The Pod termination process
- What to watch out for when terminating Pods safely


The Pod termination process
---
When something triggers Pod termination — updating a container image, running `kubectl delete pod`, and so on — the termination process for the previously running Pod begins. The overall flow looks like this:

1. The Pod's scheduled deletion time is set on the Pod resource
2. Several components watching the Pod resource each run their own termination handling
    - 2-a. Process shutdown by the kubelet
    - 2-b. Service removal by the endpoints controller and kube-proxy
    - 2-c. Removal from management by the owner resource

The three pieces of step 2 are each carried out independently by the component responsible for them, so there's no coordination between them — for example, there's no guarantee that a Pod is removed from service *before* it shuts down. Keep this in mind, since it's a key point for one of this post's central themes: safe Pod termination.

![](https://raw.githubusercontent.com/hhiroshell/alpaca-notes/master/articles/images/kubernetes-graceful-shutdown-01.dio.svg)

Below, we'll look at exactly what happens in each of these steps.

### 1. The Pod's scheduled deletion time is set on the Pod resource
`.metadata.deletionTimestamp` and `.metadata.deletionGracePeriodSeconds` are set on the Pod resource that corresponds to the Pod being deleted.

- `.metadata.deletionTimestamp`:
    - The scheduled deletion time. It's set to the time this field is written plus `.spec.terminationGracePeriodSeconds` (default: 30 seconds).
- `.metadata.deletionGracePeriodSeconds`:
    - Set to the value of `.spec.terminationGracePeriodSeconds` at the moment this field is written.

This is the trigger that causes each component watching the Pod resource to start its own termination handling.

### 2-a. Process shutdown by the kubelet
When the kubelet detects that `.metadata.deletionTimestamp` has been set on the Pod resource, it starts the following shutdown process.

- 2-a-1. Run the preStop hook
- 2-a-2. Ask the Docker daemon to stop the container

The preStop hook is processing that runs before the process is terminated. You describe it in `.spec.containers[].lifecycle.preStop`. Three kinds of action are supported: running an arbitrary command, sending an HTTP GET request to a given endpoint, or attempting to open a TCP socket.

Once the preStop hook finishes, or once `.metadata.deletionGracePeriodSeconds` has elapsed — whichever comes first — the kubelet asks the Docker daemon to stop the container[^1] [^2]. At that point, the following value is passed as the timeout for the shutdown process:

- If the preStop hook finished before `.metadata.deletionGracePeriodSeconds`:
    - `.metadata.deletionGracePeriodSeconds` minus the time the preStop hook took (rounded up to 2 seconds if the result is 2 seconds or less)
- If the preStop hook did not finish by `.metadata.deletionGracePeriodSeconds`:
    - 2 seconds

Container shutdown begins by sending a SIGTERM signal to the container. Many applications are implemented to start their shutdown handling on receiving SIGTERM (graceful shutdown).

If the container still hasn't terminated once the timeout elapses after SIGTERM, a SIGKILL signal is sent, forcibly shutting the container down.

[^1]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/kubelet/kuberuntime/kuberuntime_container.go#L544-L550
[^2]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/kubelet/kuberuntime/kuberuntime_container.go#L635-L650

#### Timeline of the shutdown process
Here's a diagram of the shutdown timeline and how it relates to `.metadata.deletionGracePeriodSeconds`.

##### When the preStop hook finishes before `.metadata.deletionGracePeriodSeconds`

![](https://raw.githubusercontent.com/hhiroshell/alpaca-notes/master/articles/images/kubernetes-graceful-shutdown-02.dio.svg)

##### When the preStop hook does not finish by `.metadata.deletionGracePeriodSeconds`
In this case, container shutdown proceeds without waiting for the preStop hook to finish. The timeout here is a fixed 2 seconds, so SIGKILL is sent 2 seconds after SIGTERM.

![](https://raw.githubusercontent.com/hhiroshell/alpaca-notes/master/articles/images/kubernetes-graceful-shutdown-03.dio.svg)

### 2-b. Service removal by the endpoints controller and kube-proxy
Once `metadata.deletionTimestamp` is set on the Pod resource, the endpoints controller removes the Pod's endpoint from the Service resource[^3] (if EndpointSlice is enabled, the equivalent happens there instead).

Once the endpoint is removed from the Service resource, kube-proxy updates the traffic-routing rules (in iptables proxy mode, this means updating the Node's iptables rules[^4]), which stops new TCP connections from being created to the Pod — i.e., it's removed from service.

![](https://raw.githubusercontent.com/hhiroshell/alpaca-notes/master/articles/images/kubernetes-graceful-shutdown-04.dio.svg)

[^3]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/controller/endpoint/endpoints_controller.go#L398-L401
[^4]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/proxy/iptables/proxier.go#L569-L571

### 2-c. Removal from management by the owner resource
An owner resource is a higher-level resource that manages a given resource. For a Pod resource, this would be a ReplicaSet, a DaemonSet, and so on. If you're running a Pod via `kubectl create` on a ReplicaSet, a DaemonSet, or a Deployment (itself the owner of a ReplicaSet), that Pod is under the owner's management.

![](https://raw.githubusercontent.com/hhiroshell/alpaca-notes/master/articles/images/kubernetes-graceful-shutdown-05.dio.svg)

Once `.metadata.deletionTimestamp` is set on the Pod resource, the Pod is removed from the owner resource's management.

#### What happens to a ReplicaSet when a Pod under it is deleted
Let's look at how a ReplicaSet behaves when one of its Pods is deleted.

A ReplicaSet's job is to keep the number of Pods it manages at the configured replica count.
The ReplicaSet controller checks the number of Pods under it on every reconciliation loop, and Pods with `.metadata.deletionTimestamp` set are not counted in that number[^5] [^6].

As a result, the controller determines that the number of Pods is below the ReplicaSet's configured replica count, and creates a new Pod[^7].

![](https://raw.githubusercontent.com/hhiroshell/alpaca-notes/master/articles/images/kubernetes-graceful-shutdown-06.dio.svg)

In other words, once a Pod deletion is triggered, a new Pod is created without waiting for the old container's shutdown to finish.

[^5]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/controller/replicaset/replica_set.go#L685
[^6]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/controller/controller_utils.go#L910-L927
[^7]: https://github.com/kubernetes/kubernetes/blob/v1.18.9/pkg/controller/replicaset/replica_set.go#L696


What to watch out for when terminating Pods safely
---
Laying the whole termination process out on a timeline again, it looks like this:

![](https://raw.githubusercontent.com/hhiroshell/alpaca-notes/master/articles/images/kubernetes-graceful-shutdown-07.dio.svg)

Once `.metadata.deletionTimestamp` is set on the Pod resource, three kinds of processing kick off — and the important part is that they run independently, with no dependency on each other.
Because of this, it's entirely possible for the container to start shutting down before it's actually removed from service, causing some traffic to fail.

Preventing this requires countermeasures like the following.

### Countermeasure 1: sleep in the preStop hook
Sleep for a sufficient amount of time in the preStop hook, so that SIGTERM isn't sent until the Pod has actually been removed from service.
SIGTERM then triggers graceful shutdown, which waits for already-connected connections to finish being handled before stopping the process.

![](https://raw.githubusercontent.com/hhiroshell/alpaca-notes/master/articles/images/kubernetes-graceful-shutdown-08.dio.svg)

### Countermeasure 2: the ultimate graceful shutdown
Trigger graceful shutdown from either the preStop hook or SIGTERM.
The graceful shutdown logic keeps accepting new connections while it waits for every connection to finish being handled, and only then stops the process.

![](https://raw.githubusercontent.com/hhiroshell/alpaca-notes/master/articles/images/kubernetes-graceful-shutdown-09.dio.svg)

### Comparing the two approaches
Countermeasure 1 is relatively easy to implement, but has the downside that setting a generous sleep time slows down Pod termination. That said, as we saw with the ReplicaSet's behavior, the owner resource already treats the Pod as terminated the moment `.metadata.deletionTimestamp` is set, so in practice this may not do much harm.

Countermeasure 2 makes the graceful shutdown logic a bit more complex, but you don't need to worry about tuning a sleep duration — so you could say it shifts the complexity into the application instead.
That said, it seems that, for example, Spring Boot doesn't always support a graceful shutdown that keeps accepting new connections while shutting down[^8]. In cases like that, going with this approach would be a tall order.

[^8]: https://docs.spring.io/spring-boot/docs/2.3.4.RELEASE/reference/htmlsingle/#boot-features-graceful-shutdown


Coming up next
---
This post stuck to a desk-level analysis, but next time I'm going to actually run some experiments!
