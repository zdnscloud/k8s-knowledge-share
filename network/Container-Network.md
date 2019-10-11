# 容器网络技术栈
  ![""](pictures/container_network_stack.jpg)

## 低层网络层

包括网络设备，iptables，路由，IPVLAN和Linux命名空间

## 容器网络层

Docker单节点网络

Docker多节点网络

CNI

## 容器编排层

K8S网络

# Linux命名空间

## 概念

namespace 是 Linux内核用来隔离内核资源的方式。通过 namespace 可以让一些进程只能看到与自己相关的一部分资源，而另外一些进程也只能看到与它们自己相关的资源，这两拨进程根本就感觉不到对方的存在。具体的实现方式是把一个或多个进程的相关资源指定在同一个 namespace 中。

Linux namespaces 是对全局系统资源的一种封装隔离，使得处于不同 namespace 的进程拥有独立的全局系统资源，改变一个 namespace 中的系统资源只会影响当前 namespace 里的进程，对其他 namespace 中的进程没有影响。

## 用途

Linux 内核实现 namespace 的一个主要目的就是实现轻量级虚拟化(容器)服务。在同一个 namespace 下的进程可以感知彼此的变化，而对外界的进程一无所知。这样就可以让容器中的进程产生错觉，认为自己置身于一个独立的系统中，从而达到隔离的目的。也就是说 linux 内核提供的 namespace 技术为 docker 等容器技术的出现和发展提供了基础条件。

我们可以从 docker 实现者的角度考虑该如何实现一个资源隔离的容器。比如是不是可以通过 chroot 命令切换根目录的挂载点，从而隔离文件系统。为了在分布式的环境下进行通信和定位，容器必须要有独立的 IP、端口和路由等，这就需要对网络进行隔离。同时容器还需要一个独立的主机名以便在网络中标识自己。接下来还需要进程间的通信、用户权限等的隔离。最后，运行在容器中的应用需要有进程号(PID)，自然也需要与宿主机中的 PID 进行隔离。也就是说这六种隔离能力是实现一个容器的基础

## 类型

  ![""](pictures/linux_namespace.png)

### IPC

用于隔离进程间通讯所需的资源（ System V IPC, POSIX message queues），PID命名空间和IPC命名空间可以组合起来用，同一个IPC名字空间内的进程可以彼此看见，允许进行交互，不同空间进程无法交互

### Network

Network Namespace为进程提供了一个完全独立的网络协议栈的视图。包括网络设备接口，IPv4和IPv6协议栈，IP路由表，防火墙规则，sockets等等。一个Network Namespace提供了一份独立的网络环境，就跟一个独立的系统一样。

### Mount

每个进程都存在于一个mount Namespace里面，mount Namespace为进程提供了一个文件层次视图。如果不设定这个flag，子进程和父进程将共享一个mount Namespace，其后子进程调用mount或umount将会影响到所有该Namespace内的进程。如果子进程在一个独立的mount Namespace里面，就可以调用mount或umount建立一份新的文件层次视图。

### PID

linux通过命名空间管理进程号，同一个进程，在不同的命名空间进程号不同！进程命名空间是一个父子结构，子空间对于父空间可见。

### User

用于隔离用户

### UTS

用于隔离主机名

### Cgroup

cgroup是为了对一组进程进行统一的资源监控和限制，包含了cpu、memory和磁盘IO

从版本号为 3.8 的内核开始，/proc/[pid]/ns 目录下会包含进程所属的 namespace 信息，使用下面的命令可以查看当前进程所属的 namespace 信息：

$ ll /proc/$$/ns

  ![""](pictures/linux_namespace_ls.png)
 
这些 namespace 文件都是链接文件。链接文件的内容的格式为 xxx:[inode number]。其中的 xxx 为 namespace 的类型，inode number 则用来标识一个 namespace，我们也可以把它理解为 namespace 的 ID。如果两个进程的某个 namespace 文件指向同一个链接文件，说明其相关资源在同一个 namespace中。

其次，在 /proc/[pid]/ns 里放置这些链接文件的另外一个作用是，一旦这些链接文件被打开，只要打开的文件描述符(fd)存在，那么就算该 namespace 下的所有进程都已结束，这个 namespace 也会一直存在，后续的进程还可以再加入进来。


