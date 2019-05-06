CSI流程
=========
1.	Driver Registrar调用Identity Service的GetPluginInfo方法后向kubelet注册csi-driver，便于后面kubelet调用 <br />  
2.	用户创建PVC，引用storageClass，storageClass所指向的External Provisioner监听到对应的PVC事件，开始调用Controller Service的CreateVolume/DeleteVolume方法完成PV的创建/删除。 <br />  
3.	Pod使用该PVC和PV，并被调度到Node节点 <br />  
4.	Node节点上Kubelet看到该Pod使用了PV，其volume Manager（通过in-tree的CSI-Plugin）会创建一个volumeattachments，并等待其attached状态变化 <br />  
5.	External Attacher监听volumeattachments，调用Controller Service的ControllerPublishVolume/ ControllerUnpublishVolume方法完成Attach/Detach <br />  
6.	Node节点上Kubelet的volume Manager观察到attached状态变为true后，调用Node Service的NodeStageVolume  和NodePublishVolume 方法完成Mount/Unmount <br />  
 
