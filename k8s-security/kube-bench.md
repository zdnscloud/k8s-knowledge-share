![img](./kube-bench.png)

​		容器安全厂商Aquq以CIS（ Center for Internet Security）推出的K8s Benchmark作为基础，开源出了一套安全检测工具Kube-bench。它是一个Go应用程序，它通过运行CIS Kubernetes Benchmark中记录的标准来检查Kubernetes是否已安全部署。

​		请注意，使用kube-bench无法检查托管群集的master，例如GKE，EKS和AKS，虽然可以在这些环境中使用kube-bench检查工作节点的配置，但是它无法访问master节点，

​		测试是使用YAML文件配置的，因此随着测试规范的发展，该工具易于更新。

## CIS Kubernetes标准支持

kube-bench支持分别在CIS标准1.3.0至1.5.0中定义的Kubernetes测试。

| CIS Kubernetes Benchmark | kube-bench config | Kubernetes版本 |
| ------------------------ | ----------------- | -------------- |
| 1.3.0                    | cis-1.3           | 1.11-1.12      |
| 1.4.1                    | cis-1.4           | 1.13-1.14      |
| 1.5.0                    | cis-1.5           | 1.15-          |

默认情况下，kube-bench将根据运行的Kubernetes版本确定要运行的测试集。

## 安装

您可以选择

