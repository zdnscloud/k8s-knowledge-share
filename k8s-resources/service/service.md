## What

>An abstract way to expose an application running on a set of Pods as a network service.

Service 是对一组提供相同功能的 Pods 的抽象，并为它们提供一个统一的入口。借助 Service，应用可以方便的实现服务发现与负载均衡，并实现应用的零宕机升级。Service 通过标签来选取服务后端，一般配合 Replication Controller 或者 Deployment 来保证后端容器的正常运行。这些匹配标签的 Pod IP 和端口列表组成 endpoints，由 kube-proxy 负责将服务 IP 负载均衡到这些 endpoints 上。

## Why

>Kubernetes Pods are mortal. They are born and when they die, they are not resurrected. If you use a Deployment to run your app, it can create and destroy Pods dynamically.
Each Pod gets its own IP address, however in a Deployment, the set of Pods running in one moment in time could be different from the set of Pods running that application a moment later.
This leads to a problem: if some set of Pods (call them “backends”) provides functionality to other Pods (call them “frontends”) inside your cluster, how do the frontends find out and keep track of which IP address to connect to, so that the frontend can use the backend part of the workload?

## How
### Service分类
Service 有五种类型（主用四种）：

- ClusterIP：默认类型，自动分配一个仅 cluster 内部可以访问的虚拟 IP
- NodePort：在 ClusterIP 基础上为 Service 在每台机器上绑定一个端口，这样就可以通过 `<NodeIP>:NodePort` 来访问该服务。如果 kube-proxy 设置了 `--nodeport-addresses=10.240.0.0/16`（v1.10 支持），那么仅该 NodePort 仅对设置在范围内的 IP 有效。
- LoadBalancer：在 NodePort 的基础上，借助 cloud provider 创建一个外部的负载均衡器，并将请求转发到 `<NodeIP>:NodePort`
- ExternalName：将服务通过 DNS CNAME 记录方式转发到指定的域名（通过 `spec.externlName` 设定）。需要 kube-dns 版本在 1.7 以上。
- ExternalIps：将服务直接通#过指定的external ip暴露，外部可直接通过node的external ip进行访问，类似于node-port，但是external ip需管理员自行管理

另外，也可以将已有的服务以 Service 的形式加入到 Kubernetes 集群中来，只需要在创建 Service 的时候不指定 Label selector，而是在 Service 创建好后手动为其添加 endpoint。

### Service 示例

Service 的定义也是通过 yaml 或 json，比如下面定义了一个名为 nginx 的服务，将服务的 80 端口转发到 default namespace 中带有标签 `run=nginx` 的 Pod 的 80 端口

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    run: nginx
  name: nginx
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  sessionAffinity: None
  type: ClusterIP
```

```sh
# service 自动分配了 Cluster IP 10.0.0.108
$ kubectl get service nginx
NAME      CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx     10.0.0.108   <none>        80/TCP    18m
# 自动创建的 endpoint
$ kubectl get endpoints nginx
NAME      ENDPOINTS       AGE
nginx     172.17.0.5:80   18m
# Service 自动关联 endpoint
$ kubectl describe service nginx
Name:			nginx
Namespace:		default
Labels:			run=nginx
Annotations:		<none>
Selector:		run=nginx
Type:			ClusterIP
IP:			10.0.0.108
Port:			<unset>	80/TCP
Endpoints:		172.17.0.5:80
Session Affinity:	None
Events:			<none>
```

当服务需要多个端口时，每个端口都必须设置一个名字

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  - name: https
    protocol: TCP
    port: 443
    targetPort: 9377
```

### 不指定Selectors的服务

在创建 Service 的时候，也可以不指定 Selectors，用来将 service 转发到 kubernetes 集群外部的服务（而不是 Pod）。
* why
    * 希望在生产中拥有外部数据库集群，但在测试环境中，您使用自己的数据库。
    * 希望将服务指向不同命名空间中服务 或在另一个集群上。
    * 正在将工作负载迁移到Kubernetes。在评估方法时，只在Kubernetes中运行一部分后端。
* 目前支持两种方法
（1）自定义 endpoint，即创建同名的 service 和 endpoint，在 endpoint 中设置外部服务的 IP 和端口

```yaml
kind: Service
apiVersion: v1
metadata:
  name: test-service-2
spec:
  ports:
    - protocol: UDP
      port: 5353
      targetPort: 53
---
kind: Endpoints
apiVersion: v1
metadata:
  name: test-service-2
subsets:
  - addresses:
      - ip: 114.114.114.114
    ports:
      - port: 53
```

