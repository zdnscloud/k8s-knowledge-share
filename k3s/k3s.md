### 什么是k3s？

k3s是微型的kubernetes发行版本

1. CNCF认证的Kubernetes发行版
2. 50MB左右二进制包，500MB左右内存消耗
3. 单一进程包含Kubernetes master, Kubelet, 和 containerd 
4. 支持SQLite/Mysql/PostgreSQL/DQlite和etcd
5. 同时为x86_64, Arm64, 和 Armv7 平台发布

### k3s 基本架构

#### 整体架构

##### 部署架构 

![cc737ff7270d834dd8ca3e6efcc4509b.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1781)

轻量化

相比k8s
移除
- 删除旧的和非必要的组件
- alpha feature
- In-tree cloud providers
- In-tree storage strivers
- Docker (optional)

增加
- 简单的安装
- etcd默认使用SQLite做数据源
- TLS管理
- 自动helm chart管理
- containerd、cordons、Flannel集成


##### k3s 代码结构

![d7fece440f1c3f7c3a4333c3c924ebfd.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1786)

注意事项：

- contaienrd是独立进程，也可以使用docker
- Tunnel Proxy 负责维护 k3s server 和 k3s agent 之间的链接，采用Basic Auth的方式来进行认证


##### Demo

k3s 进程

- server

\# 6443 k3s server, 6444 api-server

- agent

```
pstree -p -aT $pid
ps -T $pid
```

##### 参考链接

