# prow

![prow](https://github.com/kubernetes/test-infra/blob/master/prow/logo_horizontal_solid.png)

Prow 是基于Kubernetes开发的CI/CD系统

Jobs可以由多种类型的事件出发，并且报告状态给不同的服务。
除了Job执行外，Prow还提供了Github自动化执行策略，`/foo`格式的命令的chat-ops和自动PR合并


### Functions and Features

* 用于测试，批处理和产品发布的Job运行
* 基于`/foo`格式的可扩展Github bot命令，强化配置策略和进程
* 带有批量测试的Github自动合并
* 用于查看Job，合并队列状态，动态生成的帮助信息的前端界面
* 基于SCM的自动部署
* 在SCM中自动管理Github的org/repo
* 专为拥有大量仓库的多组织设计（Prow只需要一个Github bot token）
* 在Kubernetes上运行带来的高可用性
* JSON结构日志
* Prometheus metrics


### Who use Prow

Prow is used by the following organizations and projects:
- [Kubernetes](https://prow.k8s.io)
  - This includes [kubernetes](https://github.com/kubernetes), [kubernetes-client](https://github.com/kubernetes-client), [kubernetes-csi](https://github.com/kubernetes-csi), [kubernetes-incubator](https://github.com/kubernetes-incubator), and [kubernetes-sigs](https://github.com/kubernetes-sigs).
- [OpenShift](https://prow.svc.ci.openshift.org/)
  - This includes [openshift](https://github.com/openshift), [openshift-s2i](https://github.com/openshift-s2i), [operator-framework](https://github.com/operator-framework), and some repos in [kubernetes-incubator](https://github.com/kubernetes-incubator), [containers](https://github.com/containers) and [heketi](https://github.com/heketi).
- [Istio](https://prow.istio.io/)
- [Knative](https://prow.knative.dev/)
- [Jetstack](https://prow.build-infra.jetstack.net/)
- [Kyma](https://status.build.kyma-project.io/)
- [Metal³](https://prow.apps.ci.metal3.io/)
- [Prometheus](http://prombench.prometheus.io/)
- [Caicloud](https://github.com/caicloud)
- [Kubeflow](https://github.com/kubeflow)
- [Azure AKS Engine](https://github.com/Azure/aks-engine/tree/master/.prowci)
- [tensorflow/minigo](https://github.com/tensorflow/minigo#automated-tests)
- [helm/charts](https://github.com/helm/charts)
- [Daisy(google compute image tools)](https://github.com/GoogleCloudPlatform/compute-image-tools/tree/master/test-infra#prow-and-gubenator)
- [KubeEdge (Kubernetes Native Edge Computing Framework)](https://github.com/kubeedge/kubeedge)
- [Volcano (Kubernetes Native Batch System)](https://github.com/volcano-sh/volcano)
- [Loodse](https://public-prow.loodse.com/)

[Jenkins X](https://jenkins-x.io/) uses [Prow as part of Serverless Jenkins](https://medium.com/@jdrawlings/serverless-jenkins-with-jenkins-x-9134cbfe6870).


### 部署Prow

#### 创建Github bot账号

配置账户的 `personal access token`

* Must have the `public_repo` and `repo:status` scopes
* Add the `repo` scope if you plan on handing private repos
* Add the `admin_org:hook` scope if you plan on handling a github org

##### 创建Github secrets

1. 创建 `hmac-token` 用于Github webhooks 的认证

```bash
# openssl rand -hex 20 > /path/to/hook/secret
kubectl create secret generic hmac-token --from-file=hmac=/path/to/hook/secret
```

2. 创建Github OAuth2 token

```bash
# https://github.com/settings/tokens
kubectl create secret generic oauth-token --from-file=oauth=/path/to/oauth/secret
```

#### 安装`prow`到集群

```
kubectl apply -f https://github.com/gsmlg/pipeline/raw/master/updated_prow.yaml
```

默认会安装到default namesapce下，Job运行在test-pods namsapces下

通过命令查看是否安装完成

```
# kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
deck               2/2     2            2           21h
hook               2/2     2            2           21h
horologium         1/1     1            1           21h
plank              1/1     1            1           21h
sinker             1/1     1            1           21h
statusreconciler   1/1     1            1           21h
tide               1/1     1            1           21h
```

配置ingress

```
# 查看ingress
kubectl get ingress ing

# 编辑ingress
kubectl edit ingress ing

```

#### 创建webhook

配置ingress，default/ing

设置好ingress域名

打开github repo的setting页面设置webook，URL设置为ingress-domain/hook, secret为webook创建的secret


这样一个prow集群配置完成


### 添加 plugins

增加configmap plugins

内容为：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: plugins
  namespace: default
data:
  plugins.yaml: |
    plugins:
      ORG/PROJECT:
      - size
```

会自动在pull-request上添加一个size标签


