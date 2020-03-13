## DaemonSet

#### DaemonSet 的主要作用

是让你在 Kubernetes 集群里，运行一个 Daemon Pod。
 
DaemonSet确保所有(或部分)节点都运行一个Pod的副本。随着节点被添加到集群中，pod也被添加到集群中。当节点从集群中移除时，这些pods将被垃圾收集。删除DaemonSet将清理它创建的pods。

Pod 有如下三个特征：
* 这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上；
* 每个节点上只有一个这样的 Pod 实例；
* 当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。

Daemon Pod 的意义确实是非常重要的,跟其他k8s资源不一样，DaemonSet 开始运行的时机，很多时候比整个 Kubernetes 集群出现的时机都要早。

一些典型用法：

* 运行的集群存储守护程序，诸如glusterd，ceph在每个节点上。
* 在每个节点（例如fluentd或）上运行日志收集守护程序filebeat。
* 如监测守护程序普罗米修斯节点导出，Sysdig代理等。


为了弄清楚 DaemonSet 的工作原理，先从它的 API 对象的定义说起：

### DaemonSet 定义
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

#### 一些基本的定义

*  Required Fields

 apiVersion，kind和metadata。
 
 DaemonSet对象的名称，必须是有效的 DNS子域名。
 
 .spec部分。
 
* Pod Template
 
 `.spec.template` 是一个pod模板。它具有与Pod完全相同的架构，只是它是嵌套的并且没有`apiVersion `or `kind`。
 
 必须指定适当的标签。
 
 Pod模板的 RestartPolicy 默认为`Always`。

* Pod Selector
   `.spec.selector`指定了，则必须与匹配`.spec.template.metadata.labels`。这些配置不匹配的配置将被API拒绝。

因此这个 DaemonSet，管理的是一个 fluentd-elasticsearch 镜像的 Pod。通过 fluentd 将 Docker 容器里的日志转发到 ElasticSearch 中。

可以看到，DaemonSet 跟 Deployment 其实非常相似，只不过是没有 replicas 字段；它也使用 selector 选择管理所有携带了 name=fluentd-elasticsearch 标签的 Pod。

而这些 Pod 的模板，也是用 template 字段定义的。在这个字段中，我们定义了一个使用 fluentd-elasticsearch:1.20 镜像的容器，而且这个容器挂载了两个 hostPath 类型的 Volume，分别对应宿主机的 /var/log 目录和 /var/lib/docker/containers 目录。

在 DaemonSet 上，加上 resources 字段，来限制它的 CPU 和内存使用，防止它占用过多的宿主机资源。

fluentd 启动之后，它会从这两个目录里搜集日志信息，并转发给 ElasticSearch 保存。通过 ElasticSearch 就可以很方便地检索这些日志了。

#### 那么，DaemonSet 又是如何保证每个 Node 上有且只有一个被管理的 Pod 呢？

DaemonSet Controller，首先从 Etcd 里获取所有的 Node 列表，然后遍历所有的 Node，检查当前这个 Node 上是不是有一个携带了 name=fluentd-elasticsearch 标签的 Pod 在运行。

而检查的结果，可能有这么三种情况：没有这种 Pod，那么就意味着要在这个 Node 上创建这样一个 Pod；有这种 Pod，但是数量大于 1，那就说明要把多余的 Pod 从这个 Node 上删除掉；正好只有一个这种 Pod，那说明这个节点是正常的。

其中，删除节点（Node）上多余的 Pod 非常简单，直接调用 Kubernetes API 就可以了。

默认情况下，DaemonSet控制器将在所有节点上创建Pod。但是，如何在指定的 Node 上创建新 Pod 呢？

 * `.spec.template.spec.nodeSelector`，选择 Node 的名字即可。
 ```
 ...
 nodeSelector:
     name: <Node名字>
 ```
 
* 还有一个新的、功能更完善的字段可以代替它，即：nodeAffinity,示例：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: metadata.name
            operator: In
            values:
            - node-geektime
