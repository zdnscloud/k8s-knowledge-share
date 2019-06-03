Kubernetes之Garbage Collection

ReplicationController、ReplicaSet、StatefulSet、DaemonSet 和 Deployment

一个 ReplicaSet 是一组 Pod 的 Owner。 具有 Owner 的对象被称为是 Owner 的 Dependent

当创建一个 ReplicaSet 时，Kubernetes 自动设置 ReplicaSet 中每个 Pod 的 ownerReference 字段值

Kubernetes 会自动为某些对象设置 ownerReference 的值，这些对象是由 ReplicationController、ReplicaSet、StatefulSet、DaemonSet 和 Deployment 所创建或管理。  

```
apiVersion: extensions/v1beta1
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
kubectl create -f https://k8s.io/docs/concepts/controllers/my-repset.yaml
kubectl get pods --output=yaml  
```

输出显示了 Pod 的 Owner 是名为 my-repset 的 ReplicaSet:
```
apiVersion: v1
kind: Pod
metadata:
  ...
  ownerReferences:
  - apiVersion: extensions/v1beta1
    controller: true
    blockOwnerDeletion: true
    kind: ReplicaSet
    name: my-repset
    uid: d9607e19-f88f-11e6-a518-42010a800195
  ...  
```

当删除ReplicaSet时，可以指定是否级联删除。Kubernetes 中有两种 级联删除 的模式：background 模式和 foreground 模式。
如果删除对象时，不自动删除它的 Dependent，这些 Dependent 被称作是原对象的 孤儿

Background 级联删除

在 background 级联删除 模式下，Kubernetes 会立即删除 Owner 对象，然后垃圾收集器会在后台删除这些 Dependent

Foreground 级联删除
在 foreground 级联删除 模式下，根对象首先进入 "deletion in progress" 状态。在"deletion in progress" 状态会有如下的情况：

1、对象仍然可以通过 REST API 可见。
2、会设置对象的 deletionTimestamp 字段。
3、对象的 metadata.finalizers 字段包含了值 “foregroundDeletion”。
4、一旦对象被设置为  "deletion in progress"状态，垃圾收集器会删除对象的所有 Dependent。 垃圾收集器在删除了所有 “Blocking” 状态的 Dependent（对象的 ownerReference.blockOwnerDeletion=true）之后，它会删除 Owner 对象。

注意，在 “foreground 删除” 模式下，只有设置了 ownerReference.blockOwnerDeletion 值的 Dependent 才能阻止删除 Owner 对象。 在 Kubernetes 1.7 版本中将增加许可控制器（Admission Controller），基于 Owner 对象上的删除权限来控制用户去设置 blockOwnerDeletion 的值为 true，所以未授权的 Dependent 不能够延迟 Owner 对象的删除。

如果一个对象的 ownerReferences 字段被一个 Controller（例如 Deployment 或 ReplicaSet）设置，blockOwnerDeletion 会被自动设置，不需要手动修改这个字段。

设置级联删除策略

通过为 Owner 对象设置 deleteOptions.propagationPolicy 字段，可以控制级联删除策略。 可能的取值包括：“orphan”、“Foreground” 或 “Background”。

对很多 Controller 资源，包括 ReplicationController、ReplicaSet、StatefulSet、DaemonSet 和 Deployment，默认的垃圾收集策略是 orphan。 因此，除非指定其它的垃圾收集策略，否则所有 Dependent 对象使用的都是 orphan 策略

下面是一个在后台删除 Dependent 对象的例子
```
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/extensions/v1beta1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
-H "Content-Type: application/json"
```

下面是一个在前台删除 Dependent 对象的例子
```
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/extensions/v1beta1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
-H "Content-Type: application/json"
```

下面是一个孤儿 Dependent 的例子
```
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/extensions/v1beta1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
-H "Content-Type: application/json"  
```

kubectl 也支持级联删除。 通过设置 --cascade 为 true，可以使用 kubectl 自动删除 Dependent 对象。 设置 --cascade 为 false，会使 Dependent 对象成为孤儿 Dependent 对象。 --cascade 的默认值是 true。

下面是一个例子，使一个 ReplicaSet 的 Dependent 对象成为孤儿 Dependent
`kubectl delete replicaset my-repset --cascade=false`

