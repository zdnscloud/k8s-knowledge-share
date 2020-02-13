# Ingress
## Ingress是什么
Ingress将集群外部的HTTP和HTTPS路由暴露给集群中的 服务。流量路由由Ingress资源上定义的规则控制。
```
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```
可以将Ingress配置为为服务提供外部可访问的URL，负载平衡流量，终止SSL / TLS以及提供基于名称的虚拟主机。一个入口控制器负责履行入口，通常有一个负载均衡器，虽然它也可以配置您的边缘路由器或额外的前端，以帮助处理流量。
Ingress不会暴露任意端口或协议。将除HTTP和HTTPS之外的服务暴露给互联网通常使用Service.Type = NodePort或 Service.Type = LoadBalancer类型的服务。
## 先决条件
必须有一个入口控制器来满足Ingress。仅创建Ingress资源无效。
您可能需要部署Ingress控制器，例如ingress-nginx。您可以从许多 Ingress控制器中进行选择。
理想情况下，所有Ingress控制器都应符合参考规范。实际上，各种Ingress控制器的运行方式略有不同。
## Ingress资源定义
一个最小的Ingress资源示例：
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        backend:
          serviceName: test
          servicePort: 80
```
与所有其他Kubernetes资源，入口需求apiVersion，kind以及metadata各个领域。有关使用配置文件的一般信息，请参阅部署应用程序，配置容器，管理资源。Ingress经常使用注释来配置一些选项，具体取决于Ingress控制器，其中一个例子是重写目标注释。不同的Ingress控制器支持不同的注释。查看您选择的Ingress控制器的文档，了解支持哪些注释。

Ingress 规范 具有配置负载平衡器或代理服务器所需的所有信息。最重要的是，它包含与所有传入请求匹配的规则列表。Ingress资源仅支持指导HTTP流量的规则。

### 入口规则
每个HTTP规则都包含以下信息：

* 可选主机。在此示例中，未指定主机，因此该规则适用于通过指定的IP地址的所有入站HTTP流量。如果提供了主机（例如，foo.bar.com），则规则适用于该主机。
* 路径列表（例如/testpath），每个路径都有一个用serviceName 和定义的关联后端servicePort。在负载均衡器将流量定向到引用的服务之前，主机和路径都必须与传入请求的内容匹配。
* 后端是服务文档中描述的服务和端口名称的组合 。向Ingress发出的与主机和规则路径匹配的HTTP（和HTTPS）请求将发送到列出的后端。
默认后端通常在Ingress控制器中配置，以便为与规范中的路径不匹配的任何请求提供服务。

### 默认后端
没有规则的Ingress将所有流量发送到单个默认后端。默认后端通常是Ingress控制器的配置选项，并且未在Ingress资源中指定。
如果没有任何主机或路径与Ingress对象中的HTTP请求匹配，则流量将路由到您的默认后端。
## Ingress类型
### Single Service Ingress
现有的Kubernetes概念允许您公开单个服务（请参阅替代方案）。您也可以通过指定没有规则的默认后端来使用Ingress执行此操作 。
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
```
### Simple fanout
扇出配置根据请求的HTTP URI将流量从单个IP地址路由到多个服务。Ingress允许您将负载平衡器的数量降至最低。例如，设置如下：
```
foo.bar.com -> 178.91.123.132 -> / foo    service1:4200
                                 / bar    service2:8080
```
需要一个Ingress，例如：
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: service1
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: service2
          servicePort: 8080
```
使用以下命令创建Ingress时kubectl apply -f：
```
kubectl describe ingress simple-fanout-example
Name:             simple-fanout-example
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:4200 (10.8.0.90:4200)
               /bar   service2:8080 (10.8.0.91:8080)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     22s                loadbalancer-controller  default/test
```
只要服务（service1，service2）存在，Ingress控制器就会提供满足Ingress的特定于实现的负载均衡器。完成后，您可以在地址字段中查看负载均衡器的地址。
### Name based virtual hosting
基于名称的虚拟主机支持将HTTP流量路由到同一IP地址的多个主机名。
```
foo.bar.com --|                 |-> foo.bar.com service1:80
              | 178.91.123.132  |
