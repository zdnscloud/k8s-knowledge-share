# knative Demo

安装knative

knative 依赖于 `Ingress/Gateway` 控制路由请求到`knative`服务 

当前, three options exist which provide this functionality: 
- `Ambassador`, an Envoy-based API Gateway
- `Gloo`, an Envoy-based API Gateway 
- `Istio`, an Envoy-based Service Mesh

### Installing Knative with Ambassador

使用Ambassador进行安装使我们可以选择安装服务网格，以便使用Knative Serving组件路由到应用程序。请注意，Knative Eventing组件需要Istio。

### Installing Knative with Gloo

使用Gloo安装：Gloo充当Knative的轻量级网关。 如果您的群集中不需要服务网格并且希望使用较轻的替代Istio，请选择此选项。 请注意，Gloo目前不支持Knative Eventing组件。

### Installing Knative with Istio

Istio是一种流行的服务网格，包括Knative兼容的入口。 如果要使用Istio服务网格功能，请选择此选项。

### 安装Knative时有几个选项

* 完整安装 – 安装所有Knative组建，最快的安装配置方式

* 部分安装 – 安装 Knative 组建的一个子集

* 自定安装 – 花更多的时间，允许你直接选择组件和监控插件安装


## 在google cloud上安装

### 创建集群

```bash
gcloud auth login

export CLUSTER_NAME=knative
export CLUSTER_ZONE=asia-east1-c
export PROJECT=knative-project-20191906

gcloud projects create $PROJECT --set-as-default

gcloud config set core/project $PROJECT

gcloud services enable \
     cloudapis.googleapis.com \
     container.googleapis.com \
     containerregistry.googleapis.com


gcloud beta container clusters create $CLUSTER_NAME \
  --addons=HorizontalPodAutoscaling,HttpLoadBalancing,Istio \
  --machine-type=n1-standard-2 \
  --cluster-version=latest --zone=$CLUSTER_ZONE \
  --enable-stackdriver-kubernetes --enable-ip-alias \
  --enable-autoscaling --min-nodes=1 --max-nodes=10 \
  --enable-autorepair \
  --scopes cloud-platform
  
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=$(gcloud config get-value core/account)

# Admin permissions are required to create the necessary RBAC rules for Knative.

```

### 安装knative

```bash
kubectl apply --selector knative.dev/crd-install=true \
   --filename https://github.com/knative/serving/releases/download/v0.8.0/serving.yaml \
   --filename https://github.com/knative/eventing/releases/download/v0.8.0/release.yaml \
   --filename https://github.com/knative/serving/releases/download/v0.8.0/monitoring.yaml
   
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.8.0/serving.yaml \
   --filename https://github.com/knative/eventing/releases/download/v0.8.0/release.yaml \
   --filename https://github.com/knative/serving/releases/download/v0.8.0/monitoring.yaml
```

### 部署应用

```yaml
apiVersion: serving.knative.dev/v1alpha1 # Current version of Knative
kind: Service
metadata:
  name: gsmlg-blog # The name of the app
  namespace: default # The namespace the app will use
spec:
  template:
    spec:
      containers:
        - image: docker.io/gsmlg/gsmlg.org:latest # The URL to the image of the app
          env:
            - name: NODE_NAME
              value: "blog-node"
```


获取应用外网IP
```bash
kubectl get svc istio-ingressgateway -n istio-system
```

编辑应用的域名：

```bash
kubectl get cm -n knative-serving config-domain
```

查看应用信息

```bash
kubectl get knative
```

