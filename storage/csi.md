# CSI设计思想
将Pod挂载PV扩展成Provision、Attach 和 Mount 三个阶段。其中

* Provision 等价于&quot;创建磁盘&quot;

* Attach 等价于&quot;挂载磁盘到虚拟机&quot;

* Mount 等价于&quot;将该磁盘格式化后，挂载在 Volume 的宿主机目录上&quot;
## Volume生命周期
![""](volume-runtime.png)

# CSI设计方案

K8S通过gRPC协议与CSI插件交互，每个SP（存储供应商，实现CSI插件的）都必须提供两类插件：

* Node plugin：在每个节点上运行,，作为一个grpc端点服务于CSI的RPCs，执行具体的挂卷操作。
* Controller Plugin：可以在任何地方运行，一般执行全局性的操作，比如创建/删除卷。

CSI有三种RPC：

* 身份服务：Node Plugin和Controller Plugin都必须实现这些RPC集。
* 控制器服务：Controller Plugin必须实现这些RPC集。
* 节点服务：Node Plugin必须实现这些RPC集。

> 为了方便管理，建议将3个服务部署在一个Container中，使用DaemonSet方式部署到所有存储节点

# CSI架构

 ![""](csi-framework.png)

## External Components

    是从 Kubernetes 项目里面剥离出来的那部分存储管理功能，由 Kubernetes 社区来开发和维护

* Driver Registrar：用于向kubelet注册Custom Components

* External Provisioner ：用于创建PV

* External Attacher：用于将PV attache到节点

实际上还有

* External snapshotter：用于管理快照

* External resizer：用于为Volume调整容量

## Custom Components（csi-driver）

    3个RPC服务，是由第三方负责开发和维护

* Identity Service：负责对外暴露这个插件本身的信息

* Controller Service：用于创建、删除以及管理 CSI Volume

* Node Service：用于将 Volume存储卷挂载到指定的目录中

# CSI流程总结

 1. Driver Registrar调用Identity Service的GetPluginInfo方法后向kubelet注册csi-driver，便于后面kubelet调用
 2. 用户创建PVC，引用storageClass，storageClass所指向的External
    Provisioner监听到对应的PVC事件，开始调用Controller Service的CreateVolume/DeleteVolume方法完成PV的创建/删除。
 3. Pod使用该PVC和PV，并被调度到Node节点 
 4. Node节点上Kubelet看到该Pod使用了PV，其Volume Manager（通过CSI-Plugin）会创建一个volumeattachments，并等待其attached状态变化
 5. External Attacher监听volumeattachments，调用Controller Service的ControllerPublishVolume/ControllerUnpublishVolume方法完成Attach/Detach Node节点上Kubelet的volume
 6. Manager观察到attached状态变为true后，调用Node Service的NodeStageVolume和NodePublishVolume方法完成Mount/Unmount

# csi-driver

https://github.com/container-storage-interface/spec/blob/master/spec.md#rpc-interface

需要定义三个service（RPC集合）：identity、controller、node
根据不同的Capability实现其对应的接口功能，然后其他程序调用identity/GetPluginCapabilities就可以知道该csi-driver能够做什么了

## 公共部分

https://github.com/kubernetes-csi/drivers/tree/master/pkg/csi-common

    k8s实现了一个官方的公共代码，公共代码实现了CSI要求的RPC方法，我们自己开发的插件可以继承官方的公共代码，然后把自己要实现的部分方法进行覆盖即可

## identity

**GetPluginInfo**

        必须实现的。获取csi-driver的版本和名称等信息

        注：Name字段需要使用反向域名表示法

**Probe**

        必须实现的。获取csi-driver的健康和就绪状态

        注：可以返回一个空值

**GetPluginCapabilities**

        必须实现的。获取csi-driver的总的可用功能
        分为两大类
         - Service
                CONTROLLER_SERVICE
                VOLUME_ACCESSIBILITY_CONSTRAINTS
         - VolumeExpansion
                ONLINE
                OFFLINE

