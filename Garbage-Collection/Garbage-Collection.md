# GC之使用方式 #

kubernetes层面的对象:pod、ReplicationController、ReplicaSet、StatefulSet、DaemonSet和Deployment等是如何进行垃圾回收的呢？

按理说以上的kubernetes对象并不会产生垃圾，对象一直都存在，除非用户或者某种控制器将对象删除。这里称为"Garbage Collection"不太准确，实际上它想讲的是在执行删除操作时如何控制对象之间的依赖关系。比如，Deployment对象创建ReplicaSet对象，ReplicaSet对象又创建pod对象，那么在删除Deployment时，是否需要删除与之相依赖的ReplicaSet与pod呢？

## 定义 ##
一个 ReplicaSet 是一组 Pod 的 Owner。 具有 Owner 的对象被称为是 Owner 的 Dependent  
当创建一个 ReplicaSet 时，Kubernetes 自动设置 ReplicaSet 中每个 Pod 的 ownerReference 字段值  

Kubernetes 会自动为某些对象设置 ownerReference 的值，这些对象是由 ReplicationController、ReplicaSet、StatefulSet、DaemonSet 和 Deployment 所创建或管理  

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-repset
spec:
  replicas: 3
  selector:
    matchLabels:
      pod-is-for: garbage-collection-example
  template:
    metadata:
      labels:
        pod-is-for: garbage-collection-example
    spec:
      containers:
      - name: nginx
        image: nginx
```

```
kubectl create -f replicaset.yaml
kubectl get pods -o yaml  
```

输出显示了 Pod 的 Owner 是名为 my-repset 的 ReplicaSet:
```
apiVersion: v1
kind: Pod
metadata:
  ...
  ownerReferences:
  - apiVersion: apps/v1
    controller: true
    blockOwnerDeletion: true
    kind: ReplicaSet
    name: my-repset
    uid: d9607e19-f88f-11e6-a518-42010a800195
  ...
```
# 如何控制垃圾回收? #
当删除ReplicaSet时，可以指定是否级联删除。Kubernetes 中有两种 级联删除 的模式：background 模式和 foreground 模式。
如果删除对象时，不自动删除它的 Dependent，这些 Dependent 被称作是原对象的孤儿。

## Background 级联删除 ##

在 background 级联删除模式下，Kubernetes 会立即删除 Owner 对象，然后垃圾收集器会在后台删除这些 Dependent。  

## Foreground 级联删除 ##
在 foreground 级联删除模式下，根对象首先进入"deletion in progress"状态。在"deletion in progress" 状态会有如下的情况：  
1.对象仍然可以通过 REST API 可见。  
2.会设置对象的 deletionTimestamp 字段。  
3.对象的 metadata.finalizers 字段包含了值 “foregroundDeletion”。  
4.一旦对象被设置为"deletion in progress"状态，垃圾收集器会删除对象的所有 Dependent。 垃圾收集器在删除了所有 “Blocking” 状态的 Dependent（对象的 ownerReference.blockOwnerDeletion=true）之后，它会删除 Owner 对象。  

注意，在 “foreground 删除” 模式下，只有设置了 ownerReference.blockOwnerDeletion 值的 Dependent 才能阻止删除 Owner 对象。 在 Kubernetes 1.7 版本中将增加许可控制器（Admission Controller），基于 Owner 对象上的删除权限来控制用户去设置 blockOwnerDeletion 的值为 true，所以未授权的 Dependent 不能够延迟 Owner 对象的删除。

如果一个对象的 ownerReferences 字段被一个 Controller（例如 Deployment 或 ReplicaSet）设置，blockOwnerDeletion 会被自动设置，不需要手动修改这个字段。

# 设置级联删除策略 #

通过为 Owner 对象设置 deleteOptions.propagationPolicy 字段，可以控制级联删除策略。 可能的取值包括：“Orphan”、“Foreground” 或 “Background”。

对很多 Controller 资源，包括 ReplicationController、ReplicaSet、StatefulSet、DaemonSet 和 Deployment，默认的垃圾收集策略是 orphan（注意本段中说的默认值指REST API的默认行为，不是kubectl命令）。 因此，除非指定其它的垃圾收集策略，否则所有 Dependent 对象使用的都是 orphan 策略。并且当apiVersion是extensions/v1beta1, apps/v1beta1, and apps/v1beta2时，除非特别指定，默认删除策略仍然是"Orphan"。在kubernetes1.9版本中，所有类型的对象，在app/v1版本的apiVersion中，默认从对象删除。

后台删除Dependent对象的例子  
```
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
-H "Content-Type: application/json"
```
前台删除Dependent对象的示例  
```
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
-H "Content-Type: application/json"
```
不删除从对象示例  
```
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
-H "Content-Type: application/json"
```

kubectl命令也支持级联删除操作。在执行kubectl命令时指定"--cascade"选项，其值为true时删除从对象，为false不删除从对象，默认值为true且为删除模式为Background。 示例如下：  
`kubectl delete replicaset my-repset --cascade=false`

# 级联删除Deployment时的注意点 #
当级联删除Deployment时，其删除请求中的propagationPolicy必需设定成Foreground或Background。否则Deployment创建的ReplicaSet会被级联删除，但是ReplicaSet创建的pod不会被级联删除，会成为孤儿。   
```
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/deployments/emoji \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
-H "Content-Type: application/json"
```  
```
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/deployments/emoji \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
-H "Content-Type: application/json"
```  
```
curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/deployments/emoji \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
-H "Content-Type: application/json"
```  