# Docker网络

Docker内建了3个网络

  ![""](pictures/docker_network_ls.png)
 
## Docker单节点的四种网络模型

### Bridge
    通常用于在独立容器中运行的应用程序，这是默认的Docker网络模型。
Docker守护进程创建了docker0，它是一个虚拟以太网网桥，它可以在连接到它的任何port之间自动转发数据包。 默认情况下，守护进程会通过创建一对对的网络接口将主机上的所有容器连接到此网桥，将其中一个对等端分配为容器的eth0接口，并将另一个对等端放在主机的命名空间中；以及将专用IP范围内的一个子网分配给网桥。
        
  ![""](pictures/docker_bridge.png.jpg)
 
其特点如下：

- 使用一个 linux bridge，默认为 docker0
- 使用 veth 对，一头在容器的网络 namespace 中，一头在 docker0 上
- 该模式下Docker Container不具有一个公有IP，因为宿主机的IP地址与veth pair的 IP地址不在同一个网段内
- Docker采用 NAT 方式，将容器内部的服务监听的端口与宿主机的某一个端口port 进行&quot;绑定&quot;，使得宿主机以外的世界可以主动将网络报文发送至容器内部
- 外界访问容器内的服务时，需要访问宿主机的 IP 以及宿主机的端口 port
- NAT 模式由于是在三层网络上的实现手段，故肯定会影响网络的传输效率。
- 容器拥有独立、隔离的网络栈；让容器和宿主机以外的世界通过NAT建立通信

例：

  ![""](pictures/docker_bridge_1.jpg)
  
启动容器test1，自动获得一个IP 172.17.0.2，和主机docker0处于同一网络，且网关为docker0

  ![""](pictures/docker_bridge_2.jpg)

可以看到网桥docker0已经多了一个接口

  ![""](pictures/docker_bridge_3.jpg)

在主机上看到在POSTROUTING链中做了地址伪装，MASQUERADE动作，这样容器就可以通过源地址转换NAT访问外部网络了

如果容器服务监听端口并使用-P做了映射，在主机还可以看到DNAT动作，这样外部用户就可以通过主机的映射端口访问容器内部服务

docker network inspect bridge命令可以查看bridge网络详细情况

### Host

        也用于独立容器，共享主机的网络。

该模式有效地禁用了Docker容器的网络隔离特性。容器有效地继承了主机的IP地址。这种模式比桥接模式更快，因为没有路由开销，但它将容器直接暴露给公共网络，并会带来所有安全隐患。此外作为共享主机网络命名空间的后果，您需要以某种方式管理端口的分配。

  ![""](pictures/docker_host.jpg)
 
其特点包括：

- 这种模式下的容器没有隔离的 network namespace
- 容器的 IP 地址同 Docker host 的 IP 地址
- 需要注意容器中服务的端口号不能与 Docker host 上已经使用的端口号相冲突
- host 模式能够和其它模式共存

例：

  ![""](pictures/docker_host_1.jpg)
 
启动容器test2，使用—net host指定为host网络类型，可以看到容器的IP与主机一样，容器服务监听了的80端口实际上就是在主机上监听的，外部访问容器直接使用主机IP即可，不需要NAT转换。但容器的其他信息（Pid、文件系统）等还是与主机是隔离的

docker network inspect host命令可以查看host网络详细情况

### Container
      共享其他容器网络

容器会共享其他容器的网络环境，两个容器之间不存在网络隔离，而这两个容器又与宿主机以及除此之外其他的容器存在网络隔离，因此需要注意端口冲突情况，否则第二个容器将无法被启动。  Kubernetes网络使用这种模式。
        
  ![""](pictures/docker_container.jpg)
 
其特点是：

- 共享同一个容器网络的所有容器的网络信息是一致，除了网络信息外，其他均继续保持隔离

例

  ![""](pictures/docker_container_1.png)
 
启动容器test3，使用--network container:test1指定使用已经存在的test1容器的网络，可以看到test3容器的网络信息与test1一样，是共享的

### None

        禁用Docker端的网络支持，并允许自定义网络。

不为 Docker 容器构造任何网络环境。一旦Docker 容器采用了none 网络模式，那么容器内部就只能使用loopback网络设备，不会再有其他的网络资源，容器只能使用127.0.0.1的本机网络。