> Note:
The endpoint IPs must not be: loopback (127.0.0.0/8 for IPv4, ::1/128 for IPv6), or link-local (169.254.0.0/16 and 224.0.0.0/24 for IPv4, fe80::/64 for IPv6).

Endpoint IP addresses cannot be the cluster IPs of other Kubernetes Services, because kube-proxy doesn’t support virtual IPs as a destination.

（2）通过 DNS Cname，在 service 定义中指定 externalName。此时 DNS 服务会给 `<service-name>.<namespace>.svc.cluster.local` 创建一个 CNAME 记录，其值为 `my.database.example.com`。并且，该服务不会自动分配 Cluster IP，需要通过 service 的 DNS 来访问。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: test-service-3
spec:
  type: ExternalName
  externalName: linode.wangyanwei.xyz
```

### Headless Service

* why
Sometimes you don’t need load-balancing and a single Service IP. In this case, you can create what are termed “headless” Services

* how
在创建服务的时候指定 `spec.clusterIP=None`

* kind
    - 指定 Selectors，kubernetes会直接将service域名解析为指向pod ip的A记录
    - 不指定 Selectors，ExternalName类型的服务以及手动指定service endpoints的服务
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx-service
spec:
  clusterIP: None
  ports:
  - name: tcp-80-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 2
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: IfNotPresent
        name: nginx
      dnsPolicy: ClusterFirst
      restartPolicy: Always

```

```sh
# 查询创建的 nginx 服务
$ kubectl get service --all-namespaces=true
NAMESPACE     NAME         CLUSTER-IP      EXTERNAL-IP      PORT(S)         AGE
default       nginx        None            <none>           80/TCP          5m
kube-system   kube-dns     172.26.255.70   <none>           53/UDP,53/TCP   1d
$ kubectl get pod
NAME                       READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-2204978904-6o5dg     1/1       Running   0          14s       172.26.2.5   10.0.0.2
nginx-2204978904-qyilx     1/1       Running   0          14s       172.26.1.5   10.0.0.8
$ dig @172.26.255.70  nginx.default.svc.cluster.local
;; ANSWER SECTION:
nginx.default.svc.cluster.local. 30 IN	A	172.26.1.5
nginx.default.svc.cluster.local. 30 IN	A	172.26.2.5
```
### ExternalIps服务
如果有外部IP路由到一个或多个群集节点，则可以在这些节点上公开Kubernetes服务 externalIPs。在服务端口上使用外部IP（作为目标IP）进入群集的流量将路由到其中一个服务端点。externalIPs不由Kubernetes管理，并且是集群管理员的责任。

在服务规范中，externalIPs可以与任何一个一起指定ServiceTypes。在下面的示例中，my-service客户端可以通过“ 80.11.12.10:80”（externalIP:port）访问“ ”
```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-service-4
spec:
  selector:
    app: hello
  ports:
    - name: http
      protocol: TCP
      port: 9000
      targetPort: 9090
  externalIPs:
    - 192.168.134.101
    - 192.168.134.102
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: hello
  name: hello
spec:
  replicas: 2
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - image: hiwenzi/hello-world-go:latest
        imagePullPolicy: IfNotPresent
        name: hello
      dnsPolicy: ClusterFirst
      restartPolicy: Always
```
### 源IP策略

各种类型的 Service 对源 IP 的处理方法不同：

- ClusterIP Service：使用 iptables 模式，集群内部的源 IP 会保留（不做 SNAT）。如果 client 和 server pod 在同一个 Node 上，那源 IP 就是 client pod 的 IP 地址；如果在不同的 Node 上，源 IP 则取决于网络插件是如何处理的，比如使用 flannel 时，源 IP 是 node flannel IP 地址。
- NodePort Service：默认情况下，源 IP 会做 SNAT，server pod 看到的源 IP 是 Node IP。为了避免这种情况，可以给 service 设置 `spec.ExternalTrafficPolicy=Local` （1.6-1.7 版本设置 Annotation `service.beta.kubernetes.io/external-traffic=OnlyLocal`），让 service 只代理本地 endpoint 的请求（如果没有本地 endpoint 则直接丢包），从而保留源 IP。
>配置之后，如果一个节点相应的port收到外部请求，但是该节点上没有运行这个nodeport类型service的pod，那么不会重定向请求，会直接丢弃
- LoadBalancer Service：默认情况下，源 IP 会做 SNAT，server pod 看到的源 IP 是 Node IP。设置 `service.spec.ExternalTrafficPolicy=Local` 后可以自动从云平台负载均衡器中删除没有本地 endpoint 的 Node，从而保留源 IP。

## 工作原理

kube-proxy 负责将 service 负载均衡到后端 Pod 中，如下图所示