- 从容器内部运行kube-bench（与主机共享PID名称空间）
- 在主机上运行容器安装kube-bench，然后直接在主机上运行kube-bench
- 从[发布页面](https://github.com/aquasecurity/kube-bench/releases)安装最新的二进制文件，
- 从源代码编译它。

## 运行kube-bench

如果您直接从命令行运行kube-bench，则可能需要root / sudo才能访问所有配置文件。

kube-bench的`controls`根据检测到的节点类型和集群正在运行的Kubernetes版本自动选择要使用的版本。通过在命令行上指定`master`或`node`子命令和 `--version`标志，可以覆盖此行为。

Kubernetes版本也可以使用`KUBE_BENCH_VERSION`环境变量进行设置。参数`--version`优先于环境变量`KUBE_BENCH_VERSION`。

例如，运行kube-bench对master版本自动检测：

```
kube-bench master
```

或者指定工作节点为版本1.13进行测试：

```
kube-bench node --version 1.13
```

`kube-bench`将按如上映射表所示，获取k8s版本映射到相应的CIS Benchmark版本。例如，如果指定--version 1.13`其会映射到CIS Benchmark version `cis-1.14`。

或者，您可以指定`--benchmark`运行特定的CIS Benchmark版本：

```
kube-bench node --benchmark cis-1.4
```

如果要针对特定的CIS标准测试目标（例如，主，节点，etcd等），则可以使用`run --targets`子命令。

```
kube-bench --benchmark cis-1.4 run --targets master,node
```

或者

```
kube-bench --benchmark cis-1.5 run --targets master,node,etcd,policies
```

下表显示了基于CIS Benchmark版本的有效范围。

| 独联体基准 | 目标                                       |
| ---------- | ------------------------------------------ |
| 顺式1.3    | master, node                               |
| 顺式1.4    | master, node                               |
| 顺式1.5    | master, controlplane, node, etcd, policies |

如果未指定目标，`kube-bench`则将根据CIS Benchmark版本确定适当的目标。

### 在容器内运行

```
docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t aquasec/kube-bench:latest [master|node] --version 1.13
```

> 注意：为了自动检测Kubernetes版本，测试在路径中需要kubelet或kubectl二进制文件。您可以通过`-v $(which kubectl):/usr/bin/kubectl`解决此问题。您还需要传入kubeconfig凭据。例如：

```
docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -v $(which kubectl):/usr/bin/kubectl -v ~/.kube:/.kube -e KUBECONFIG=/.kube/config -t aquasec/kube-bench:latest [master|node] 
```

您可以使用自己的配置，方法是将其挂载在默认配置中 `/opt/kube-bench/cfg/`

```
docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t -v path/to/my-config.yaml:/opt/kube-bench/cfg/config.yam -v $(which kubectl):/usr/bin/kubectl -v ~/.kube:/.kube -e KUBECONFIG=/.kube/config aquasec/kube-bench:latest [master|node]
```

### 在Kubernetes集群中运行

您可以在Pod内运行kube-bench，但它需要访问主机的PID名称空间才能检查正在运行的进程，以及访问主机上存储配置文件和其他文件的某些目录。

kube-bench会自动检测主节点，并在可能时运行主节点检查。通过验证配置文件中定义的master必需组件是否正在运行来完成检测。

`job.yaml`可以将提供的文件作为job运行测试。例如：

```
$ kubectl apply -f job.yaml
job.batch/kube-bench created

$ kubectl get pods
NAME                      READY   STATUS              RESTARTS   AGE
kube-bench-j76s9   0/1     ContainerCreating   0          3s

# Wait for a few seconds for the job to complete
$ kubectl get pods
NAME                      READY   STATUS      RESTARTS   AGE
kube-bench-j76s9   0/1     Completed   0          11s

# The results are held in the pod's logs
kubectl logs kube-bench-j76s9
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 API Server
...
```

只检查master节点：使用`job-master.yaml`

只检查worker节点：使用`job-node.yaml`。

### 从容器安装

此命令将Docker容器中的kube-bench二进制文件和配置文件复制到您的主机中：**仅针对linux-x86-64编译的二进制文件（因此它们将无法在macOS或Windows上运行）**

```
docker run --rm -v `pwd`:/host aquasec/kube-bench:latest install
```

然后可以运行`./kube-bench [master|node]`。

### 从源安装

克隆后运行：

```
go get github.com/aquasecurity/kube-bench
go get github.com/golang/dep/cmd/dep
cd $GOPATH/src/github.com/aquasecurity/kube-bench
$GOPATH/bin/dep ensure -vendor-only
go build -o kube-bench .

# See all supported options
./kube-bench --help

# Run all checks
./kube-bench
```

## 输出

有三种输出状态：

- [PASS]和[FAIL]表示测试已成功运行，并且通过或失败。
- [WARN]表示此测试需要进一步关注，例如，它是需要手动运行的测试。
- [INFO]提示信息输出。

注意：

- 如果测试为“手动”，则始终会生成“警告”（因为用户必须手动运行它）
- 如果测试为“Scored”，而kube-bench无法运行测试，则会生成“失败”（因为该测试尚未通过，并且作为“Scored”测试，则必须视为失败）。
- 如果测试为“Not Scored”，而kube-bench无法运行测试，则会生成警告。
- 如果测试为“Scored”，类型为空，并且没有出现`test_items`，则会生成警告。

## 配置

Kubernetes的配置以及二进制文件的位置和名称因安装而异，因此可以在`cfg/config.yaml`文件中进行配置。

特定于版本的配置文件中的任何设置都`cfg/<version>/config.yaml`优先于主`cfg/config.yaml`文件中的设置。

您可以`kube-bench`在我们的[文档中](https://github.com/aquasecurity/kube-bench/blob/master/docs/README.md#configuration-and-variables)阅读有关配置的更多信息。

## 测试配置YAML表现

测试（或“controls”）表示为YAML文档（默认情况下安装到中`./cfg`）。这些测试YAML文件有不同版本，反映了CIS Kubernetes Benchmark的不同版本。您可以在我们的[文档中](https://github.com/aquasecurity/kube-bench/blob/master/docs/README.md)找到有关测试文件YAML定义的更多信息。

### 省略检查

如果您确定建议不适合您的环境，则可以选择通过编辑测试YAML文件为它提供检查类型来忽略它，`skip`如以下示例所示：

```
  checks:
  - id: 2.1.1
    text: "Ensure that the --allow-privileged argument is set to false (Scored)"
    type: "skip"
    scored: true
```

该检查将不会运行任何测试，并且输出将标记为[INFO]。

## 路线图

展望未来，我们计划发布对kube-bench的更新，以增加对Benchmark新版本的支持，从而可以预期每个Kubernetes新版本都会推出。
