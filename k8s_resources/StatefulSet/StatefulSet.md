# StatefulSet


StatefulSet是为了解决有状态服务的问题（对应Deployments和ReplicaSets是为无状态服务而设计），其应用场景包括：

- 稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据
- 稳定的网络标志，即Pod重新调度后其PodName和HostName不变
- 有序部署，有序扩展，即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依次进行（即从0到N-1，在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态）
- 有序收缩，有序删除（即从N-1到0）

从上面的应用场景可以发现，StatefulSet由以下几个部分组成：

- 用于定义网络标志（DNS domain）的Headless Service
- 用于创建PersistentVolumes的volumeClaimTemplates
- 定义具体应用的StatefulSet


### 与Deployment的区别
一般来讲，无状态的我们更关注的是群体，任何一个都可以轻易的被其它的所取代，换一个然后再新增一个，和原来一不一样都没关系。只要应用程序一样都ok，因为它没有任何数据和本地的状态，但是对有状态的应用来讲我们必须要把他们当宠物，相当于群体和个体（cattle[畜生]，pet[宠物]）的区别。k8s从1.3开始支持叫PetSet，就叫宠物集，在1.5的时候才改名叫StatefulSet。

### StatefulSet中Pod的域名
StatefulSet中每个Pod的DNS格式为FQDN:`statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local`，其中

- `serviceName`为Headless Service的名字
- `0..N-1`为Pod所在的序号，从0开始到N-1
- `statefulSetName`为StatefulSet的名字
- `namespace`为服务所在的namespace，Headless Servic和StatefulSet必须在相同的namespace
- `.cluster.local`为Cluster Domain


## 限制

