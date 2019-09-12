# Pod健康检测

当你使用kubernetes的时候，有没有遇到过Pod在启动后一会就挂掉然后又重新启动这样的恶性循环？你有没有想过kubernetes是如何检测pod是否还存活？虽然容器已经启动，但是kubernetes如何知道容器的进程是否准备好对外提供服务了呢？

## Probe探针类型

**1. livenessProbe(存活探针)**
![livenesspic][1]
- 表明容器是否正在运行。
- 如果存活探测失败，则 kubelet 会杀死容器，并且容器将受到其 `重启策略`的影响。
- 如果容器不提供存活探针，则默认状态为 `Success`。

**2. readinessProbe(就绪探针)**
![readnesspic][2]
- 表明容器是否可以正常接受请求。
- 如果就绪探测失败，端点控制器将从与 Pod 匹配的所有 Service 的端点中删除该 Pod 的 IP 地址。
- 初始延迟之前的就绪状态默认为 `Failure`。
- 如果容器不提供就绪探针，则默认状态为 `Success`。

## Handler

`探针`是`kubelet`对容器执行定期的诊断，主要通过调用容器配置的三类`Handler`实现：

**Handler的类型**：

- `ExecAction`：在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
- `TCPSocketAction`：对指定端口上的容器的 IP 地址进行 TCP 检查。如果端口打开，则诊断被认为是成功的。
- `HTTPGetAction`：对指定的端口和路径上的容器的 IP 地址执行 HTTP Get 请求。如果响应的状态码大于等于200 且小于 400，则诊断被认为是成功的。

**探测结果**为以下三种之一：

- `成功`：容器通过了诊断。
- `失败`：容器未通过诊断。
- `未知`：诊断失败，因此不会采取任何行动。

## 探针使用方式

- 如果容器异常可以自动崩溃，则不一定要使用探针，可以由Pod的`restartPolicy`执行重启操作。
- `存活探针`适用于希望容器探测失败后被杀死并重新启动，需要指定`restartPolicy` 为 Always 或 OnFailure。
- `就绪探针`适用于希望Pod在不能正常接收流量的时候被剔除，并且在就绪探针探测成功后才接收流量。

## 定义 liveness 命令

许多长时间运行的应用程序最终会转换到broken状态，除非重新启动，否则无法恢复。Kubernetes提供了liveness probe来检测和补救这种情况。

在本次操作将基于 `googlecontainer/busybox`镜像创建运行一个容器的Pod。以下是Pod的配置文件`exec-liveness.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    image: googlecontainer/busybox
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

该配置文件给Pod配置了一个容器。`periodSeconds` 规定kubelet要每隔5秒执行一次liveness probe。  `initialDelaySeconds` 告诉kubelet在第一次执行probe之前要的等待5秒钟。探针检测命令是在容器中执行 `cat /tmp/healthy` 命令。如果命令执行成功，将返回0，kubelet就会认为该容器是活着的并且很健康。如果返回非0值，kubelet就会杀掉这个容器并重启它。

容器启动时，执行该命令：

```bash
/bin/sh -c "touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600"
```

在容器生命的最初30秒内有一个 `/tmp/healthy` 文件，在这30秒内 `cat /tmp/healthy`命令会返回一个成功的返回码。30秒后， `cat /tmp/healthy` 将返回失败的返回码。

创建Pod：

```bash
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/exec-liveness.yaml
```

在30秒内，查看Pod的event：

```
kubectl describe pod liveness-exec
```

结果显示没有失败的liveness probe：

```bash
FirstSeen    LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
24s       24s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "googlecontainer/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "googlecontainer/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
```

启动35秒后，再次查看pod的event：

```bash
kubectl describe pod liveness-exec
```

在最下面有一条信息显示liveness probe失败，容器被删掉并重新创建。

```bash
FirstSeen LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
37s       37s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "googlecontainer/busybox"
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "googlecontainer/busybox"
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
2s        2s      1   {kubelet worker0}   spec.containers{liveness}   Warning     Unhealthy   Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```

再等30秒，确认容器已经重启：

```bash
kubectl get pod liveness-exec
```

从输出结果来`RESTARTS`值加1了。

```bash
NAME            READY     STATUS    RESTARTS   AGE
liveness-exec   1/1       Running   1          1m
```

## 定义一个liveness HTTP请求

我们还可以使用HTTP GET请求作为liveness probe。下面是一个基于`googlecontainer/liveness`镜像运行了一个容器的Pod的例子`http-liveness.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    args:
    - /server
    image: googlecontainer/liveness
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
          - name: X-Custom-Header
            value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

该配置文件只定义了一个容器，`livenessProbe` 指定kubelet需要每隔3秒执行一次liveness probe。`initialDelaySeconds` 指定kubelet在该执行第一次探测之前需要等待3秒钟。该探针将向容器中的server的8080端口发送一个HTTP GET请求。如果server的`/healthz`路径的handler返回一个成功的返回码，kubelet就会认定该容器是活着的并且很健康。如果返回失败的返回码，kubelet将杀掉该容器并重启它。

任何大于等于200小于400的返回码都会认定是成功的返回码。其他返回码都会被认为是失败的返回码。