例

  ![""](pictures/docker_none_1.jpg)
 
启动容器test4，使用--network none放弃为容器配置网络，进入容器查看除了lo外，没有其他网络设备，容器只能使用127.0.0.1的本地网络

docker network inspect none命令可以查看none网络详细情况

## 自定义网络

### Bridge
创建自定义网络

docker network create --driver network\_type network\_name

其中--driver后面支持的类型有三种：bridge、macvlan、overlay

查看网络信息

docker network inspect network\_name  

查看网桥

brctl show

在主机上可以看到两个网桥对应的路由信息

  ![""](pictures/docker_bridge_4.jpg)

创建容器，通过--network my-bridge加入这个网络，同处于my-bridge的容器间网络是互通的，网络信息与默认的bridge没有差别（依然SNAT出网和DNAT入网），但该网桥与其他网桥之间是不通的。这种方式适用于单节点

### Overlay

Overlay 可以使得我们将报文在 IP 报文之上再次封装，VXLAN 技术为其核心，VXLAN是将以太网报文封装成UDP报文进行隧道传输，UDP目的端口为已知端口，源端口可按流分配，标准5元组方式有利于在IP网络转发过程中进行负载分担；隔离标识采用24比特来表示；未知目的、广播、组播等网络流量均被封装为组播转发。

>   单机模式是无法创建overlay网络模型的，需要借助分部署存储（etcd或者编排系统），部署完etcd后需要配置docker指定etcd地址后重启docker。

创建overlay 网络（只需在一个节点执行）

Docker network create -d overlay my-overlay

在两个主机分别创建容器，使用--network my-overlay加入overlay网络，可以看到容器有两块网卡，eth1为走普通NAT模式，eth0 是 overlay 网段上分配的IP地址，也就是它走的是 overlay 网络，它的 MTU 是 1450 而不是 1500。

docker 会为每个 overlay 网络创建一个独立的 network namespace，其中会有一个 linux bridge br0， veth pair 一端连接到容器中（即 eth0），另一端连接到 namespace 的 br0 上。br0 除了连接所有的 veth pair，还会连接一个 vxlan 设备，用于与其他 host 建立 vxlan tunnel。容器之间的数据就是通过这个 tunnel 通信的。
```
ln -s /var/run/docker/netns/ /var/run/netns
ip netns
```
可以看到两个 host 上有一个相同的 namespace

ip netns exec 1-625e8bbfff brctl show查看 namespace 中的 br0 上的设备

  ![""](pictures/docker_overlay.jpg)

不同的 overlay 网络是相互隔离的。

docker 默认为 overlay 网络分配 24 位掩码的子网（10.0.0.0/24），所有主机共享这个 subnet，容器启动时会顺序从此空间分配 IP。当然我们也可以通过 --subnet 指定 IP 空间。

## Docker多节点网络方案

### 基于实现方式分为

#### 隧道

- Weave：UDP广播，本机建立新的BR，通过PCAP互通。
- Open vSwitch(OVS)：基于VxLAN和GRE协议，但是性能方面损失比较严重。
- Flannel：UDP广播，VxLan。

隧道方案在IaaS层的网络中应用也比较多，大家共识是随着节点规模的增长复杂度会提升，而且出了网络问题跟踪起来比较麻烦，大规模集群情况下这是需要考虑的一个点。

#### 路由

- Calico：基于BGP协议的路由方案，支持很细致的ACL控制，对混合云亲和度比较高。
- Macvlan：从逻辑和Kernel层来看隔离性和性能最优的方案，基于二层隔离，所以需要二层路由器支持，大多数云服务商不支持，所以混合云上比较难以实现。

### 基于网络模型分为

#### Docker Libnetwork Container Network Model(CNM)阵营

- Docker Swarm overlay
- Macvlan &amp; IP network drivers
- Calico
- Contiv（from Cisco）

Docker Libnetwork的优势就是原生，而且和Docker容器生命周期结合紧密；缺点也可以理解为是原生，被Docker&quot;绑架&quot;

#### Container Network Interface(CNI)阵营

- Kubernetes
- Weave
- Macvlan
- Flannel
- Calico
- Contiv
- Mesos CNI

