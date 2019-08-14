# Kubernetes 资源配额

当 Kubernetes 调度 Pod 时，容器有足够的资源来实际运行非常重要。 如果在一个资源有限的节点上调度一个大型应用程序，那么这个节点就有可能耗尽内存或 CPU 资源，导致整体服务出现问题.

### 可能的问题

- 应用程序的资源占用可能会超出现有的所有资源.
- 集群上也可能会运行太多太多的副本.
- 错误的配置更改, 导致程序失去控制.

其中有人为的, 也可能有代码的问题, 还可能是运气不好. 

如何来避免类似的问题呢?

加入资源配额.

# Limits和Requests

当分配计算资源时，每个容器可以为cpu或者内存指定一个请求值和一个限度值。可以配置限额值来限制它们中的任何一个值。 如果指定了`requests.cpu` 或者 `requests.memory`的限额值，那么就要求传入的每一个容器显式的指定这些资源的请求。如果指定了`limits.cpu`或者`limits.memory`，那么就要求传入的每一个容器显式的指定这些资源的限度。

`Limits`: 表示容器会被分配到合适的节点。

`Requests`: 限制容器的资源使用, 使系统稳定。

    # 创建namespace
    $ kubectl create namespace myspace
    
    # 创建resourcequota
    $ cat <<EOF > limit-request.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: test-memory-cpu
    spec:
      containers:
      - name: test-memory-cpu
        image: nginx:stable
        resources:
          requests:
            memory: "50Mi"
            cpu: "250m"
          limits:
            memory: "100Mi"
            cpu: "1"
    EOF
    $ kubectl create -f ./limit-request.yaml --namespace=myspace
    
    # 查询resourcequota
    $ kubectl get pod test-memory-cpu --namespace=myspace
    NAME                      READY   STATUS              RESTARTS   AGE
    test-memory-cpu           1/1     Running             0          15s
    
    # 查询resourcequota的详细信息
    $ kubectl describe pod test-memory-cpu --namespace=myspace
    ...
    Limits:
          cpu:     1
          memory:  100Mi
        Requests:
          cpu:        250m
          memory:     50Mi
    ...

# **ResourceQuota**

`ResourceQuota`对象用来定义某个命名空间下所有资源的使用限额，

所有限额如下

    const (
    	// Pods, number
    	ResourcePods ResourceName = "pods"
    	// Services, number
    	ResourceServices ResourceName = "services"
    	// ReplicationControllers, number
    	ResourceReplicationControllers ResourceName = "replicationcontrollers"
    	// ResourceQuotas, number
    	ResourceQuotas ResourceName = "resourcequotas"
    	// ResourceSecrets, number
    	ResourceSecrets ResourceName = "secrets"
    	// ResourceConfigMaps, number
    	ResourceConfigMaps ResourceName = "configmaps"
    	// ResourcePersistentVolumeClaims, number
    	ResourcePersistentVolumeClaims ResourceName = "persistentvolumeclaims"
    	// ResourceServicesNodePorts, number
    	ResourceServicesNodePorts ResourceName = "services.nodeports"
    	// ResourceServicesLoadBalancers, number
    	ResourceServicesLoadBalancers ResourceName = "services.loadbalancers"
    	// CPU request, in cores. (500m = .5 cores)
    	ResourceRequestsCPU ResourceName = "requests.cpu"
    	// Memory request, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
    	ResourceRequestsMemory ResourceName = "requests.memory"
    	// Storage request, in bytes
    	ResourceRequestsStorage ResourceName = "requests.storage"
    	// Local ephemeral storage request, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
    	ResourceRequestsEphemeralStorage ResourceName = "requests.ephemeral-storage"
    	// CPU limit, in cores. (500m = .5 cores)
    	ResourceLimitsCPU ResourceName = "limits.cpu"
    	// Memory limit, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
    	ResourceLimitsMemory ResourceName = "limits.memory"
    	// Local ephemeral storage limit, in bytes. (500Gi = 500GiB = 500 * 1024 * 1024 * 1024)
    	ResourceLimitsEphemeralStorage ResourceName = "limits.ephemeral-storage"
    )

其实包括：

- 计算资源的配额
- 存储资源的配额
- 对象数量的配额

如果集群的总容量小于命名空间的配额总额，可能会产生资源竞争。这时会按照先到先得来处理。 资源竞争和配额的更新都不会影响已经创建好的资源。

## **1. 启动资源配额**

Kubernetes 的众多发行版本默认开启了资源配额的支持。当在apiserver的`--admission-control`配置中添加`ResourceQuota`参数后，便启用了。 当一个命名空间中含有`ResourceQuota`对象时，资源配额将强制执行。

## **2. 计算资源配额**

可以在给定的命名空间中限制可以请求的计算资源（[compute resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)）的总量。