最开始的10秒该容器是活着的， `/healthz` handler返回200的状态码。这之后将返回500的返回码。

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```

容器启动3秒后，kubelet开始执行健康检查。第一次健康监测会成功，但是10秒后，健康检查将失败，kubelet将杀掉和重启容器。

创建一个Pod来测试一下HTTP liveness检测：

```bash
kubectl create -f https://k8s.io/docs/tasks/configure-pod-container/http-liveness.yaml
```

10秒后，查看Pod的event，确认liveness probe失败并重启了容器。

```bash
kubectl describe pod liveness-http
```

## 定义TCP liveness探针

第三种liveness probe使用TCP Socket。 使用此配置，kubelet将尝试在指定端口上打开容器的套接字。 如果可以建立连接，容器被认为是健康的，如果不能就认为是失败的。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: googlecontainer/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

如您所见，TCP检查的配置与HTTP检查非常相似。 此示例同时使用了readiness和liveness probe。 容器启动后5秒钟，kubelet将发送第一个readiness probe。 这将尝试连接到端口8080上的goproxy容器。如果探测成功，则该pod将被标记为就绪。Kubelet将每隔10秒钟执行一次该检查。

除了readiness probe之外，该配置还包括liveness probe。 容器启动15秒后，kubelet将运行第一个liveness probe。 就像readiness probe一样，这将尝试连接到goproxy容器上的8080端口。如果liveness probe失败，容器将重新启动。

创建一个pod来检测tcp liveness
```bash
kubectl apply -f https://k8s.io/examples/pods/probe/tcp-liveness-readiness.yaml
```

启动15秒后，再次查看pod的event：

```bash
kubectl describe pod goproxy
```

## 使用命名的端口(named port)

可以使用命名的ContainerPort作为HTTP或TCP liveness检查：

```yaml
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
  path: /healthz
  port: liveness-port
```

## 定义readiness探针

有时，应用程序暂时无法对外部流量提供服务。 例如，应用程序可能需要在启动期间加载大量数据或配置文件。 在这种情况下，你不想杀死应用程序，但你也不想发送请求。 Kubernetes提供了readiness probe来检测和减轻这些情况。 Pod中的容器可以报告自己还没有准备，不能处理Kubernetes服务发送过来的流量。

Readiness probe的配置跟liveness probe很像。唯一的不同是使用 `readinessProbe `而不是`livenessProbe`。

```yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

Readiness probe的HTTP和TCP的探测器配置跟liveness probe一样。

Readiness和livenss probe可以并行用于同一容器。 使用两者可以确保流量无法到达未准备好的容器，并且容器在失败时重新启动。

## 配置Probe

Probe 中有很多精确和详细的配置，通过它们你能准确的控制liveness和readiness检查：

- `initialDelaySeconds`：容器启动后第一次执行探测是需要等待多少秒。
- `periodSeconds`：执行探测的频率。默认是10秒，最小1秒。
- `timeoutSeconds`：探测超时时间。默认1秒，最小1秒。
- `successThreshold`：探测失败后，最少连续探测成功多少次才被认定为成功。默认是1。对于liveness必须是1。最小值是1。 
- `failureThreshold`：探测成功后，最少连续探测失败多少次才被认定为失败。默认是3。最小值是1。

HTTP probe 中可以给 `httpGet`设置其他配置项：

- `host`：连接的主机名，默认连接到pod的IP。
- `scheme`：连接使用的schema，默认HTTP。
- `path`: 访问的HTTP server的path。
- `httpHeaders`：自定义请求的header。HTTP运行重复的header。
- `port`：访问的容器的端口名字或者端口号。端口号必须介于1和65535之间。

对于HTTP探测器，kubelet向指定的路径和端口发送HTTP请求以执行检查。 Kubelet将probe发送到POD的IP地址，除非地址被`httpGet`中的可选`host`字段覆盖。

## LivenessProbe与ReadinessProbe区别

LivenessProbe探针： 用于判断容器是否存活，即Pod是否为running状态，如果LivenessProbe探针探测到容器不健康，则kubelet将kill掉容器，并根据容器的重启策略是否重启，如果一个容器不包含LivenessProbe探针，则Kubelet认为容器的LivenessProbe探针的返回值永远成功。 
ReadinessProbe探针： 用于判断容器是否启动完成，即容器的Ready是否为True，可以接收请求，如果ReadinessProbe探测失败，则容器的Ready将为False，控制器将此Pod的Endpoint从对应的service的Endpoint列表中移除，从此不再将任何请求调度此Pod上，直到下次探测成功。

## 一些使用建议

1. 建议对服务同时设置readiness和liveness的健康检查
2. 通过TCP对端口检查形式（TCPSocketAction），仅适用于端口已关闭或进程停止情况。因为即使服务异常，只要端口是打开状态，健康检查仍然是通过的。
3. 基于第二点，一般建议用ExecAction自定义健康检查逻辑，或采用HTTP Get请求进行检查（HTTPGetAction）。
4. 无论采用哪种类型的探针，一般建议设置检查服务（readiness）的时间短于检查Container（liveness）的时间，也可以将检查服务（readiness）的探针与Container（liveness）的探针设置为一致。目的是故障服务先下线，如果过一段时间还无法自动恢复，那么根据重启策略，重启该container、或其他机器重新创建一个pod恢复故障服务。

### 参考
https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#define-a-liveness-command
https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes
http://dockone.io/article/2587

  [1]: https://pic1.zhimg.com/v2-05ec3fb33fa8412a4ba60a24ea510f6c_b.webp
  [2]: https://pic3.zhimg.com/v2-7dd35978297b4c44a916dac271929872_b.webp