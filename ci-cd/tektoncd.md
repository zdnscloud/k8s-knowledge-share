# Tekton Pipelines

## Intro

Tekton Pipelines是一个为`Kubernetes`应用程序配置和运行`CI / CD`风格的`Pipelined`的开源实现

`Pipeline` 创建 `Custom Resources` 作为构建模块来声明`pipelines`

Tekton Pipelines 是云原生的

- 运行于`Kubernetes`
- 将`Kubernetes`集群作为一级资源类型
- 使用容器作为构建块

Tekton Pipelines 是解耦的

- Pipeline 可以被部署于任意k8s集群
- 组成`pipeline`的`task`可以分开独立运行
- 向Git repos之类的资源可以轻松的在运行之间交换

Tekton Pipelines are Typed

- 类型化的资源意味着对于诸如Image之类的资源，可以轻松地将资源输出

### 此设计的高级细节：

- Pipeline 运行管道，可以实现一个流程，可以由事件出发，也可以通过`PipelineRun`来运行
- Task 基本运行单元，可以通过`TaskRun`来运行
- PipelineResource Pipeline的输入和输出资源


## 安装

运行 kubectl 安装指定的yaml文件

```shell
kubectl apply -f https://raw.githubusercontent.com/gsmlg/pipeline/master/updated.yaml
```

检查所有pod都处于`running`状态时，安装完成

```shell
kubectl -n tekton-pipelines get pods
```

安装dashboard，更方便的查看pipeline

```shell
kubectl apply -f https://raw.githubusercontent.com/gsmlg/pipeline/master/updated_dashboard.yaml
```


## 演示运行一个`singlecloud`的构建过程

创建账户

```yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pipeline-run-role
rules:
- apiGroups:
  - extensions
  resources:
  - deployments
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pipeline-run-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pipeline-run-role
subjects:
- kind: ServiceAccount
  name: pipeline-run-service
  namespace: default

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-run-service
  namespace: default
secrets:
  - name: regcred

---

apiVersion: v1
data:
  .dockerconfigjson: <encoded docker registry auth data>
kind: Secret
metadata:
  name: regcred
  namespace: default
type: kubernetes.io/dockerconfigjson

```



定义资源

```yaml

apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: singlecloud-ui-image
spec:
  type: image
  params:
    - name: url
      value: docker.io/gsmlg/singlecloud-ui 

---

apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: singlecloud-image
spec:
  type: image
  params:
    - name: url
      value: docker.io/gsmlg/singlecloud

---

apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: zcloud-image
spec:
  type: image
  params:
    - name: url
      value: docker.io/gsmlg/zcloud


```


创建task

```yaml

apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-image-from-git
spec:
  inputs:
    resources:
      - name: docker-source
        type: git
    params:
      - name: pathToDockerFile
        type: string
        description: The path to the dockerfile to build
        default: /workspace/docker-source/Dockerfile
      - name: pathToContext
        type: string
        description:
          The build context used by Kaniko
          (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
        default: /workspace/docker-source
      - name: imageTagName
        type: string
        description:
          The build image's tagName
        default: latest
  outputs:
    resources:
      - name: builtImage
        type: image
  steps:
    - name: build-and-push
      image: docker.io/gsmlg/kaniko-project-executor:v0.13.0
      # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
      env:
        - name: "DOCKER_CONFIG"
          value: "/builder/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(inputs.params.pathToDockerFile)
        - --destination=$(outputs.resources.builtImage.url):$(inputs.params.imageTagName)
        - --context=$(inputs.params.pathToContext)

---

apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-zcloud
spec:
  inputs:
    resources:
      - name: docker-source
        type: git
      - name: image
        type: image
      - name: uiImage
        type: image
    params:
      - name: pathToDockerFile
        type: string
        description: The path to the dockerfile to build
        default: /workspace/docker-source/Dockerfile
      - name: pathToContext
        type: string
        description:
          The build context used by Kaniko
          (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
        default: /workspace/docker-source
      - name: imageTagName
        type: string
        description:
          The build image's tagName
        default: latest
  outputs:
    resources:
      - name: builtImage
        type: image
  steps:
    - name: setup-dockerfile
      image: docker.io/ubuntu:18.04
      command:
        - /workspace/docker-source/setup.sh
      args:
        - $(inputs.resources.image.url)@$(inputs.resources.image.digest)"
        - $(inputs.resources.uiImage.url)@$(inputs.resources.uiImage.digest)"
        - /workspace/docker-source/Dockerfile
    - name: build-and-push
      image: docker.io/gsmlg/kaniko-project-executor:v0.13.0
      # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
      env:
        - name: "DOCKER_CONFIG"
          value: "/builder/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(inputs.params.pathToDockerFile)
        - --destination=$(outputs.resources.builtImage.url):$(inputs.params.imageTagName)
        - --context=$(inputs.params.pathToContext)
        - --build-arg="repo=$(inputs.resources.image.url)"
        - --build-arg="uirepo=$(inputs.resources.uiImage.url)"
        - --build-arg="branch=$(inputs.resources.image.digiest)"
        - --build-arg="uibranch=$(inputs.resources.uiImage.digiest)"

---

apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: zcloud-build-pipeline
spec:
  resources:
    - name: singlecloud-repo
      type: git
    - name: singlecloud-ui-repo
      type: git
    - name: zcloud-repo
      type: git
    - name: singlecloud-image
      type: image
    - name: singlecloud-ui-image
      type: image
    - name: zcloud-image
      type: image
  params:
    - name: imageTagName
      default: "latest"
  tasks:
    - name: build-singlecloud-ui
      taskRef:
        name: build-image-from-git
      resources:
        inputs:
          - name: docker-source
            resource: singlecloud-ui-repo
        outputs:
          - name: builtImage
            resource: singlecloud-ui-image
    - name: build-singlecloud
      taskRef:
        name: build-image-from-git
      resources:
        inputs:
          - name: docker-source
            resource: singlecloud-repo
        outputs:
          - name: builtImage
            resource: singlecloud-image
    - name: build-zcloud
      taskRef:
        name: build-zcloud
      runAfter:
        - build-singlecloud
        - build-singlecloud-ui
      params:
      - name: imageTagName
        value: $(params.imageTagName)
      resources:
        inputs:
        - name: docker-source
          resource: zcloud-repo
        - name: uiImage
          resource: singlecloud-ui-image
          from:
          - build-singlecloud-ui
        - name: image
          resource: singlecloud-image
          from:
          - build-singlecloud
        outputs:
        - name: builtImage
          resource: zcloud-image


```


运行pipelinue:

```yaml

apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  generateName: zcloud-build-run-
spec:
  pipelineRef:
    name: zcloud-build-pipeline
  serviceAccount: pipeline-run-service
  resources:
  - name: singlecloud-repo
    resourceSpec:
      type: git
      params:
      - name: revision
        value: master
      - name: url
        value: https://github.com/zdnscloud/singlecloud
  - name: singlecloud-ui-repo
    resourceSpec:
      type: git
      params:
      - name: revision
        value: master
      - name: url
        value: https://github.com/zdnscloud/singlecloud-ui
  - name: zcloud-repo
    resourceSpec:
      type: git
      params:
      - name: revision
        value: master
      - name: url
        value: https://github.com/gsmlg/zcloud-image
  - name: singlecloud-image
    resourceRef:
      name: singlecloud-image
  - name: singlecloud-ui-image
    resourceRef:
      name: singlecloud-ui-image
  - name: zcloud-image
    resourceRef:
      name: zcloud-image
  params:
  - name: imageTagName
    value: master


```