[Untitled](https://www.notion.so/303051cdd8df4176ba00709da467912f)

## **3. 存储资源配额**

可以在给定的命名空间中限制可以请求的存储资源（[storage resources](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)）的总量。

[Untitled](https://www.notion.so/6305cbe1dac740368eec2074bf5e2261)

## **4. 对象数量的配额**

[Untitled](https://www.notion.so/6664ebd963924c5388ac54bc9431a498)

## 5**. 查看和设置配额**

    # 创建namespace
    $ kubectl create namespace myspace
    
    # 创建resourcequota
    $ cat <<EOF > compute-resources.yaml
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: compute-resources
    spec:
      hard:
        pods: "4"
        requests.cpu: "1"
        requests.memory: 1Gi
        limits.cpu: "2"
        limits.memory: 2Gi
    EOF
    $ kubectl create -f ./compute-resources.yaml --namespace=myspace
    
    # 查询resourcequota
    $ kubectl get quota --namespace=myspace
    NAME                    AGE
    compute-resources       30s
    
    # 查询resourcequota的详细信息
    $ kubectl describe quota compute-resources --namespace=myspace
    Name:                  compute-resources
    Namespace:             myspace
    Resource               Used Hard
    --------               ---- ----
    limits.cpu             0    2
    limits.memory          0    2Gi
    pods                   0    4
    requests.cpu           0    1
    requests.memory        0    1Gi

# **LimitRange**

`LimitRange`对象用来定义某个`命名空间`下某种`资源对象`的使用限额，其中资源对象包括：`Pod`、`Container`、`PersistentVolumeClaim`。

定义如下

    const (
    	// Limit that applies to all pods in a namespace
    	LimitTypePod LimitType = "Pod"
    	// Limit that applies to all containers in a namespace
    	LimitTypeContainer LimitType = "Container"
    	// Limit that applies to all persistent volume claims in a namespace
    	LimitTypePersistentVolumeClaim LimitType = "PersistentVolumeClaim"
    )

## **1. 为namespace配置CPU和内存的默认值**

如果在一个拥有默认内存或CPU限额的命名空间中创建一个容器，并且这个容器未指定它自己的内存或CPU的`limit`， 它会被分配这个默认的内存或CPU的`limit`。既没有设置pod的`limit`和`request`才会分配默认的内存或CPU的`request`。

### **1.1. namespace的内存默认值**

    # 创建namespace
    $ kubectl create namespace default-mem-example
    
    # 创建LimitRange
    $ cat memory-defaults.yaml
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: mem-limit-range
    spec:
      limits:
      - default:
          memory: 512Mi
        defaultRequest:
          memory: 256Mi
        type: Container
    
    $ kubectl create -f https://k8s.io/docs/tasks/administer-cluster/memory-defaults.yaml --namespace=default-mem-example
    
    # 创建Pod,未指定内存的limit和request
    $ cat memory-defaults-pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: default-mem-demo
    spec:
      containers:
      - name: default-mem-demo-ctr
        image: nginx
    
    $ kubectl create -f https://k8s.io/docs/tasks/administer-cluster/memory-defaults-pod.yaml --namespace=default-mem-example
    
    # 查看Pod
    $ kubectl get pod default-mem-demo --output=yaml --namespace=default-mem-example
    containers:
    - image: nginx
      imagePullPolicy: Always
      name: default-mem-demo-ctr
      resources:
        limits:
          memory: 512Mi
        requests:
          memory: 256Mi

### **1.2. namespace的CPU默认值**

    # 创建namespace
    $ kubectl create namespace default-cpu-example
    
    # 创建LimitRange
    $ cat cpu-defaults.yaml 
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: cpu-limit-range
    spec:
      limits:
      - default:
          cpu: 1
        defaultRequest:
          cpu: 0.5
        type: Container
    
    $ kubectl create -f https://k8s.io/docs/tasks/administer-cluster/cpu-defaults.yaml --namespace=default-cpu-example    
    
    # 创建Pod，未指定CPU的limit和request
    $ cat cpu-defaults-pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: default-cpu-demo
    spec:
      containers:
      - name: default-cpu-demo-ctr
        image: nginx
    
    $ kubectl create -f https://k8s.io/docs/tasks/administer-cluster/cpu-defaults-pod.yaml --namespace=default-cpu-example
    
    # 查看Pod
    $ kubectl get pod default-cpu-demo --output=yaml --namespace=default-cpu-example
    containers:
    - image: nginx
      imagePullPolicy: Always
      name: default-cpu-demo-ctr
      resources:
        limits:
          cpu: "1"
        requests:
          cpu: 500m

### **1.3 说明**

1. 如果没有指定pod的`request`和`limit`，则创建的pod会使用`LimitRange`对象定义的默认值（request和limit）
2. 如果指定pod的`limit`但未指定`request`，则创建的pod的`request`值会取`limit`的值，而不会取LimitRange对象定义的request默认值。
3. 如果指定pod的`request`但未指定`limit`，则创建的pod的`limit`值会取`LimitRange`对象定义的`limit`默认值。

**默认Limit和request的动机**

如果命名空间具有`资源配额（ResourceQuota）`, 它为内存限额（CPU限额）设置默认值是有意义的。 以下是资源配额对命名空间施加的两个限制：

- 在命名空间运行的每一个容器必须有它自己的内存限额（CPU限额）。
- 在命名空间中所有的容器配置的内存总量（CPU总量）之和不能超出指定的限额。

如果一个容器没有指定它自己的内存限额（CPU限额），它将被赋予默认的限额值，然后它才可以在被配额限制的命名空间中运行。

## **2. 为namespace配置最大最小值**

**创建LimitRange**

    # 创建namespace
    $ kubectl create namespace constraints-mem-example
    
    # 创建LimitRange
    $ cat memory-constraints.yaml
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: mem-min-max-demo-lr
    spec:
      limits:
      - max:
          memory: 1Gi
        min:
          memory: 500Mi
        type: Container
    
    $ kubectl create -f https://k8s.io/docs/tasks/administer-cluster/memory-constraints.yaml --namespace=constraints-mem-example
    
    # 查看LimitRange
    $ kubectl get limitrange cpu-min-max-demo --namespace=constraints-mem-example --output=yaml
    ...
      limits:
      - default:
          memory: 1Gi
        defaultRequest:
          memory: 1Gi
        max:
          memory: 1Gi
        min:
          memory: 500Mi
        type: Container
    ...
    # LimitRange设置了最大最小值，但没有设置默认值，也会被自动设置默认值。

**创建符合要求的Pod**

    # 创建符合要求的Pod
    $ cat memory-constraints-pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: constraints-mem-demo
    spec:
      containers:
      - name: constraints-mem-demo-ctr
        image: nginx
        resources:
          limits:
            memory: "800Mi"
          requests:
            memory: "600Mi"
    
    $ kubectl create -f https://k8s.io/docs/tasks/administer-cluster/memory-constraints-pod.yaml --namespace=constraints-mem-example
    
    # 查看Pod
    $ kubectl get pod constraints-mem-demo --output=yaml --namespace=constraints-mem-example
    ...
    resources:
      limits:
         memory: 800Mi
      requests:
        memory: 600Mi
    ...

**创建超过最大内存limit的pod**

    $ cat memory-constraints-pod-2.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: constraints-mem-demo-2
    spec:
      containers:
      - name: constraints-mem-demo-2-ctr
        image: nginx
        resources:
          limits:
            memory: "1.5Gi"  # 超过最大值 1Gi
          requests:
            memory: "800Mi"
    
    $ kubectl create -f https://k8s.io/docs/tasks/administer-cluster/memory-constraints-pod-2.yaml --namespace=constraints-mem-example
    
    # Pod创建失败，因为容器指定的limit过大
    Error from server (Forbidden): error when creating "docs/tasks/administer-cluster/memory-constraints-pod-2.yaml":
    pods "constraints-mem-demo-2" is forbidden: maximum memory usage per Container is 1Gi, but limit is 1536Mi.

**创建小于最小内存request的Pod**

    $ cat memory-constraints-pod-3.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: constraints-mem-demo-3
    spec:
      containers:
      - name: constraints-mem-demo-3-ctr
        image: nginx
        resources:
          limits:
            memory: "800Mi"
          requests:
            memory: "100Mi"   # 小于最小值500Mi
    
    $ kubectl create -f https://k8s.io/docs/tasks/administer-cluster/memory-constraints-pod-3.yaml --namespace=constraints-mem-example         
    
    # Pod创建失败，因为容器指定的内存request过小
    Error from server (Forbidden): error when creating "docs/tasks/administer-cluster/memory-constraints-pod-3.yaml":
    pods "constraints-mem-demo-3" is forbidden: minimum memory usage per Container is 500Mi, but request is 100Mi.

**创建没有指定任何内存limit和request的pod**

    $ cat memory-constraints-pod-4.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: constraints-mem-demo-4
    spec:
      containers:
      - name: constraints-mem-demo-4-ctr
        image: nginx
    
    $ kubectl create -f https://k8s.io/docs/tasks/administer-cluster/memory-constraints-pod-4.yaml --namespace=constraints-mem-example
    
    # 查看Pod
    $ kubectl get pod constraints-mem-demo-4 --namespace=constraints-mem-example --output=yaml
    ...
    resources:
      limits:
        memory: 1Gi
      requests:
        memory: 1Gi
    ...

容器没有指定自己的 CPU 请求和限制，所以它将从 LimitRange 获取默认的 CPU 请求和限制值。

### **2.3. 说明**

LimitRange 在 namespace 中施加的最小和最大内存（CPU）限制只有在创建和更新 Pod 时才会被应用。改变 LimitRange 不会对之前创建的 Pod 造成影响。

# 参考:

[https://kubernetes.io/docs/concepts/policy/resource-quotas/](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

[https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)

[https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/)

[https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)

[https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)

[https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)

[https://www.huweihuang.com/kubernetes-notes/resource/resource-quota.html](https://www.huweihuang.com/kubernetes-notes/resource/resource-quota.html)

[https://www.huweihuang.com/kubernetes-notes/resource/limit-range.html](https://www.huweihuang.com/kubernetes-notes/resource/limit-range.html)