bar.foo.com --|                 |-> bar.foo.com service2:80
```
以下Ingress告诉后备负载均衡器根据主机头来路由请求。
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
```
如果创建没有在规则中定义任何主机的Ingress资源，则可以匹配任何到Ingress控制器的IP地址的Web流量，而不需要基于名称的虚拟主机。
例如，下面的入口资源将流量路由请求first.bar.com到service1，second.foo.com到service2，任何流量而不在请求中定义的主机名的IP地址（即，如果没有请求头被呈现）给service3。
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: first.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: second.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
  - http:
      paths:
      - backend:
          serviceName: service3
          servicePort: 80
```
### TLS
您可以通过指定Secret来保护Ingress  包含TLS私钥和证书。目前，Ingress仅支持单个TLS端口443，并假定TLS终止。如果Ingress中的TLS配置部分指定了不同的主机，则它们将根据通过SNI TLS扩展指定的主机名在同一端口上进行多路复用（前提是Ingress控制器支持SNI）。TLS机密必须包含已命名的密钥tls.crt，tls.key并且包含用于TLS的证书和私钥。例如：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```
在Ingress中引用此秘密告诉Ingress控制器使用TLS将通道从客户端保护到负载均衡器。您需要确保您创建的TLS秘密来自包含CN的证书sslexample.foo.com。
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
    - sslexample.foo.com
    secretName: testsecret-tls
  rules:
    - host: sslexample.foo.com
      http:
        paths:
        - path: /
          backend:
            serviceName: service1
            servicePort: 80
```
## 负载均衡
Ingress控制器通过一些负载均衡策略设置进行自举，该策略设置适用于所有Ingress，例如负载平衡算法，后端权重方案等。尚未通过Ingress公开更高级的负载平衡概念（例如，持久会话，动态权重）。您可以通过用于服务的负载平衡器来获取这些功能。
## Ingress Controller
为了使 Ingress 资源正常工作，集群必须有 Ingress 控制器运行。 这不同于其他类型的控制器，它们通常作为 kube-controller-manager 二进制文件的一部分运行，并且通常作为集群创建的一部分自动启动。
k8s自身是没有实现的，需要自行创建或实现，一般常用的是nginx-ingress-controller（也是我们目前使用的ingress-controller）
### 一些可选的controller
* Ambassador API Gateway is an Envoy based ingress controller with community or commercial support from Datawire.
* AppsCode Inc. offers support and maintenance for the most widely used HAProxy based ingress controller Voyager.
* Contour is an Envoy based ingress controller provided and supported by Heptio.
* Citrix provides an Ingress Controller for its hardware (MPX), virtualized (VPX) and free containerized (CPX) ADC for baremetal and cloud deployments.
* F5 Networks provides support and maintenance for the F5 BIG-IP Controller for Kubernetes.
* Gloo is an open-source ingress controller based on Envoy which offers API Gateway functionality with enterprise support from solo.io.
* HAProxy Technologies offers support and maintenance for the HAProxy Ingress Controller for Kubernetes. See the official documentation.
* Istio based ingress controller Control Ingress Traffic.
* Kong offers community or commercial support and maintenance for the Kong Ingress Controller for Kubernetes.
* NGINX, Inc. offers support and maintenance for the NGINX Ingress Controller for Kubernetes.
* Skipper HTTP router and reverse proxy for service composition, including use cases like Kubernetes Ingress, designed as a library to build your custom proxy
* Traefik is a fully featured ingress controller (Let’s Encrypt, secrets, http2, websocket), and it also comes with commercial support by Containous.
### 使用多个Ingress Controller
您可以在群集中部署任意数量的入口控制器。创建入口时，应使用适当的注释对每个入口进行注释， ingress.class 以指示在群集中存在多个入口控制器时应使用哪个入口控制器。
如果您没有定义类，则云提供商可能会使用默认的入口控制器。
理想情况下，所有入口控制器都应满足此规范，但各种入口控制器的运行方式略有不同。
## 思考
* ingress虽然是namespace的子资源，但是域名及http路由是全局的，即一个namespace的用户所创建ingress有可能会对其他namespace的ingress产生干扰（破坏了ns的隔离作用），ingress-nginx-controller对于冲突ingress资源的行为是先入为主
contour的ingressRoute crd可以实现多ns间的域名和路由隔离（一个域名只能出现一次，多ns间可进行委派）
* backend只支持单一service，实际上backend多service需求很常见，例如应用蓝绿发布等，这也是istio等一些crd的优势所在