```

在这个 Pod 里，声明了一个 spec.affinity 字段，然后定义了一个 nodeAffinity。其中，spec.affinity 字段，是 Pod 里跟调度相关的一个字段,DaemonSet控制器将在与该节点相似性匹配的节点上创建Pod 。

nodeAffinity 的含义是：
* requiredDuringSchedulingIgnoredDuringExecution：它的意思是说，这个 nodeAffinity 必须在每次调度的时候予以考虑。同时，这也意味着你可以设置在某些情况下不考虑这个 nodeAffinity；

* 这个 Pod，将来只允许运行在“metadata.name”是“node-geektime”的节点上。

* operator: In  部分匹配；如果定义 operator: Equal，就是完全匹配
> 因此可以看出 nodeAffinity 是可以取代 nodeSelector 。


所以， DaemonSet Controller 会在创建 Pod 的时候，自动在这个 Pod 的 API 对象里，加上这样一个 nodeAffinity 定义。其中，需要绑定的节点名字，正是当前正在遍历的这个 Node。

当然，DaemonSet 并不需要修改用户提交的 YAML 文件里的 Pod 模板，而是在向 Kubernetes 发起请求之前，直接修改根据模板生成的 Pod 对象。

此外，pod中还有一个字段叫做tolerations，这个字段意味着这个 Pod，会“容忍”（Toleration）某些 Node 的“污点”（Taint）。

tolerations 字段，格式如下所示：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
```

这个 Toleration 的含义是：“容忍”所有被标记为 unschedulable“污点”的 Node；“容忍”的效果是允许调度。
> 可以简单地把“污点”理解为一种特殊的 Label。

而在正常情况下，被标记了 unschedulable“污点”的 Node，是不会有任何 Pod 被调度上去的（effect: NoSchedule）。可是，DaemonSet 自动地给被管理的 Pod 加上了这个特殊的 Toleration，就使得这些 Pod 可以忽略这个限制，继而保证每个节点上都会被调度一个 Pod。当然，如果这个节点有故障的话，这个 Pod 可能会启动失败，而 DaemonSet 则会始终尝试下去，直到 Pod 启动成功。

这种机制，正是在部署 Kubernetes 集群的时候，能够先部署 Kubernetes 本身、再部署其他插件的根本原因。

至此，通过上面这些内容，能够明白，DaemonSet 其实是一个非常简单的控制器。在它的控制循环中，只需要遍历所有节点，然后根据节点上是否有被管理 Pod 的情况，来决定是否要创建或者删除一个 Pod。

只不过，在创建每个 Pod 的时候，DaemonSet 会自动给这个 Pod 加上一个 nodeAffinity，从而保证这个 Pod 只会在指定节点上启动。同时，它还会自动给这个 Pod 加上一个 Toleration，从而忽略节点的 unschedulable“污点”。

当然，也可以在 Pod 模板里加上更多种类的 Toleration，从而利用 DaemonSet 实现自己的目的。例如上述例子：

```
tolerations:
- key: node-role.kubernetes.io/master
  effect: NoSchedule
```

这是因为在默认情况下，Kubernetes 集群不允许用户在 Master 节点部署 Pod。因为，Master 节点默认携带了一个叫作node-role.kubernetes.io/master的“污点”。所以，为了能在 Master 节点上部署 DaemonSet 的 Pod，就必须让这个 Pod“容忍”这个“污点”。

在理解了 DaemonSet 的工作原理之后，接下来通过一个具体的实践来掌握 DaemonSet 的使用方法。

首先，创建这个 DaemonSet 对象：

```
$ kubectl create -f fluentd-elasticsearch.yaml
```

而创建成功后，你就能看到，如果有 N 个节点，就会有 N 个 fluentd-elasticsearch Pod 在运行。比如在我们的例子里，会有一个 Pod，如下所示:

```
$ kubectl get pod -n kube-system -l name=fluentd-elasticsearch
NAME                          READY     STATUS    RESTARTS   AGE
fluentd-elasticsearch-8nqm2   1/1       Running   0          53m
```

通过 kubectl get 查看一下 Kubernetes 集群里的 DaemonSet 对象：

```
$ kubectl get ds -n kube-system fluentd-elasticsearch
NAME                    DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   1         1        1         1            1           <none>          1h
```

> 备注：Kubernetes 里比较长的 API 对象都有短名字，比如 DaemonSet 对应的是 ds，Deployment 对应的是 deploy。

就会发现 DaemonSet 和 Deployment 一样，也有 DESIRED、CURRENT 等多个状态字段。这也就意味着，DaemonSet 可以像 Deployment 那样，进行版本管理。这个版本，可以使用 kubectl rollout history 看到：

```

$ kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonsets "fluentd-elasticsearch"
REVISION  CHANGE-CAUSE
1         <none>
```