![](images/service-flow.png)

## 支持协议
* TCP：所有类型的Service，也是service默认的网络协议
* UDP：绝大多数Service，对于LoadBalancer类型的Service来说，取决于云服务商
* Proxy：取决于云提供商或ingress conroller
* SCTP：目前是alpha

## 虚拟IP和服务代理
### 为什么不使用DNS轮询？
不时出现的问题是Kubernetes依赖代理将入站流量转发到后端的原因。其他方法呢？例如，是否可以配置具有多个A值（或IPv6的AAAA）的DNS记录，并依赖于循环名称解析？
使用代理服务有几个原因：
* DNS实现的历史很长，不尊重记录TTL，并且在名称查找到期后缓存名称查找的结果。
* 某些应用程序仅执行一次DNS查找并无限期地缓存结果。
* 即使应用程序和库进行了适当的重新解析，DNS记录中的低TTL或零TTL也会对DNS施加高负荷，然后变得难以管理。
### kube-proxy代理模式
* 用户空间代理模式
在此模式下，kube-proxy会监视Kubernetes主服务器以添加和删除Service和Endpoint对象。对于每个服务，它在本地节点上打开一个端口（随机选择）。与此“代理端口”的任何连接都代理到Service的后端Pod之一（通过端点报告）。kube-proxy SessionAffinity在决定使用哪个后端Pod时会考虑服务的设置。

最后，用户空间代理安装iptables规则，捕获流量到服务clusterIP（虚拟）和port。规则将该流量重定向到代理后端Pod的代理端口。

默认情况下，用户空间模式下的kube-proxy通过循环算法选择后端。
* iptables代理模式
在此模式下，kube-proxy监视Kubernetes控制平面以添加和删除Service和Endpoint对象。对于每一个服务，它安装iptables规则，它捕获流量服务的clusterIP和port，而流量重定向到服务的后端集之一。对于每个Endpoint对象，它会安装选择后端Pod的iptables规则。

默认情况下，iptables模式下的kube-proxy随机选择后端。

使用iptables处理流量具有较低的系统开销，因为流量由Linux netfilter处理而无需在用户空间和内核空间之间切换。这种方法也可能更可靠。

如果kube-proxy在iptables模式下运行，并且所选的第一个Pod没有响应，则连接失败。这与用户空间模式不同：在该场景中，kube-proxy将检测到与第一个Pod的连接已失败，并将自动使用不同的后端Pod重试。
* IPVS代理模式
在ipvs模式下，kube-proxy监视Kubernetes服务和端点，调用netlink接口以相应地创建IPVS规则，并定期与Kubernetes服务和端点同步IPVS规则。此控制循环可确保IPVS状态与所需状态匹配。访问服务时，IPVS将流量定向到其中一个后端Pod。
IPVS代理模式基于netfilter钩子函数，类似于iptables模式，但使用哈希表作为底层数据结构并在内核空间中工作。这意味着IPVS模式下的kube-proxy重定向流量的延迟低于iptables模式下的kube-proxy，在同步代理规则时具有更好的性能。与其他代理模式相比，IPVS模式还支持更高的网络流量吞吐量。
IPVS为平衡后端Pod的流量提供了更多选择; 这些是：
    * rr：轮询
    * lc：最少连接（最小数量的打开连接）
    * dh：目标哈希
    * sh：源哈希
    * sed：预计延迟时间最短
    * nq：从不排队
>注意：
要在IPVS模式下运行kube-proxy，必须在启动kube-proxy之前使节点上的IPVS Linux可用。

当kube-proxy在IPVS代理模式下启动时，它会验证IPVS内核模块是否可用。如果未检测到IPVS内核模块，则kube-proxy将回退到在iptables代理模式下运行。


## 集群外部访问服务

Service 的 ClusterIP 是 Kubernetes 内部的虚拟 IP 地址，无法直接从外部直接访问。但如果需要从外部访问这些服务该怎么办呢，有多种方法

* 使用 NodePort 服务在每台机器上绑定一个端口，这样就可以通过 `<NodeIP>:NodePort` 来访问该服务。
* 使用 LoadBalancer 服务借助 Cloud Provider 创建一个外部的负载均衡器，并将请求转发到 `<NodeIP>:NodePort`。该方法仅适用于运行在云平台之中的 Kubernetes 集群。对于物理机部署的集群，可以使用 [MetalLB](https://github.com/google/metallb) 实现类似的功能。
* 使用 Ingress Controller 在 Service 之上创建 L7 负载均衡并对外开放。

## to do
* endpoint controller与service controller的相关逻辑

## 参考资料

- https://kubernetes.io/docs/concepts/services-networking/service/