兼容其他容器技术(e.g. rkt)及上层编排系统(Kuberneres &amp; Mesos)，而且社区活跃势头迅猛，Kubernetes加上CoreOS主推；缺点是非Docker原生

# K8S网络

## Pod网络与Docker网络

  ![""](pictures/k8s-docker.png)

 K8S中每个Pod在创建时都是先创建一个pause容器，业务容器与pause容器使用同一个网络名称空间。
 
 docker inspect $ID 会发现业务容器的网络模式是container，而pause容器的网络模式是none，因为k8s使用了cni，具体的容器网卡、IP、路由等信息是有cni配置的

## Kubelet逻辑分析

kubelet调用createPodSandbox来创建pause容器，并为其配置网络环境

github.com/kubernetes/pkg/kubelet/kuberuntime/kuberuntime\_sandbox.go
```
func (m *kubeGenericRuntimeManager) createPodSandbox(pod *v1.Pod, attempt uint32) (string, string, error) {

      podSandBoxID, err := m.runtimeService.RunPodSandbox(podSandboxConfig, runtimeHandler)

}
```
github.com/kubernetes/pkg/kubelet/dockershim/docker\_sandbox.go
```
func (ds *dockerService) RunPodSandbox(ctx context.Context, r *runtimeapi.RunPodSandboxRequest) (*runtimeapi.RunPodSandboxResponse, error) {

      err = ds.client.StartContainer(createResp.ID)

     #DNS配置：通过docker inspecft查看容器信息，然后再重写容器位于宿主机的resolv文件

      err = ds.network.SetUpPod(config.GetMetadata().Namespace, config.GetMetadata().Name, cID, config.Annotations, networkOptions)

}
```
github.com/kubernetes /pkg/kubelet/dockershim/network/plugins.go
```
func (pm *PluginManager) SetUpPod(podNamespace, podName string, id kubecontainer.ContainerID, annotations, options map[string]string) error {

      if err := pm.plugin.SetUpPod(podNamespace, podName, id, annotations, options); err != nil {

}
#调用plugin的SetUpPod方法，这里plugin是一个interface, 具体使用哪个plugin是由kubelet的启动参数–network-plugin决定的，我们配置的是cni

}
```
github.com/kubernetes/pkg/kubelet/dockershim/network/cni/cni.go
```
获取配置文件

func getDefaultCNINetwork(confDir string, binDirs []string) (*cniNetwork, error) {

      files, err := libcni.ConfFiles(confDir, []string{.conf, .conflist, .json})

      switch {

      case err != nil:

              return nil, err

      case len(files) == 0:

              return nil, fmt.Errorf(No networks found in %s, confDir)

      }

      sort.Strings(files)

}

配置pause容器的eth0接口的网络

func (plugin *cniNetworkPlugin) SetUpPod(namespace string, name string, id kubecontainer.ContainerID, annotations, options map[string]string) error {

      _, err = plugin.addToNetwork(plugin.getDefaultNetwork(), name, namespace, id, netnsPath, annotations, options)

}

func (plugin *cniNetworkPlugin) addToNetwork(network *cniNetwork, podName string, podNamespace string, podSandboxID kubecontainer.ContainerID, podNetnsPath string, annotations, options map[string]string) (cnitypes.Result, error) {

      res, err := cniNet.AddNetworkList(netConf, rt)

}

func (c *CNIConfig) AddNetworkList(list *NetworkConfigList, rt *RuntimeConf) (types.Result, error) {

      prevResult, err = invoke.ExecPluginWithResult(pluginPath, newConf.Bytes, c.args(ADD, rt))

}
该函数会遍历plugin，根据cni的type在binDir中找到同名插件，返回该插件的全路径。最后执行ExecPluginWithResult函数，它将调用cni的二进制文件并传入newConf参数以及RuntimeConf和一个ADD参数，其中ADD代表给容器添加网络。
```
分析到这，kubelet的网络配置已经完成了，我们最终会看到kubelet在生成新pod的时候会先生成一个sandbox容器（pause），kubelet会根据pod的yaml信息和kubelet的cni参数配置生成一个cni runtime配置，最后调用cni插件完成docker容器的网络配置。