## controller

CreateVolume

        如果ControllerServiceCapability要实现CREATE_DELETE_VOLUME，那么这个方法必须实现，用来完成卷的创建

DeleteVolume

        如果ControllerServiceCapability要实现CREATE_DELETE_VOLUME，那么这个方法必须实现，用来完成卷的删除

ControllerPublishVolume

        如果ControllerServiceCapability要实现PUBLISH_UNPUBLISH_VOLUME，那么这个方法必须实现，用来完成卷的附加

ControllerUnpublishVolume

        如果ControllerServiceCapability要实现PUBLISH_UNPUBLISH_VOLUME ，那么这个方法必须实现，用来完成卷的分离

**ValidateVolumeCapabilities**

        用来检查卷具有的VolumeCapabilities是否能满足所需的所有VolumeCapabilities功能。一般是kubelet执行NodePublishVolume/NodeStageVolume失败后会调用这个方法验证一下。

ListVolumes

        如果ControllerServiceCapability要实现LIST_VOLUMES，那么这个方法必须实现，用来展示所有卷的信息（支持分页）。一般用于当CreateVolume超时后且K8S不再需要这个卷，K8S有三种方案：
        - 重新执行CreateVolume，成功后得到Volume ID，进行DleteVolume
        - 执行ListVolumes获取，进行DleteVolume
        - 不做任何操作，由管理员手动进行

GetCapacity

        如果ControllerServiceCapability要实现GET_CAPACITY，那么这个方法必须实现，用来返回存储池的可用容量

**ControllerGetCapabilities**

        必须实现的。用来返回control service的所拥有的功能

CreateSnapshot

        如果ControllerServiceCapability要实现CREATE_DELETE_SNAPSHOT，那么这个方法必须实现，用来为一个源卷创建快照

DeleteSnapshot

        如果ControllerServiceCapability要实现CREATE_DELETE_SNAPSHOT，那么这个方法必须实现，用来删除快照

ListSnapshots

        如果ControllerServiceCapability要实现LIST_SNAPSHOTS ，那么这个方法必须实现，用来展示所有快照的信息。与ListVolumes类似

ControllerExpandVolume

        如果ControllerServiceCapability要实现EXPAND_VOLUME ，那么这个方法必须实现，用来给卷扩容/缩容

## node

NodeStageVolume

        如果NodeServiceCapability要实现STAGE_UNSTAGE_VOLUME ，那么这个方法必须实现，完成卷的临时挂载。这么做的目的是K8S允许多个Pod使用同一个卷。
        
        注：如果有需要这个功能，则格式化需要在这里进行

NodeUnstageVolume

        如果NodeServiceCapability要实现STAGE_UNSTAGE_VOLUME ，那么这个方法必须实现，完成卷的临时卸载

**NodePublishVolume**

        必须实现的。完成卷的挂载

**NodeUnpublishVolume**

        必须实现的。完成卷的卸载

NodeGetInfo

        如果ControllerServiceCapability要实现PUBLISH_UNPUBLISH_VOLUME ，那么这个方法必须实现，返回此插件节点的ID和Topology信息

**NodeGetCapabilities**

        必须实现的。用来返回node service的所拥有的功能

NodeGetVolumeStats

        如果NodeServiceCapability要实现GET_VOLUME_STATS ，那么这个方法必须实现，用来展示卷的容量统计信息

        注：暂时不知道如果被调用

NodeExpandVolume

        如果NodeServiceCapability要实现EXPAND_VOLUME ，那么这个方法必须实现，完成卷的扩容/缩容。

        注：暂时不知道如果被调用

## 能力列表

https://github.com/container-storage-interface/spec/lib/go/csi/csi.pb.go

### UNKNOWN

