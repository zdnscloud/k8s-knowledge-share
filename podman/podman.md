###  Podman 是什么？   
* [Podman](https://podman.io/)（Pod Manager）是功能齐全的容器引擎，它是一个简单的无守护程序工具。
* 提供了与Docker-CLI相类似的命令行，可简化从其他容器引擎的过渡，并允许管理Pod，容器和映像。
* 大多数Podman命令可以作为普通用户运行，而无需额外的权限。
* 是libpod的一部分。
* 它的定义可以简单用这个命令表示：`alias docker=podman`。
> Libpod 是一个用于创建并运行 OCI 容器 pod 的库和工具，支持 Fedora、RHEL 与 Ubuntu 等的不同版本。
> 

#### 简述 OCI
OCI(Open Container Initiative)是由多家公司共同成立的项目，并由linux基金会进行管理，致力于container runtime的标准的制定和runc的开发等工作。
	
OCI项目计划：
* Runtime: runc (or any OCI compliant runtime) and OCI runtime tools to generate the spec
* Images: Image management using containers/image
* Storage: Container and image storage is managed by containers/storage
* Networking: Networking support through use of CNI
* Builds: Builds are supported via Buildah.
* Conmon: Conmon is a tool for monitoring OCI runtimes.
>
> * container runtime: 主要负责的是容器的生命周期的管理。oci的runtime spec标准中对于容器的状态描述，以及对于容器的创建、删除、查看等操作进行了定义。
>
> * runc: 是对于OCI标准的一个参考实现，是一个可以用于创建和运行容器的CLI(command-line interface)工具。runc直接与容器所依赖的cgroup/linux kernel等进行交互，负责为容器配置cgroup/namespace等启动容器所需的环境，创建启动容器的相关进程。
>
为了兼容oci标准，docker也做了架构调整。将容器运行时相关的程序从docker daemon剥离出来，形成了containerd。Containerd向docker提供运行容器的API，二者通过grpc进行交互。containerd最后会通过runc来实际运行容器。

![](http://xuxinkun.github.io/img/docker-oci-runc-k8s/containerd.png)
	
CRI-O为kubernetes提供了一个符合OCI兼容运行时的标准接口。runc是OCI运行时规范的参考实现。Kubernetes通过CRI-O调用runtC运行时，然后runC与Linux内核对话以运行容器。这绕过了对Docker守护程序的需要，并进行了容器化。使用CRI-O，不需要Docker守护程序让kubernetes集群运行容器。
	
![](https://xuxinkun.github.io/img/docker-oci-runc-k8s/kubelet.png)
	
在较高的层面上，Libpod 和 Podman 的作用范围如下：
* 支持多种镜像格式，包括 OCI 和 Docker图像模式。
* 支持多种方式下载镜像，包括信任和镜像验证。
* 容器镜像管理（管理镜像层、覆盖文件系统等）。
* 全面管理容器生命周期。
* 支持 pod 管理容器组。
* pos 和容器的资源隔离。
* 与 CRI-O 集成以共享容器和后端代码。

***

### docker 是如何工作的？
在下图中，我们可以看到Docker守护程序提供了所需的所有功能：
* 从图像注册表中拉出图像
* 在本地容器存储中制作图像副本，并在这些容器中添加图层
* 提交容器并从主机存储库中删除本地容器映像
* 要求内核运行具有正确名称空间和cgroup等的容器。

本质上，Docker守护程序使用注册表，映像，容器和内核来完成所有工作。Docker命令行界面（CLI）要求守护程序代表您执行此操作。

![](https://developers.redhat.com/blog/wp-content/uploads/2019/02/fig1.png)

随着使用量的增加，Docker用户担心这种方法有几个原因。列举一些：

* 单个过程可能是单个故障点。
* 该进程拥有所有子进程（正在运行的容器）。
* 如果发生故障，则存在孤立的进程。
* 构建容器导致安全漏洞。
* 所有Docker操作都必须由具有相同完全根权限的一个或多个用户执行。
* Docker 守护程序在多个核心上占用 100% CPU 资源，并导致主机无法正常使用。
***
### Podman 是如何工作的？
* Podman在内部使用Buildah来创建容器图像。 两个工具共享图像（而不是容器）存储，因此每个工具可以使用或操纵由另一个创建的图像（但不能操纵容器）。


![](https://developers.redhat.com/blog/wp-content/uploads/2019/02/fig2.png)

***

### 比较 Docker 与 Podman
* Podman可以替换Docker中了大多数子命令`（RUN，PUSH，PULL等）`，一个显着的区别是Podman在某些命令中添加了一些便利标志,比如 `--all(-a)`。

	> Podman指令链接 
		https://github.com/containers/libpod/blob/master/docs/podman.1.md
	   
* Podman将Containers和Images存储在与Docker不同的位置。
	* Podman的本地存储库位于  /var/lib/containers
	* Docker的本地存储库位于 /var/lib/docker
	
	>  Podman使用用户主目录中的存储库：  ~/.local/share/containers,确保了每个用户都有单独的容器和图像集，并且所有人都可以在同一主机上同时使用Podman。
* Podman不需要守护进程，而是使用用户命名空间来模拟容器中的root，无需连接到具有root权限的套接字保证容器的体系安全。
***

### 安装Podman
* Fedora, CentOS 
 ```
 sudo yum -y install podman
 ```
* Ubuntu
``` 
sudo apt-get update -qq
sudo apt-get install -qq -y software-properties-common uidmap
sudo add-apt-repository -y ppa:projectatomic/ppa
sudo apt-get update -qq
sudo apt-get -qq -y install podman
```
* Linux
```
sudo pacman -S podman
```
* MacOS
```
brew cask install podman
```
***

### Podmam 的简单使用
查询镜像
```
podman search zhanqq/qq-repositoty
```
获取镜像
```
pull zhanqq/qq-repositoty:part1
```
查看镜像
```
podman images 
```
启动容器
```
podman run -d -p 4000:80 zhanqq/qq-repositoty:part1
```
> 输入`curl -4 http://localhost:4000`，查看结果
> 
查看容器
```
podman ps 
```
暂停容器
```
podman stop -a
```
删除容器
```
podman rm 
```
使用Dockerfile构建镜像
```
podman build  -t  aaa  -f Dockerfile.hello
```
推送到Docker Hub
```
podman push 3385c0ac669e docker://docker.io/zhanqq/qq-repositoty
```
> 先登陆 `podman login docker.io`
***
### 简述 Buildah
用来构建OCI图像,相比Docker构建Podmang构建镜像速度更快，并使用覆盖存储驱动程序。
* 可以使用Dockerfiles构建镜像，并且不需要任何root权限。
* Buildah也支持非Dockerfiles构建镜像，可以允许将其他脚本语言集成到构建过程中。

此外,[Buildah](https://buildah.io/)可以从头开始构建图像，即根本没有图像。实际上，查看通过buildah from scratch 命令创建的容器存储将产生一个空目录。这对于创建非常轻量级的映像很有用，该映像仅包含运行应用程序所需的软件包。

![](https://developers.redhat.com/blog/wp-content/uploads/2019/02/fig3-1024x703.png)
***
### PodMan 和 Buildah的关系
* Buildah和Podman是可以在绝大多数的Linux平台上的两个互补的开源项目，都是用于处理开放容器倡议（OCI）图像和容器的命令行工具，这两个项目的专业不同。
* Buildah专门构建OCI图像，其最终目标是提供一个较低级别的coreutils接口来构建映像。Buildah遵循简单的fork-exec模型，并且不作为守护程序运行，但是它基于golang中全面的API，可以将其出售给其他工具。
* Podman专门维护和修改OCI图像的所有命令和功能。为了通过Dockerfiles构建容器映像，Podman使用Buildah的golang API，可以独立于Buildah进行安装。
* Podman和Buildah之间的主要区别在于它们的容器概念。

	Podman允许用户创建传统容器，并且将在整个容器生命周期（暂停，检查点/还原等）中存在。虽然Buildah容器的创建只是为了允许将内容添加到容器映像中。每个项目都有一个不共享的单独内部容器表示。因此，您无法在Buildah中看到Podman容器，反之亦然。但是，在Buildah和Podman之间，容器映像的内部表示是相同的。鉴于此，由一个创建，拉出或修改的任何容器映像都可以被另一个查看和使用。


  |Command        | Podman Behavior   | Buildah Behavior  |
    | --------   | -----:   | :----: |
    | build        | Calls buildah bud      |  	Provides the build-using-dockerfile (bud) command that emulates Docker’s build command.    |
    | commit        | Commits a Podman container into a container image. Does not work on a Buildah container. Once committed the resulting image can be used by either Podman or Buildah.      |   Commits a Buildah container into a container image. Does not work on a Podman container. Once committed, the resulting image can be used by either Buildah or Podman.   |
    | mount        | 	Mounts a Podman container. Does not work on a Buildah container.      |   Mounts a Buildah container. Does not work on a Podman container.    |
	| pull and push        | Pull or push an image from a container image registry. Functionally the same as Buildah.      |   Pull or push an image from a container image registry. Functionally the same as Podman.   |
	| run        | 	Run a process in a new container in the same manner as docker run.      |   Runs the container in the same way as the RUN command in a Dockerfile.    |
	| rm        | Removes a Podman container. Does not work on a Buildah container.      |   Removes a Buildah container. Does not work on a Podman container.   |
	| rmi, images, tag        |  tag	Equivalent on both projects.      |    tag	Equivalent on both projects.    |
	| containers and ps        | ps is used to list Podman containers. The containers command does not exist.     |   containers is used to list Buildah containers. The ps command does not exist.    |
	
##### 总结两个项目之间差异，buildah run该命令在Dockerfile中模拟RUN命令，而podman run该docker run命令在功能上模拟该命令。
> *  Dockerfile的run 用于指定 `docker build `过程中要运行的命令 ，通常我们首先定义Dockerfile 文件，然后通过 `docker build `构建得到镜像文件，之后才能基于镜像文件通过 `docker run` 启动一个容器的实例。
> 
> * 那么在启动一个容器的时候，就可以改变镜像文件中的一些参数，而镜像文件中的这些参数往往是通过Dockerfile文件定义的，比如ENTRYPOINT、CMD、WORKDIR等指令。但并非Dockerfile文件中的所有定义都可以在启动容器的时候被重新定义：比如FROM、MAINTAINER、RUN、COPY等指令。
	
##### 简而言之，Buildah是创建OCI映像的有效方法，而Podman允许您使用熟悉的容器CLI命令在生产环境中管理和维护这些映像和容器。它们共同构成了支持OCI容器映像和容器需求的坚实基础
****

### 配合使用
* 专门从事签名并将图像推送到各种存储后端。有关这些任务，请参见 [Skopeo](https://github.com/containers/skopeo/)。
* 容器运行时守护程序，用于使用Kubernetes CRI接口。[CRI-O](https://github.com/cri-o/cri-o) 专长于此。
* 配套docker-compose。我们相信Kubernetes是构成Pod和协调容器的事实上的标准，这使Kubernetes YAML成为了事实上的标准文件格式。因此，Podman允许从Kubernetes YAML文件创建和执行Pod（请参阅 podman-play-kube）。Podman还可以基于容器或Pod生成Kubernetes YAML（请参阅 [podman-generate-kube](https://github.com/containers/libpod/blob/master/docs/podman-play-kube.1.md)），从而可以轻松地从本地开发环境过渡到生产Kubernetes集群。如果Kubernetes不符合您的要求，那么还有其他支持docker -compose格式的第三方工具，例如 [kompose](https://github.com/kubernetes/kompose/)和 [podman-compose](https://github.com/containers/podman-compose) 可能适合您的环境
	