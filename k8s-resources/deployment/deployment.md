
## Deployments

### 前言

在之前的介绍中：Pod 这个看似复杂的 API 对象，实际上就是对容器的进一步抽象和封装而已。

说得更形象些，“容器”镜像虽然好用，但是容器这样一个“沙盒”的概念，对于描述应用来说，还是太过简单了。这就好比，集装箱固然好用，但是如果它四面都光秃秃的，吊车还怎么把这个集装箱吊起来并摆放好呢？

所以，Pod 对象，其实就是容器的升级版。它对容器进行了组合，添加了更多的属性和字段。这就好比给集装箱四面安装了吊环，使得 Kubernetes 这架“吊车”，可以更轻松地操作它。

Kubernetes 操作这些“集装箱”的逻辑，都由控制器（Controller）完成。而Deployment 这个最基本的控制器对象。

### 简述

先从一个简单的示例开始：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
这个名叫 nginx-deployment 的 Deployment 确保携带了 app=nginx 标签的 Pod 的个数，永远等于 spec.replicas 指定的个数，即 2 个。

这就意味着，如果在这个集群中，携带 app=nginx 标签的 Pod 的个数大于 2 的时候，就会有旧的 Pod 被删除；反之，就会有新的 Pod 被创建。

可以看到，Deployment 这个 template 字段里的内容，跟一个标准的 Pod 对象的 API 定义，丝毫不差。而所有被这个 Deployment 管理的 Pod 实例，其实都是根据这个 template 字段的内容创建出来的。

像 Deployment 定义的 template 字段，在 Kubernetes 项目中有一个专有的名字，叫作 PodTemplate（Pod 模板）。

如下图所示，类似 Deployment 这样的一个控制器，实际上都是由上半部分的控制器定义（包括期望状态），加上下半部分的被控制对象的模板组成的

![nginx-deployment 结构图解](./images/nginx-deployment.png)


### 定义：

官方定义： A Deployment provides declarative updates for Pods and ReplicaSets.

作为Kubernetes的基础资源，它实现了项目中一个非常重要的功能：Pod 的“水平扩展 / 收缩”（horizontal scaling out/in）。

举个例子，如果你更新了 Deployment 的 Pod 模板（比如，修改了容器的镜像），那么 Deployment 就需要遵循一种叫作“滚动更新”（rolling update）的方式，来升级现有的容器。

而这个能力的实现，依赖的是 Kubernetes 项目中的一个非常重要的概念（API 对象）：ReplicaSet。

#### ReplicaSet

可以通过一个 YAML 文件查看一下ReplicaSet 的结构：

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-set
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
```
一个 ReplicaSet 对象，其实就是由副本数目的定义和一个 Pod 模板组成的。
不难发现，它的定义其实是 Deployment 的一个子集。

更重要的是，Deployment 控制器实际操纵的，正是这样的 ReplicaSet 对象，而不是 Pod 对象。

明白了这个原理，再来分析一个如下所示的 Deployment：
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

通过这张图，我们就很清楚的看到，一个定义了 replicas=3 的 Deployment，与它的 ReplicaSet，以及 Pod 的关系，实际上是一种“层层控制”的关系。

![ Deployment，与它的 ReplicaSet，以及 Pod 的关系](./images/deploy-rs-pod-1.png)

因此 Deployment Controller 只需要修改它所控制的 ReplicaSet 的 Pod 副本个数就可以实现 “水平扩展 / 收缩”的功能。

### 示例

水平扩展： 将副本数改为4:

创建deployment：
```
$ kubectl apply -f  nginx-deployment.yaml
```


查看deployments：

```
$ kubectl get deployments

NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           4m13s
```

> 参数解析：

 * READY：当前处于 Running 状态的 Pod 的个数；

 * UP-TO-DATE：当前处于最新版本的 Pod 的个数，所谓最新版本指的是 Pod 的 Spec 部分与 Deployment 里 Pod 模板里定义的完全一致；

 * AVAILABLE：当前已经可用的 Pod 的个数，即：既是 Running 状态，又是最新版本，并且已经处于 Ready（健康检查正确）状态的 Pod 的个数。

查看nginx-deployment的详情：
```
$ kubectl describe deployment  nginx-deployment
```

执行 kubectl scale命令：

```
$ kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled
```

查看deployments：

```
$ kubectl get deployments

NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   4/4     4            4           5m5s
```

删除 nginx-deployment：
```
$ k delete deployments nginx-deployment
deployment.extensions "nginx-deployment" deleted
```


### ”滚动更新“

还以这个 Deployment 为例，简述“滚动更新”的过程：

在创建的时候，额外加了一个–record 参数。它的作用，是记录下你每次操作所执行的命令，以方便后面查看： 

```
$ kubectl create -f nginx-deployment.yaml --record
```

查看 nginx-deployment 创建后的状态信息: 
```
$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           7s
```

查看一下这个 Deployment 所控制的 ReplicaSet：
```
$ kubectl get rs

NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-76bf4969df   3         3         3       8m3s
```
> 参数解析：

* DESIRED：用户期望的 Pod 副本个数（spec.replicas 的值）；

* CURRENT：当前处于 Running 状态的 Pod 的个数；

如上所示，在用户提交了一个 Deployment 对象后，Deployment Controller 就会立即创建一个 Pod 副本个数为 3 的 ReplicaSet。

这个 ReplicaSet 的名字，则是由 Deployment 的名字和一个随机字符串共同组成。这个随机字符串叫作 pod-template-hash，在我们这个例子里就是：76bf4969df。

ReplicaSet 会把这个随机字符串加在它所控制的所有 Pod 的标签里，从而保证这些 Pod 不会与集群里的其他 Pod 混淆。

查看下pods：
```
$ kubectl get pods

NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-76bf4969df-b8ctw   1/1     Running   0          11m
nginx-deployment-76bf4969df-l8hlw   1/1     Running   0          11m
nginx-deployment-76bf4969df-m94q8   1/1     Running   0          11m

```

查看具体的一个pod：
```
$  kubectl describe pod nginx-deployment-76bf4969df-b8ctw

Name:               nginx-deployment-76bf4969df-b8ctw
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               worker/202.173.9.70
Start Time:         Thu, 19 Dec 2019 19:23:03 +0800
Labels:             app=nginx
                    pod-template-hash=76bf4969df
Annotations:        <none>
...

```

现在修改pod模板的，则”滚动更新“ 就会自动触发：

```
$ vim nginx-deployment.yaml

... 
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1 # 1.7.9 -> 1.9.1
        ports:
        - containerPort: 80
...

```

更新：
```
$  kubectl apply -f  nginx-deployment.yaml
```

通过查看 Deployment 的 Events，看到这个“滚动更新”的流程：
```
& kubectl describe deployment nginx-deployment
...
Events:
  Type    Reason             Age                From                    Message
  ----    ------             ----               ----                    -------
  Normal  ScalingReplicaSet  45m                deployment-controller   Scaled up replica set nginx-deployment-76bf4969df to 3
  Normal  ScalingReplicaSet  18m                deployment-controller   Scaled up replica set nginx-deployment-779fcd779f to 1
  Normal  ScalingReplicaSet  17m                deployment-controller   Scaled down replica set nginx-deployment-76bf4969df to 2
  Normal  ScalingReplicaSet  17m                deployment-controller   Scaled up replica set nginx-deployment-779fcd779f to 2
  Normal  ScalingReplicaSet  17m                deployment-controller   Scaled down replica set nginx-deployment-76bf4969df to 1
  Normal  ScalingReplicaSet  17m                deployment-controller   Scaled up replica set nginx-deployment-779fcd779f to 3
  Normal  ScalingReplicaSet  17m                deployment-controller   Scaled down replica set nginx-deployment-76bf4969df to 0
  Normal  ScalingReplicaSet  10m                deployment-controller   Scaled up replica set nginx-deployment-76bf4969df to 1
  Normal  ScalingReplicaSet  10m                deployment-controller   Scaled down replica set nginx-deployment-779fcd779f to 2
  Normal  ScalingReplicaSet  10m                deployment-controller   Scaled up replica set nginx-deployment-76bf4969df to 2
  Normal  InjectionSkipped   10m (x9 over 45m)  linkerd-proxy-injector  Linkerd sidecar proxy injection skipped: neither the namespace nor the pod have the annotation "linkerd.io/inject:enabled"
  Normal  ScalingReplicaSet  10m (x3 over 10m)  deployment-controller   (combined from similar events): Scaled down replica set nginx-deployment-779fcd779f to 0
...
```

可以看到，首先，当你修改了 Deployment 里的 Pod 定义之后，Deployment Controller 会使用这个修改后的 Pod 模板，创建一个新的 ReplicaSet（hash=779fcd779f），这个新的 ReplicaSet 的初始 Pod 副本数是：0。

如此交替进行，新 ReplicaSet 管理的 Pod 副本数，从 0 个变成 1 个，再变成 2 个，最后变成 3 个。而旧的 ReplicaSet 管理的 Pod 副本数则从 3 个变成 2 个，再变成 1 个，最后变成 0 个。这样，就完成了这一组 Pod 的版本升级过程。像这样，将一个集群中正在运行的多个 Pod 版本，交替地逐一升级的过程，就是“滚动更新”。

在这个“滚动更新”过程完成之后，你可以查看一下新、旧两个 ReplicaSet 的最终状态：
```
$ kubectl get rs

NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-76bf4969df   0         0         0       29m
nginx-deployment-779fcd779f   3         3         3       2m29s
```

![滚动跟新时候三者的关系图](./images/deploy-rs-pod-2.png)

如果在更改的过程中出错了，可以支持回滚到某个版本：

首先，需要使用 kubectl rollout history 命令，查看每次 Deployment 变更对应的版本。而由于在创建这个 Deployment 的时候，指定了–record 参数，所以我们创建这些版本时执行的 kubectl 命令，都会被记录下来。

这个操作的输出如下所示：

```
$ kubectl rollout history deployment/nginx-deployment

deployment.extensions/nginx-deployment
REVISION  CHANGE-CAUSE
2         kubectl create --filename=nginx-deployment.yaml --record=true
3         kubectl create --filename=nginx-deployment.yaml --record=true
```

通过这个 kubectl rollout history 指令，看到每个版本对应的 Deployment 的 API 对象的细节:
```
$ kubectl rollout history deployment/nginx-deployment --revision=2
```

```
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment.extensions/nginx-deployment rolled back
```

在查看一下ReplicaSet的状态：

```
$ kubectl get rs

NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-76bf4969df   2         2         1       35m
nginx-deployment-779fcd779f   2         2         2       8m17s
```

### 总结

Deployment 实际上是一个两层控制器。首先，它通过 ReplicaSet 的个数来描述应用的版本；然后，它再通过 ReplicaSet 的属性（比如 replicas 的值），来保证 Pod 的副本数量