### CONTROLLER\_SERVICE

 - ControllerServiceCapability\_RPC\_UNKNOWN
   
 - ControllerServiceCapability\_RPC\_CREATE\_DELETE\_VOLUME

 - ControllerServiceCapability\_RPC\_PUBLISH\_UNPUBLISH\_VOLUME

 - ControllerServiceCapability\_RPC\_LIST\_VOLUMES

 - ControllerServiceCapability\_RPC\_GET\_CAPACITY

 - ControllerServiceCapability\_RPC\_CREATE\_DELETE\_SNAPSHOT

 - ControllerServiceCapability\_RPC\_LIST\_SNAPSHOTS

 - ControllerServiceCapability\_RPC\_CLONE\_VOLUME

 - ControllerServiceCapability\_RPC\_PUBLISH\_READONLY

 - ControllerServiceCapability\_RPC\_EXPAND\_VOLUME

### VOLUME\_ACCESSIBILITY\_CONSTRAINTS

 - VolumeCapability\_AccessMode\_UNKNOWN
 
 - VolumeCapability\_AccessMode\_SINGLE\_NODE\_WRITER
 
 - VolumeCapability\_AccessMode\_SINGLE\_NODE\_READER\_ONLY
 
 - VolumeCapability\_AccessMode\_MULTI\_NODE\_READER\_ONLY
 
 - VolumeCapability\_AccessMode\_MULTI\_NODE\_SINGLE\_WRITER
 
 - VolumeCapability\_AccessMode\_MULTI\_NODE\_MULTI\_WRITER

### NODE\_SERVICE

 - NodeServiceCapability\_RPC\_UNKNOWN
 
 - NodeServiceCapability\_RPC\_STAGE\_UNSTAGE\_VOLUME
 
 - NodeServiceCapability\_RPC\_GET\_VOLUME\_STATS
 
 - NodeServiceCapability\_RPC\_EXPAND\_VOLUME

# driver-registrar

https://github.com/kubernetes-csi/node-driver-registrar

## 作用

- 向Kubelet注册CSI驱动程序。因为kubelet调用CSI的方法时需要知道向哪个套接字发出调用
- 将CSI驱动程序自定义NodeId添加到Kubernetes Node API对象上的标签。便于后面ControllerPublishVolume调用能够获取到nodeid与csi-driver的映射关系

## 逻辑

 1. 通过CSI driver socket **调用csi-driver/identity的GetPluginInfo**函数，获取CSI驱动的名称
 2. 通过Registration socket，向kubelet注册CSI驱动程序，Kubelet的plugin watcher观察到这个socket之后开始注册，**调用csi-driver/node的NodeGetInfo**，获取节点的ID用于给节点打标签

## 源码

cmd/csi-node-driver-registrar/main.go

    131 csiDriverName, err := csirpc.GetDriverName(ctx, csiConn)

github.com/kubernetes-csi/csi-lib-utils/rpc/common.go

    43 rsp, err := client.GetPluginInfo(ctx, &req)

cmd/csi-node-driver-registrar/node\_register.go

    37 registrar := newRegistrationServer(csiDriverName, *kubeletRegistrationPath, supportedVersions)

    62 grpcServer := grpc.NewServer()
    
    64 registerapi.RegisterRegistrationServer(grpcServer, registrar)
    
k8s.io/kubernetes/pkg/kubelet/apis/pluginregistration/v1alpha1/api.pb.go
    
    214 func RegisterRegistrationServer(s *grpc.Server, srv RegistrationServer) {
    215         s.RegisterService(&_Registration_serviceDesc, srv)
    216 }
    
github.com/kubernetes/pkg/volume/csi/csi\_plugin.go

    154 driverNodeID, maxVolumePerNode, accessibleTopology, err := csi.NodeGetInfo(ctx)


github.com/kubernetes/pkg/volume/csi/nodeinfomanager/nodeinfomanager.go

    113 updateNodeIDInNode(driverName, driverNodeID),
    
    274 node.ObjectMeta.Annotations[annotationKeyNodeID] = string(jsonObj)