接下来，我们来把这个 DaemonSet 的容器镜像版本到 v2.2.0：

```
$ kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record -n=kube-system
```

这个 kubectl set image 命令里，第一个 fluentd-elasticsearch 是 DaemonSet 的名字，第二个 fluentd-elasticsearch 是容器的名字。

这时候，可以使用 kubectl rollout status 命令看到这个“滚动更新”的过程，如下所示：

```
$ kubectl rollout status ds/fluentd-elasticsearch -n kube-system
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 0 out of 1 new pods have been updated...
daemon set "fluentd-elasticsearch" successfully rolled out
```

由于这一次我在升级命令后面加上了–record 参数，所以这次升级使用到的指令就会自动出现在 DaemonSet 的 rollout history 里面，如下所示：

```
$ kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonsets "fluentd-elasticsearch"
REVISION  CHANGE-CAUSE
1         <none>
2          kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record=true --namespace=kube-system
```

有了版本号，也就可以像 Deployment 一样，将 DaemonSet 回滚到某个指定的历史版本了。

而在前面的文章中讲解 Deployment 对象的时候，曾经提到过，Deployment 管理这些版本，靠的是“一个版本对应一个 ReplicaSet 对象”。可是，DaemonSet 控制器操作的直接就是 Pod，不可能有 ReplicaSet 这样的对象参与其中。那么，它的这些版本又是如何维护的呢？

Kubernetes 有一个 API 对象，名叫 ControllerRevision，专门用来记录某种 Controller 对象的版本。可以通过如下命令查看 fluentd-elasticsearch 对应的 ControllerRevision：

```
$ kubectl get controllerrevision -n kube-system -l name=fluentd-elasticsearch
NAME                               CONTROLLER                             REVISION   AGE
fluentd-elasticsearch-5467956799   daemonset.apps/fluentd-elasticsearch   1          1h
fluentd-elasticsearch-569dc54fc6   daemonset.apps/fluentd-elasticsearch   2          1h
```

使用 kubectl describe 查看这个 ControllerRevision 对象：

```
$  kubectl describe controllerrevision fluentd-elasticsearch-5467956799 -n kube-system
Name:         fluentd-elasticsearch-5467956799
Namespace:    kube-system
Labels:       controller-revision-hash=5467956799
              name=fluentd-elasticsearch
Annotations:  deprecated.daemonset.template.generation: 1
API Version:  apps/v1
Data:
  Spec:
    Template:
      $ Patch:  replace
      Metadata:
        Creation Timestamp:  <nil>
        Labels:
          Name:  fluentd-elasticsearch
      Spec:
        Containers:
          Image:              k8s.gcr.io/fluentd-elasticsearch:1.20
          Image Pull Policy:  IfNotPresent
          Name:               fluentd-elasticsearch
		  ....
```

就会看到，这个 ControllerRevision 对象，实际上是在 Data 字段保存了该版本对应的完整的 DaemonSet 的 API 对象。并且，在 Annotation 字段保存了创建这个对象所使用的 kubectl 命令。

可以尝试将这个 DaemonSet 回滚到 Revision=1 时的状态：

```
$ kubectl rollout undo daemonset fluentd-elasticsearch --to-revision=1 -n kube-system
daemonset.extensions/fluentd-elasticsearch rolled back
```

这个 kubectl rollout undo 操作，实际上相当于读取到了 Revision=1 的 ControllerRevision 对象保存的 Data 字段。而这个 Data 字段里保存的信息，就是 Revision=1 时这个 DaemonSet 的完整 API 对象。

所以，现在 DaemonSet Controller 就可以使用这个历史 API 对象，对现有的 DaemonSet 做一次 PATCH 操作（等价于执行一次 kubectl apply -f “旧的 DaemonSet 对象”），从而把这个 DaemonSet“更新”到一个旧版本。

这也是为什么，在执行完这次回滚完成后，你会发现，DaemonSet 的 Revision 并不会从 Revision=2 退回到 1，而是会增加成 Revision=3。这是因为，一个新的 ControllerRevision 被创建了出来。

### 总结

相比于 Deployment，DaemonSet 只管理 Pod 对象，然后通过 nodeAffinity 和 Toleration 这两个调度器的小功能，保证了每个节点上有且只有一个 Pod。

与此同时，DaemonSet 使用 ControllerRevision，来保存和管理对应的“版本”。
