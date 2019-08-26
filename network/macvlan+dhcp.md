# MacVlan

## 简介

macvlan是Linux操作系统内核提供的网络虚拟化方案之一，更准确的说法是网卡虚拟化方案。它可以为一张物理网卡设置多个mac地址（要求物理网卡打开混杂模式），针对每个mac地址，都可以设置IP地址。

macvlan 下的虚拟机或者容器网络和主机在同一个网段中，共享同一个广播域。macvlan 和 bridge 比较相似，但因为它省去了 bridge 的存在，所以配置和调试起来比较简单，而且效率也相对高。除此之外，macvlan 自身也完美支持 VLAN。

可以通过下面的命令判断内核是否支持
```
modprobe macvlan
lsmod | grep macvlan
  macvlan    19046    0
```

## 特点

- Macvlan比bridge更快
- 使用macvlan时，主机将无法通过macvlan接口与容器通信,当然可以把物理网卡也添加为macvlan子接口，实现主机和虚拟机子接口通信
- 根据物理网卡的硬件限制，可能会限制macvlan设备的数量
- 调试macvlan相关问题可能非常困难，因为它在所有内核驱动程序和物理卡上的行为可能不同

>   用 Macvlan 技术虚拟出来的虚拟网卡，在逻辑上和物理网卡是对等的。物理网卡也就相当于一个交换机，记录着对应的虚拟网卡和 MAC 地址，当物理网卡收到数据包后，会根据目的 MAC 地址判断这个包属于哪一个虚拟网卡。这也就意味着，只要是从 Macvlan 子接口发来的数据包（或者是发往 Macvlan 子接口的数据包），物理网卡只接收数据包，不处理数据包，所以这就引出了一个问题：本机 Macvlan 网卡上面的 IP 无法和物理网卡上面的 IP 通信！（Ptp可以解决）

## 应用场景

- 它的速度性能极高，在要求网络性能较高的场景下比较适用
- 如果希望容器或者虚拟机放在主机相同的网络中，实现了k8s扁平二层网络，享受已经存在网络栈的各种优势

## 工作模式

### private

过滤掉所有来自其他 macvlan 接口的报文，因此不同 macvlan 接口之间无法互相通信

### Vepa

需要主接口连接的交换机支持 VEPA/802.1Qbg 特性。所有发送出去的报文都会经过交换机，交换机作为再发送到对应的目标地址（即使目标地址就是主机上的其他 macvlan 接口），也就是 hairpin mode 模式，这个模式用在交互机上需要做过滤、统计等功能的场景。

### Bridge

通过虚拟的交换机讲主接口的所有 macvlan 接口连接在一起，这样的话，不同 macvlan 接口之间能够直接通信，不需要将报文发送到主机之外。这个模式下，主机外是看不到主机上 macvlan interface 之间通信的报文的。

VEPA 和 passthru 模式下，两个 macvlan 接口之间的通信会经过主接口两次：第一次是发出的时候，第二次是返回的时候。这样会影响物理接口的宽带，也限制了不同 macvlan 接口之间通信的速度。如果多个 macvlan 接口之间通信比较频繁，对于性能的影响会比较明显。

private 模式下，所有的 macvlan 接口都不能互相通信，对性能影响最小。

bridge 模式下，数据报文是通过内存直接转发的，因此效率会高一些，但是会造成 CPU 额外的计算量。

# IPAM-DHCP

## 特点

可以利用现有的DHCP服务器为集群Pod统一分配管理IP地址

## 原理

是一种C/S架构，需要在每个节点运行dhcp daemon，该守护进程监听socket，并支持两个命令DHCP.Allocat 和 DHCP.Release，分别用于kubelet在创建/删除Pod时使用dhcp二进制文件通过前面的socket向守护进程发起调用，完成IP地址的申请和注销。在这里，dhcp守护进程将充当dhcp客户机并向网络发送dhcp请求。

>   DHCP和DHCP Daemon使用同一个二进制文件

# macvlan+dhcp方案实施
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: dhcp-config
  namespace: kube-system
data:
  10-macvlan.conflist: |-
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "macvlan",  
          "name": "macvlannet",
          "master": "ens9",
          "mode": "bridge",
          "ipam": {     
            "type": "dhcp"
            }
        }
      ]
    }
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: dhcp-node
  namespace: kube-system
  labels:
    k8s-app: dhcp-node
spec:
  template:
    metadata:
      labels:
        k8s-app: dhcp-node
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                - key: beta.kubernetes.io/os
                  operator: NotIn
                  values:
                    - windows
      tolerations:
      - operator: Exists
        effect: NoSchedule
      - operator: Exists
        effect: NoExecute
      hostNetwork: true
      hostPID: true
      initContainers:
        - name: install-cni
          image: zdnscloud/cni-dhcp-init:v0.1
          command: ["/bin/sh","-c","/install-cni.sh"]
          volumeMounts:
          - name: dhcp-config
            mountPath: /etc/cni/net.d/
          - name: cni-dir
            mountPath: /host/etc/cni/net.d
          - name: cni-bin
            mountPath: /host/opt/cni/bin
      containers:
        - name: dhcp-node
          image: busybox
          command: ["/bin/sh","-c","/opt/cni/bin/dhcp daemon"]
          securityContext:
            privileged: true
          lifecycle:
            preStop:
                exec:
                  command: ["/bin/sh","-c","rm -f /run/cni/dhcp.sock"]
          volumeMounts:
          - name: dhcp-run
            mountPath: /run/cni/
          - name: cni-bin
            mountPath: /opt/cni/bin
      volumes:
        - name: dhcp-config
          configMap:
            name: dhcp-config
        - name: cni-dir
          hostPath:
            path: /etc/cni/net.d
        - name: cni-bin
          hostPath:
            path: /opt/cni/bin
        - name: dhcp-run
          hostPath:
            path: /run/cni/
```
>   dhcp服务器不能在集群内

>   指定的网卡既作为macvlan的父接口，又作为dhcp广播接口