> 注：
> 
> 打标签有两种模式，一个模式是自己给 node 打上这个 annotation，并且在退出的时候把这个 annotation去掉。另一个模式是交给 kubelet 的 pluginswatcher 来管理， kubelet 自己会根据node-driver-registrar 提供的 socket 然后调用 gRPC 从 registrar 获取 NodeId 和DriverName 自己把 annotation 打上。

# external-provisioner

## 作用

根据用户的PVC请求创建/删除PV，完成Provision和Delete。

## 逻辑

    用户创建PVC引用storageclass，storageclass的provisioner属性会让kubernetes在创建PVC时指定annotations（volume.beta.kubernetes.io/storage-provisioner
provisioner监听Kube-API中的PVC对象，执行其Provision函数

1. **调用csi-driver/identity的GetPluginCapabilities**
2. **调用csi-driver/controller的ControllerGetCapabilities**
3. 判断其是否拥有PluginCapability\_CONTROLLER\_SERVICE和ControllerCapability\_CREATE\_DELETE\_VOLUME，如果任意一个功能没有则结束provision，判断ControllerCapability\_CREATE\_DELETE\_SNAPSHOT功能是否需要，如果前面两个功能都有，则
4. **调用csi-driver/identity的GetPluginInfo**** ，获取csi-driver的相关信息
5. 再根据PVC和Storageclass的属性组成CreateVolumeRequest对象，传递并**调用csi-driver/controller的CreateVolume/DeleteVolume** 函数返回CreateVolumeResponse对象，根据返回信息组成pv并返回，然后kube-apiserver 中的 VolumeController 的 PersistentVolumeController进行创建和绑定PVC。

## 源码