* 给定 Pod 的存储必须由PersistentVolume基于所请求的 `storage class` 来提供，或者由管理员预先提供。
* 删除或收缩 StatefulSet 并*不会*删除它关联的存储卷。这样做是为了保证数据安全，它通常比自动清除 StatefulSet 所有相关的资源更有价值。
* StatefulSet 当前需要 [headless 服务](/docs/concepts/services-networking/service/#headless-services) 来负责 Pod 的网络标识。您需要负责创建此服务。
* 当删除 StatefulSets 时，StatefulSet 不提供任何终止 Pod 的保证。为了实现 StatefulSet 中的 Pod 可以有序和优雅的终止，可以在删除之前将 StatefulSet 缩放为 0。
* 在默认 [Pod 管理策略](#pod-management-policies)(`OrderedReady`) 时使用 [滚动更新](#rolling-updates)，可能进入需要 [人工干预](#forced-rollback) 才能修复的损坏状态。

## 使用 StatefulSet

StatefulSet 适用于有以下某个或多个需求的应用：

- 稳定，唯一的网络标志
- 稳定，持久化存储
- 有序，优雅地部署和 scale
- 有序，自动的滚动升级

在上文中，稳定是 Pod（重新）调度中持久性的代名词。 如果应用程序不需要任何稳定的标识符、有序部署、删除和 scale，则应该使用提供一组无状态副本的 controller 来部署应用程序，例如 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment)  可能更适合无状态的需求。


## 组件

下面的示例中描述了 StatefulSet 中的组件。

- 一个名为 nginx 的 headless service，用于控制网络域。
- 一个名为 web 的 StatefulSet，它的 Spec 中指定在有 3 个运行 nginx 容器的 Pod。
- volumeClaimTemplates 使用 lvm 作为稳定存储。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx-headless"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "lvm"
      resources:
        requests:
          storage: 1Gi
```

## Pod 身份

StatefulSet Pod 具有唯一的身份，包括序数，稳定的网络身份和稳定的存储。 身份绑定到 Pod 上，不管它（重新）调度到哪个节点上。

### 有序索引

对于一个有 N 个副本的 StatefulSet，每个副本都会被指定一个整数序数，在 [0,N)之间，且唯一。

### 稳定的网络 ID

StatefulSet 中的每个 Pod 从 StatefulSet 的名称和 Pod 的序数派生其主机名。构造的主机名的模式是`$（statefulset名称)-$(序数)`。 上面的例子将创建三个名为`web-0，web-1，web-2`的 Pod。

StatefulSet 可以使用 [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) 来控制其 Pod 的域。此服务管理的域的格式为：`$(服务名称).$(namespace).svc.cluster.local`，其中 “cluster.local” 是集群域。

在创建每个Pod时，它将获取一个匹配的 DNS 子域，采用以下形式：`$(pod 名称).$(管理服务域)`，其中管理服务由 StatefulSet 上的 `serviceName` 字段定义。

以下是 Cluster Domain，服务名称，StatefulSet 名称以及如何影响 StatefulSet 的 Pod 的 DNS 名称的一些示例。

| Cluster Domain | Service (ns/name) | StatefulSet (ns/name) | StatefulSet Domain              | Pod DNS                                  | Pod Hostname |
| -------------- | ----------------- | --------------------- | ------------------------------- | ---------------------------------------- | ------------ |
| cluster.local  | default/nginx     | default/web           | nginx.default.svc.cluster.local | web-{0..N-1}.nginx.default.svc.cluster.local | web-{0..N-1} |
| cluster.local  | foo/nginx         | foo/web               | nginx.foo.svc.cluster.local     | web-{0..N-1}.nginx.foo.svc.cluster.local | web-{0..N-1} |
| kube.local     | foo/nginx         | foo/web               | nginx.foo.svc.kube.local        | web-{0..N-1}.nginx.foo.svc.kube.local    | web-{0..N-1} |

注意 Cluster Domain 将被设置成 `cluster.local` 除非进行了其他配置。

### 稳定存储

Kubernetes 为每个 VolumeClaimTemplate 创建一个 [PersistentVolume](https://kubernetes.io/docs/concepts/storage/volumes)。上面的 nginx 的例子中，每个 Pod 将具有一个由 `lvm` 存储类创建的 1 GB 存储的 PersistentVolume。当该 Pod （重新）调度到节点上，`volumeMounts` 将挂载与 PersistentVolume Claim 相关联的 PersistentVolume。请注意，与 PersistentVolume Claim 相关联的 PersistentVolume 在 产出 Pod 或 StatefulSet 的时候不会被删除。

## 部署和伸缩

- 对于有 N 个副本的 StatefulSet，Pod 将按照 {0..N-1} 的顺序被创建和部署。
- 当 删除 Pod 的时候，将按照逆序来终结，从{N-1..0}
- 对 Pod 执行 scale 操作之前，它所有的前任必须处于 Running 和 Ready 状态。
- 在终止 Pod 前，它所有的继任者必须处于完全关闭状态。

上面的 nginx 示例创建后，3 个 Pod 将按照如下顺序创建 web-0，web-1，web-2。在 web-0 处于 [运行并就绪](https://kubernetes.io/docs/user-guide/pod-states) 状态之前，web-1 将不会被部署，同样当 web-1 处于运行并就绪状态之前 web-2也不会被部署。如果在 web-1 运行并就绪后，web-2 启动之前， web-0 失败了，web-2 将不会启动，直到 web-0 成功重启并处于运行并就绪状态。

如果用户通过修补 StatefulSet 来 scale 部署的示例，以使 `replicas=1`，则 web-2 将首先被终止。 在 web-2 完全关闭和删除之前，web-1 不会被终止。 如果 web-0 在 web-2 终止并且完全关闭之后，但是在 web-1 终止之前失败，则 web-1 将不会终止，除非 web-0 正在运行并准备就绪。

## Pod 管理策略

在 Kubernetes 1.7 和之后版本，StatefulSet 允许您放开顺序保证，同时通过 `.spec.podManagementPolicy` 字段保证身份的唯一性。

#### 有序的 Pod 管理

StatefulSet 中默认使用的是 `OrderedReady` pod 管理。

#### 并行的 Pod 管理

`Parallel` pod 管理告诉 StatefulSet controller 并行的启动和终止 Pod，在启动和终止其他 Pod 之前不会等待 Pod 变成 运行并就绪或完全终止状态。

## 更新策略

在 Kubernetes 1.7 及以后的版本中，StatefulSet 的 `.spec.updateStrategy` 字段让您可以配置和禁用掉自动滚动更新 Pod 的容器、标签、资源请求／限制、以及注解。

### 删除

`OnDelete` 更新策略实现了遗留（1.6和以前）的行为。 当StatefulSet 的 `.spec.updateStrategy.type` 设置为 `OnDelete` 时，StatefulSet 控制器将不会自动更新 `StatefulSet` 中的 Pod。 用户必须手动删除 Pod 以使控制器创建新的 Pod，以反映对StatefulSet的 `.spec.template` 进行的修改。

### 滚动更新

`RollingUpdate` 更新策略在 StatefulSet 中实现 Pod 的自动滚动更新。当 `spec.updateStrategy` 未指定时，这是默认策略。 当StatefulSet的 `.spec.updateStrategy.type` 设置为 `RollingUpdate` 时，StatefulSet 控制器将在 StatefulSet 中删除并重新创建每个 Pod。 它将以与 Pod 终止相同的顺序进行（从最大的序数到最小的序数），每次更新一个 Pod。 在更新其前身之前，它将等待正在更新的 Pod 状态变成正在运行并就绪。

### 分区

通过声明 `.spec.updateStrategy.rollingUpdate.partition`的方式，`RollingUpdate`更新策略可以实现分区。如果声明了一个分区，当 StatefulSet 的`.spec.template` 被更新时，所有序号大于等于该分区序号的 Pod 都会被更新。所有序号小于该分区序号的 Pod 都不会被更新，并且，即使他们被删除也会依据之前的版本进行重建。如果 StatefulSet 的 `.spec.updateStrategy.rollingUpdate.partition` 大于它的 `.spec.replicas`，对它的 `.spec.template` 的更新将不会传递到它的 Pod。
在大多数情况下，您不需要使用分区，但如果您希望进行阶段更新、执行金丝雀或执行分阶段展开，则这些分区会非常有用。

### 强制回滚 {#forced-rollback}

在默认 [Pod 管理策略](#pod-management-policies)(`OrderedReady`) 时使用 [滚动更新](#rolling-updates) ，可能进入需要人工干预才能修复的损坏状态。

如果更新后 Pod 模板配置进入无法运行或就绪的状态（例如，由于错误的二进制文件或应用程序级配置错误），StatefulSet 将停止回滚并等待。

在这种状态下，仅将 Pod 模板还原为正确的配置是不够的。由于 [已知问题](https://github.com/kubernetes/kubernetes/issues/67250)，
StatefulSet 将继续等待损坏状态的 Pod 准备就绪（永远不会发生），然后再尝试将其恢复为正常工作配置。

恢复模板后，还必须删除 StatefulSet 尝试使用错误的配置来运行的 Pod。这样，StatefulSet 才会开始使用被还原的模板来重新创建 Pod。