- [k3s 代码组织结构分析参考]( https://mp.weixin.qq.com/s/1o9X0Dlv2WhUS6iC9P-KQA)
- [k3s 部署参考](https://mp.weixin.qq.com/s/V-VyWrCZux5WXxD__QpEWQ)
- [k3s 为何轻量](https://mp.weixin.qq.com/s/5aprEfYSWJyVoW4trDp4Hw)

#### k3s的高可用架构

![efddb62fd824b694d539d686cbdf5cbd.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1787)

![01a39ea3b8684c5889e69a1c625e8a95.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1788)

![983420ce2dbc3d812c68639fae050dee.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1789)

##### 高可用 - 外置数据库

- PostgreSQL (v10.7、v11.5) ü 
- MySQL (v5.7)
- etcd (v3.3.15)

实现方式：
rancher/kine

![fb719eff62e7da623289c06cf66e9f2e.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1790)


##### 高可用 - 分布数据库

**SQLite**

- 软件库，无独立进程:C 语言编写
- 零管理配置:不需要服务管理进程
- 事务安全:兼容ACID，可并发访问
- 标准SQL支持
- 单一磁盘文件

k3s默认使用
`/var/lib/rancher/k3s/server/db/state.db`


**Dqlite**

- 软件库，无独立进程:C 语言编写
- 分布式一致:C-Raft 实现
- 兼容 SQLite

Dqlite 原理

- 一个k3sserver进程内含SQL的client组 件和 server 组件
- client 仅连接一个 server
- server 链接 SQLite 软件库
- 奇数个k3sserver进程
- client 需连接所有 server
- server 链接 Dqlite 软件库
- server 通过 C-Raft 来选主
- client 发现 主 server
- client 把请求都发送到主 server
- 主server通过C-Raft给从server发送差分日志作为数据同步

![e8387ec04f8c4c9843d12e1c2c5659fb.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1792)


##### 参考链接

- [SQLite 基础命令](https://www.runoob.com/sqlite/sqlite-commands.html)
- [SQLite Go语言驱动实现](https://github.com/mattn/go-sqlite3)
- [CGO的动静态链接介绍](https://books.studygolang.com/advanced-go-programming-book/ch2-cgo/ch2-06-static-shared-lib.html)
- [Dqlite 讲解](https://fosdem.org/2020/schedule/event/dqlite/)
- [Raft 协议介绍](https://raft.github.io/)
- [k3s 社区高可用介绍](https://mp.weixin.qq.com/s/3by95UIJ7v41KXElt7uvGA)


#### Containerd

`Containerd`的设计目的是嵌入更大的系统

Docker的历史

- 1.8 之前:docker -d  -   2015年，OCI 成立
- 1.8 - 1.11:docker daemon - runtime-spec 制定
- 1.11以后:docker、dockerd - libcontainer -> runC
- dockerd = docker engine + containerd + containerd-shim + runC


![a558aca87f7a8679d9fe1a2da820ec21.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1794)

![5471d6d59bbe3ba83a514e23af38dcef.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1795)

![1c7a968cb544086aaaa167819a26a1b0.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1796)

![33bede9601ebd8546cd7de980cc55135.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1797)

![1cc7615d71efe2799e6bbcd38bb3d718.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1798)


**操作**

|镜像操作|Docker|ContainerD|
|:---|:---|:---|:---|
|本地镜像列表 |docker images |crictl images| ctr images list
|下载镜像| docker pull | crictl pull| ctr (images) pull
|上传镜像 |docker push | |ctr (images) push
|删除本地镜像 |docker rmi |crictl rmi |ctr images remove
|标记本地镜像 |docker tag | | ctr (images) tag
|镜像详情|docker inspect|crictl inspecti 
|容器列表 |docker ps |crictl ps |ctr containers list
|创建容器 |docker create |crictl create |ctr containers create
|运行容器 |docker start |crictl start |ctr (tasks) start
|停止容器 |docker stop |crictl stop |ctr (tasks) pause
|删除容器 |docker rm |crictl rm |ctr (tasks) rm
|容器详情 |docker inspect |crictl inspect
|连接容器 |docker attach |crictl attach |ctr (tasks) attach
|容器内操作 |docker exec |crictl exec |ctr (tasks) exec
|容器日志 |docker logs |crictl logs
|容器状态 |docker stats |crictl stats
|显示POD列表 ||crictl pods
|查看POD详情 ||crictl inspectp
|运行POD ||crictl runp
|停止POD ||crictl stopp
|删除POD ||crictl rmp
|转发端口到POD ||crictl port-foward
   

k3s 内置containerd

- ctr : 单纯的容器管理
- crictl : 从 Kubernetes视角出发，对POD、容器进行管理

k3s 修改containerd配置
修改`/var/lib/rancher/k3s/agent/etc/containerd/config.toml`同目录下`config.toml.tmpl`文件，重启k3s

contaienrd日志
`/var/lib/rancher/k3s/agent/containerd/containerd.log`

##### 参考链接

- [cri的演进史](https://zhuanlan.zhihu.com/p/87602649)
- [runC的介绍](https://www.infoq.cn/article/docker-standard-container-execution-engine-runc)
- [k3s 社区 Containerd 使用介绍](https://mp.weixin.qq.com/s/EJqS7G36f7_srgxSZaCuPw)
- [crictl 基本操作](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md)
- [Containerd CNI插件配置介绍](https://github.com/containerd/cri/blob/master/docs/config.md)



### k3s拓展功能概览

#### Helm Controller

- k3s Helm Controller同时支持v2与v3版本
  - 从v1.17.0+k3s1 or above默认使用helm_v3
- Helm v2 vs Helm v3
  - 移除了Tiller(from SA to kubeconfig)
  - 三方会谈 (Three-way Strategic merge patch)
  - 使用Secret作为默认存储
  - crd-install hook迁移到了crds/路径等...
- Helm Controller的优势
  - 更方便的用户体验
  - HelmChart CRD 可以支持更丰富的拓展 
  

**设计原理**

1. Helm-controller 运行在master节点并list/watch HelmChart CRD对象
2. CRD onChange时执行Job更新
3. Job Container使用rancher/kilipper-helm为entrypoint
4. Killper-helm内置helm cli，可以安装/升级/删除对应的chart
   

* K3s会自动部署在/var/lib/rancher/k3s/server/manifests路径下的HelmChart
* 通过HelmChart CRD部署的chart是兼容helm v3 CLI的
* 在k3s中管理和部署Helm应用

相关工具包
    - rancher/helm-controller
    - rancher/kilipper-helm






### k3s云边协作模式

![a4bdfa2ed0fbef25adc5c819ddf132d8.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1783)


### k3s案例

![c219a97aa1e92f765e8c8c1662362845.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1784)

![db52d9567d5d3b9cbf97f98c24c4a0cf.png](evernotecid://E3BC4C29-F29D-4084-B4EA-41B1F94CFABB/appyinxiangcom/3965488/ENResource/p1785)