github.com/kubernetes-incubator/external-storage/lib/controller/controller.go

    1015 volume, err = ctrl.provisioner.Provision(options)
    
    1042 if _, err = ctrl.client.CoreV1().PersistentVolumes().Create(volume); err == nil || apierrs.IsAlreadyExists(err) {

external-provisioner/pkg/controller/controller.go

    169 rsp, err := client.GetPluginInfo(ctx, &req)
    
    232 rsp, err := client.GetPluginCapabilities(ctx, &req)
    
    246 rsp, err := client.ControllerGetCapabilities(ctx, &req)
    
    317 func makeVolumeName(prefix, pvcUID string, volumeNameUUIDLength int) (string, error) {
    
    354 driverState, err := checkDriverState(p.grpcClient, p.timeout, needSnapshotSupport)
    
    481 rep, err = p.csiClient.CreateVolume(ctx, &req)
    
    651 _, err = p.csiClient.DeleteVolume(ctx, &req)

> 注：可以部署多个provisioner，但只能有一个provisioner领导者。可以在启动时指定--enable-leader-election开启选举。
--volume-name-prefix参数可以指定PV的名称前缀（默认pvc-&lt;uuid&gt;）

# Volume-Manager

## 作用

调用in-tree 的 CSI-Plugin创建VolumeAttachment，并使用WaitForAttach等待其状态变为true后再通过in-tree 的 CSI-Plugin 调用csi-driver/node的NodeStageVolume和NodePublishVolume方法完成Mount

## 逻辑

kubelet 有一个 volume manager 来管理 volume 的 mount/attach 操作。volume Manager 有两个 goroutine ，一个是同步状态，一个 reconciler.reconcile。

 - desiredStateOfWorld：是从 podManager 同步的理想状态
 - actualStateOfWorld：是目前 kubelet 的上运行的 pod 的状态

每次 volume manager 需要把 actualStateOfWorld 中 volume 的状态同步到 desired 指定的状态。

reconcile方法先后分别调用in-tree 的 CSI plugin 的 Attach、WaitForAttach
当WaitForAttach满足后调用in-tree 的 CSI plugin 的 MountDevice和SetUp方法

## 源码

github.com/kubernetes/pkg/kubelet/volumemanager/volume\_manager.go

    240 go vm.desiredStateOfWorldPopulator.Run(sourcesReady, stopCh)
    
    244 go vm.reconciler.Run(stopCh)

github.com/kubernetes/pkg/kubelet/volumemanager/reconciler/reconciler.go

    156 func (rc reconciler) reconcile() {
    
    214 err := rc.operationExecutor.AttachVolume(volumeToAttach, rc.actualStateOfWorld)
    
    234 err := rc.operationExecutor.MountVolume(
    235         rc.waitForAttachTimeout,
    236         volumeToMount.VolumeToMount,
    237         rc.actualStateOfWorld,
    238         isRemount)

github.com/kubernetes/pkg/volume/util/operationexecutor/operation\_executor.go

    598 func (oe *operationExecutor) AttachVolume(
    
    602 oe.operationGenerator.GenerateAttachVolumeFunc(volumeToAttach, actualStateOfWorld)
    
    721 func (oe *operationExecutor) MountVolume(
    
    731  if fsVolume {

    734  generatedOperations = oe.operationGenerator.GenerateMountVolumeFunc(
    735  waitForAttachTimeout, volumeToMount, actualStateOfWorld, isRemount)
    737         } else {
    740             generatedOperations, err = oe.operationGenerator.GenerateMapVolumeFunc(
    741             waitForAttachTimeout, volumeToMount, actualStateOfWorld)
    742         }

github.com/kubernetes/pkg/volume/util/operationexecutor/operation\_generator.go

    294 func (og *operationGenerator) GenerateAttachVolumeFunc(
    
    348 devicePath, attachErr := volumeAttacher.Attach(
    
    349 volumeToAttach.VolumeSpec, volumeToAttach.NodeName)
    
    520 func (og *operationGenerator) GenerateMountVolumeFunc(
    
    595 devicePath, err = volumeAttacher.WaitForAttach(
    
    620 err = volumeDeviceMounter.MountDevice(
    
    662 mountErr := volumeMounter.SetUp(fsGroup)

# external-attacher

## 作用

将PV附加/分离Node节点，完成Attach和Detach

> 这在云环境中很常见，在云环境中，云API能够将卷附加到节点上，而不需要在节点上运行任何代码。

## 逻辑

    用户创建的Pod在调度到Node节点之后，kubelet会创建一个VolumeAttachment资源

Attacher监听Kube-API中的volumeattachments对象，然后

 1. **调用csi-driver/identity的Probe**判断其健康状态，如果正常则
 2. **调用csi-driver/identity的GetPluginInfo** 得到其名称版本信息，如果得到则
 3. **调用csi-driver/identityGetPluginCapabilities** ，判断csi-driver是否有PluginCapability\_Service\_CONTROLLER\_SERVICE能力，如果没有就使用TrivialHandler处理attach。如果有则
 4. **调用csi-driver/controller的ControllerGetCapabilities**判断其是否有ControllerServiceCapability\_RPC\_PUBLISH\_UNPUBLISH\_VOLUME能力，如果没有就使用TrivialHandler处理attach，如果有就用CSIHandler处理attach。
    使用两种不同的handler，都会执行SyncNewOrUpdatedVolumeAttachment ，最终将volumeattachment的attached置为true。
 
    CSIHandler：先**调用csi-driver/controller的ControllerPublishVolume/ ControllerUnpublishVolume**函数，再调用markAsAttached将VolumeAttachment.Attached重置为true

    TrivialHandler：直接调用markAsAttached，将VolumeAttachment.Attached重置为true


## 源码

external-attacher/cmd/csi-attacher/main.go

    130 supportsService, err := csiConn.SupportsPluginControllerService(ctx)
    
    140 supportsAttach, err := csiConn.SupportsControllerPublish(ctx)
    
    151 handler = controller.NewCSIHandler(clientset, csiClientset, attacher, csiConn, pvLister, nodeLister, nodeInfoLister, vaLister, timeout)
    
    154 handler = controller.NewTrivialHandler(clientset)
pkg/controller/csi_handler.go

    90 func (h *csiHandler) SyncNewOrUpdatedVolumeAttachment(va storage.VolumeAttachment) {
    
    119 va, metadata, err := h.csiAttach(va)
    
    134 if _, err := markAsAttached(h.client, va, metadata); err != nil {
    
    321 publishInfo, _, err := h.csiConnection.Attach(ctx, volumeHandle, readOnly, nodeID, volumeCapabilities, attributes, secrets)

pkg/connection/connection.go

    124 rsp, err := client.GetPluginInfo(ctx, &req)
    
    140 _, err := client.Probe(ctx, &req)
    
    175 rsp, err := client.GetPluginCapabilities(ctx, &req)
    
    151 rsp, err := client.ControllerGetCapabilities(ctx, &req)
    
    195 func (c *csiConnection) Attach(ctx context.Context, volumeID string, readOnly bool, nodeID string, caps *csi.VolumeCapability, context, secrets map[string]string) (metadata map[string]string, detached bool, err error) {
    
    207 rsp, err := client.ControllerPublishVolume(ctx, &req)
    
    223 _, err = client.ControllerUnpublishVolume(ctx, &req)

pkg/controller/trivial\_handler.go

    47 func (h *trivialHandler) SyncNewOrUpdatedVolumeAttachment(va *storage.VolumeAttachment) {
    
    51 if _, err := markAsAttached(h.client, va, nil); err != nil {

注：

> 可以部署多个attacher，但只能有一个attacher领导者。可以在启动时指定--leader-election开启选举
> 文件存储就不需要attache，因为不需要绑定设备到节点上，直接使用网络接口就可以了

# CSI-Plugin（In-tree）

## 作用

真正调用csi-driver的方法，完成Volume的mount/unmount。

## 逻辑

1. Attach方法创建一个volumeattachment资源
2. WaitForAttach方法等待volumeattachment的attached状态变化
3. MountDevice方法先调用 **csi-driver/node的NodeGetCapabilities** 方法，判断是否有NodeServiceCapability\_RPC\_STAGE\_UNSTAGE\_VOLUME功能，如果没有则结束MountDevice，如果有，则
4. **调用csi-driver/node的NodeStageVolume**方法完成临时挂载
5. SetUp方法先调用 **csi-driver/node的NodeGetCapabilities** 方法，判断是否有NodeServiceCapability\_RPC\_STAGE\_UNSTAGE\_VOLUME功能，如果有则填充deviceMountPath作为下面的一个参数
6. 再**调用csi-driver/node的NodePublishVolume**方法完成挂载

## 源码

github.com/kubernetes/pkg/volume/csi/csi\_attacher.go

    60 func (c *csiAttacher) Attach(spec *volume.Spec, nodeName types.NodeName) (string, error) {
    
    76 attachment := &storage.VolumeAttachment{
    77                  ObjectMeta: meta.ObjectMeta{
    78                          Name: attachID,
    79                          },
    80                   Spec: storage.VolumeAttachmentSpec{
    81                         NodeName: node,
    82                         Attacher: pvSrc.Driver,
    83                         Source: storage.VolumeAttachmentSource{
    84                                 PersistentVolumeName: &pvName,
    85                         },
    86                 },
    87         }
    88
    89         _, err = c.k8s.StorageV1().VolumeAttachments().Create(attachment)
    
    269 func (c *csiAttacher) MountDevice(spec *volume.Spec, devicePath string, deviceMountPath string) (err error) {
    370         err = csi.NodeStageVolume(ctx,
    371                 csiSource.VolumeHandle,
    372                 publishContext,
    373                 deviceMountPath,
    374                 fsType,
    375                 accessMode,
    376                 nodeStageSecrets,
    377                 csiSource.VolumeAttributes)
github.com/kubernetes/pkg/volume/csi/csi\_mounter.go

    95 func (c *csiMountMgr) SetUp(fsGroup *int64) error {
    96         return c.SetUpAt(c.GetPath(), fsGroup)
    97 }
    98
    99 func (c *csiMountMgr) SetUpAt(dir string, fsGroup *int64) error {
    
    243         err = csi.NodePublishVolume(
    244                 ctx,
    245                 volumeHandle,
    246                 readOnly,
    247                 deviceMountPath,
    248                 dir,
    249                 accessMode,
    250                 publishContext,
    251                 volAttribs,
    252                 nodePublishSecrets,
    253                 fsType,
    254                 mountOptions,
    255         )

> 注：
> 
> 文件系统类型的存储（NFS/GlusterFS等），只需要NodePublishVolume一步即可

# 其他

## external-snapshotter
https://github.com/kubernetes-csi/external-snapshotter

卷快照是K8S中另外一种存储资源类型,3个标准资源VolumeSnapshot、VolumeSnapshotContent、VolumeSnapshotClass，它们与PersistentVolumeClaim和PersistentVolume以及storageClass的结构类似。

external-snapshotter用来监听VolumeSnapshot和VolumeSnapshotClass，**调用csi-driver的ControllerGetCapabilities**检查csi-driver是否具有CREATE_DELETE_SNAPSHOT功能，如果有则**调用csi-driver/controller的CreateSnapshot和DeleteSnapshot** 完成快照的创建和删除。

external-snapshotter/cmd/csi-snapshotter/main.go

    150         snapshotterName, err = csirpc.GetDriverName(ctx, csiConn)
    
    165         supportsCreateSnapshot, err := supportsControllerCreateSnapshot(ctx, csiConn)
    
    235         capabilities, err := csirpc.GetControllerCapabilities(ctx, conn)

external-snapshotter/pkg/snapshotter/snapshotter.go

    76         rsp, err := client.CreateSnapshot(ctx, &req)
    
    97         if _, err := client.DeleteSnapshot(ctx, &req); err != nil

## external-resizer
https://github.com/kubernetes-csi/external-resizer

监听API对PVC的编辑，当容量大小发生变化时， **调用csi-driver/controller的ControllerExpandVolume**完成容量调整。

### 版本要求
    CSI:v1.1.0
    
    K8S:v1.14
### K8S开启功能

    --feature-gates=ExpandCSIVolumes=true
    
    --feature-gates=ExpandInUsePersistentVolumes=true

### csi-driver增加plugin功能

    {
            Type: &csi.PluginCapability_VolumeExpansion_{
                    VolumeExpansion: &csi.PluginCapability_Service{
                            Type: csi.PluginCapability_VolumeExpansion_ONLINE,
                    },
            },
    },
    {
            Type: &csi.PluginCapability_VolumeExpansion_{
                    VolumeExpansion: &csi.PluginCapability_Service{
                            Type: csi.PluginCapability_VolumeExpansion_OFFLINE,
                    },
            },
    },

external-resizer/pkg/resizer/csi\_resizer.go

    72         supportControllerService, err := supportsPluginControllerService(csiClient, timeout)
    
    81         supportControllerResize, err := supportsControllerResize(csiClient, timeout)
    
    87                 supportsNodeResize, err := supportsNodeResize(csiClient, timeout)
    
    166         newSizeBytes, nodeResizeRequired, err := r.client.Expand(ctx, volumeID, requestSize.Value(), secrets)

github.com/kubernetes-csi/csi-lib-utils/rpc/common.go

    43         rsp, err := client.GetPluginInfo(ctx, &req)
    
    61         rsp, err := client.GetPluginCapabilities(ctx, &req)
    
    87         rsp, err := client.ControllerGetCapabilities(ctx, &req)

external-resizer/pkg/csi/client.go

    99         rsp, err := c.nodeClient.NodeGetCapabilities(ctx, &csi.NodeGetCapabilitiesRequest{})
    
    128         resp, err := c.ctrlClient.ControllerExpandVolume(ctx, req)


