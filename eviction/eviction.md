Eviction
======

# Resource
## Threshold
驱逐条件配置存放在thresthold数组中，threshold具体字段如下：

	type Threshold struct {
    	Signal Signal
    	Operator ThresholdOperator
    	Value ThresholdValue
    	GracePeriod time.Duration
    	MinReclaim *ThresholdValue
	}
	
Signal为触发pod驱逐的信号，支持的Signal如下：

	// SignalMemoryAvailable is memory available (i.e. capacity - workingSet), in bytes.
    SignalMemoryAvailable Signal = "memory.available"
    // SignalNodeFsAvailable is amount of storage available on filesystem that kubelet uses for volumes, daemon logs, etc.
    SignalNodeFsAvailable Signal = "nodefs.available"
    // SignalNodeFsInodesFree is amount of inodes available on filesystem that kubelet uses for volumes, daemon logs, etc.
    SignalNodeFsInodesFree Signal = "nodefs.inodesFree"
    // SignalImageFsAvailable is amount of storage available on filesystem that container runtime uses for storing images and container writable layers.
    SignalImageFsAvailable Signal = "imagefs.available"
    // SignalImageFsInodesFree is amount of inodes available on filesystem that container runtime uses for storing images and container writeable layers.
    SignalImageFsInodesFree Signal = "imagefs.inodesFree"
    // SignalAllocatableMemoryAvailable is amount of memory available for pod allocation (i.e. allocatable - workingSet (of pods), in bytes.
    SignalAllocatableMemoryAvailable Signal = "allocatableMemory.available"
    // SignalPIDAvailable is amount of PID available for pod allocation
    SignalPIDAvailable Signal = "pid.available"
    
 
Operator 目前仅支持LessThan

ThresholdValue字段如下：

	type ThresholdValue struct {
    	Quantity *resource.Quantity
    	Percentage float32
	}
	
GracePerio 为触发驱逐的忍受时间
MinReclaim 为回收资源的最小值，即回收后满足ThresholdValue＋MinReclaim

通过函数ParseThresholdConfig获取配置的Threshold

	thresholds, err := eviction.ParseThresholdConfig(enforceNodeAllocatable, kubeCfg.EvictionHard, kubeCfg.EvictionSoft, kubeCfg.EvictionSoftGracePeriod, kubeCfg.EvictionMinimumReclaim)
	
其中参数enforceNodeAllocatable（--enforce-node-allocatable），默认为pods，要为kube组件和System进程预留资源，则需要设置为pods,kube-reserved,system-reserve

## SignalObservations
### cAdvisor
cAdvisor 是google的开源容器监控工具。只需在宿主机上部署cAdvisor容器，可通过Web界面或REST服务访问当前节点和容器的性能数据(CPU、内存、网络、磁盘、文件系统)。
### Summary
由cAdvisor收集的信息存放在Summary返回

	type Summary struct {
		Node NodeStats `json:"node"`
		Pods []PodStats `json:"pods"`
	}

	type PodStats struct {
		PodRef PodReference `json:"podRef"`
		StartTime metav1.Time `json:"startTime"`
		Containers []ContainerStats `json:"containers" patchStrategy:"merge" patchMergeKey:"name"`
		CPU *CPUStats `json:"cpu,omitempty"`
		Memory *MemoryStats `json:"memory,omitempty"`
		Network *NetworkStats `json:"network,omitempty"`
		VolumeStats []VolumeStats `json:"volume,omitempty" patchStrategy:"merge" patchMergeKey:"name"`
		EphemeralStorage *FsStats `json:"ephemeral-storage,omitempty"`
	}

	type NodeStats struct {
		NodeName string `json:"nodeName"`
		SystemContainers []ContainerStats `json:"systemContainers,omitempty" patchStrategy:"merge" patchMergeKey:"name"`
		StartTime metav1.Time `json:"startTime"`
		CPU *CPUStats `json:"cpu,omitempty"`
		Memory *MemoryStats `json:"memory,omitempty"`
		Network *NetworkStats `json:"network,omitempty"`
		Fs *FsStats `json:"fs,omitempty"`
		Runtime *RuntimeStats `json:"runtime,omitempty"`
		Rlimit *RlimitStats `json:"rlimit,omitempty"｀
	}

### signalObservation
kubelet会将cAdvisor收集的信息Summary转换成Signal对应资源信息signalObservation的map，capacity为资源总量，available为剩余资源总量，time为资源状态更新时间

	type signalObservations map[evictionapi.Signal]signalObservation

	type signalObservation struct {
	    capacity *resource.Quantity
    	available *resource.Quantity
    	time metav1.Time
	}

# Workflow
1. 每10s进行一次收集信息和检查，如果可用资源超过设置的阀值且观察时间超过了忍耐值，则进行驱逐pod且每次只需成功驱逐一个即可
2. 构建pod排序函数，用于驱逐pod时的排序，每个Signal对应一种排序函数
  * 针对内存信号(SignalMemoryAvailable、SignalAllocatableMemoryAvailable)
    * pod内存使用量超过请求值的大小
    * pod的priority
    * pod内存使用量
 
  * 针对node级别信号（SignalNodeFsAvailable、SignalNodeFsInodesFree、SignalImageFsAvailable、SignalImageFsInodesFree）
    * pod的存储使用量超过请求值的大小  
    * pod的priority
    * pod的存储使用量


3. 构造回收node级别资源的函数，每个Signal对应不同的回收策略
  * 如果imagefs 和 nodefs 在不同的 device
    * 针对nodefs信号（SignalNodeFsAvailable、SignalNodeFsInodesFree） 删除logs
    * 针对imagefs信号（SignalImageFsAvailable、SignalImageFsInodesFree） 删除没用的images
  * 如果imagefs 和 nodefs 在相同的 device，所有信号都是删除logs和没用的images
  

4. 获取所有非终结状态的pods，终结状态pods包含
  * pod 的 status.Phase 是 "Failed" 或者 "Succeeded"
  * pod 的 DeletionTimestamp 不是空
  * pod 的 status.ContainerStatus.State 包含Terminated 和 Waiting
 
 
5. 从cAdvisor获取资源使用信息Summary
6. 将Summary转换成signalObservations
7. 使用配置的thresholds，依次查看signalObservations的available，如果available已经少于配置的阀值，则将该配置的threshold添加到本轮的thresholds
8. 使用相同的方式拿到上一轮还未解决的thresholds，然后与本轮的thresholds合并
9. 设置thresholds的初次观察时间为当前时间，且只设置本轮的thresholds
10. 过滤thresholds，只保存超过忍耐时间的threshold
11. 缓存本轮的thresholds
12. 按照Signal的优先级别对thresholds进行排序，即memory的优先级别最高，包含SignalMemoryAvailable、SignalAllocatableMemoryAvailable
13. 由于每轮只驱逐一个pod，所以取thresholds第一个值，根据第一个threshold的Signal，确定超出阀值的资源类型(ResourceMemory、ResourceEphemeralStorage、resourceInodes)
14. 根据第一个threshold的Signal尝试释放node级别的资源，然后重新获取状态，再次检查是否需要释放资源，如果不需要则本轮结束
15. 根据第一个threshold的Signal拿到排序函数，对所有非终结的pods进行排序
16. 使用得到的资源类型，和获取的pod资源使用状态PodStats，构造驱逐pod的说明信息，对非终结状态的pods进行驱逐，如果成功驱逐一个就结束本轮驱逐 


