垃圾收集器在 Kubernetes 中的作用就是删除之前有所有者但是现在所有者已经不存在的对象，例如删除 ReplicaSet 时会删除它依赖的 Pod，虽然它的名字是垃圾收集器，但是它在 Kubernetes 中还是以控制器的形式进行设计和实现的。  

设计目标包括:  
* 支持服务器端级联删除。  
* 集中级联删除逻辑，而不是在控制器中扩展。  
* 允许选择性地孤立依赖对象。  
非目标包括:  
* 立即释放对象的名称，以便可以尽快重用它。  
* 在级联删除中传播宽限期。  

在 Kubernetes 引入垃圾收集器之前，所有的级联删除逻辑都是在客户端完成的，kubectl 会先删除 ReplicaSet 持有的 Pod 再删除 ReplicaSet，但是垃圾收集器的引入就让级联删除的实现移到了服务端，我们在这里就会介绍垃圾收集器的设计和实现原理。

## 概述
垃圾收集主要提供的功能就是级联删除，它向对象的 API 中加入了 metadata.ownerReferences 字段，这一字段会包含当前对象的所有依赖者，在默认情况下，如果当前对象的所有依赖者都被删除，那么当前对象就会被删除：  
```go
type ObjectMeta struct {  
	...  
	OwnerReferences []OwnerReference  
}  
  
type OwnerReference struct {  
	APIVersion string  
	Kind string  
	Name string  
	UID types.UID  
}  
```
OwnerReference 包含了足够的信息来标识当前对象的依赖者，对象的依赖者必须与当前对象位于同一个命名空间 namespace，否则两者就无法建立起依赖关系。  
通过引入 metadata.ownerReferences 能够建立起不同对象的关系，但是我们依然需要其他的组件来负责处理对象之间的联系并在所有依赖者不存在时将对象删除，这个处理不同对象联系的组件就是 GarbageCollector，也是 Kubernetes 控制器的一种。
## the Garbage Collector
如果对象的OwnerReferences中列出的所有者都不存在，则垃圾收集器负责删除对象。垃圾收集器由扫描器、垃圾处理器和传播器组成。
* 扫描器:
  * 使用discovery API检测系统支持的所有资源。
  * 定期扫描系统中的所有资源，并将每个对象添加到脏队列中。
* 垃圾处理器:
  * 由脏队列和workers组成。
  * 每个worker:
    * 从脏队列中删除项。
    * 如果项目的OwnerReferences为空，则继续处理脏队列中的下一个项目。
    * 否则检查所有者引用中的每个条目:
      * 如果存在至少一个所有者，则什么也不做。
      * 如果不存在任何所有者，则请求API服务器删除项。
* 传播器（Propagator）:
  * 传播器用于优化，而不是用于正确性。
  * 由事件队列、单个worker和所有者依赖关系的DAG组成。
    * DAG只存储name/uid/orphan三个属性，而不是每个项目的整个主体。
  * 监视所有资源的创建/更新/删除事件，将事件插入到事件队列。
  * Worker:
    * 从事件队列中删除项。
    * 如果item是创建或更新，则相应地更新DAG。
      * 如果对象有一个所有者，而该所有者还不存在于DAG中，那么除了将该对象添加到DAG之外，还将该对象排队到脏队列中。
    * 如果item是删除，则从DAG中删除该对象，并将其所有依赖对象排队到脏队列。
  * 传播器不需要执行任何RPCs，因此一个worker就足够了。这使得锁定更加容易。
  * 使用传播器，我们只需要在启动GC时运行扫描器来填充DAG和脏队列。
## 用"orphan" finalizer使后代成为孤儿  
用户可能希望在孤立依赖对象(例如pods)的同时删除拥有的对象(例如replicaset)，也就是说，保持依赖对象不变。我们通过引入"orphan" finalizer来支持这种用例。Finalizer 是一个泛型API，所以我们首先描述泛型终结器框架，然后描述"orphan" finalizer的特定设计。
### The finalizer framework

### API changes

```go
type ObjectMeta struct {
	…
	Finalizers []string
}
```
**ObjectMeta.Finalizers**:在删除对象之前需要运行的终结器列表。在从注册表中删除对象之前，此列表必须为空。列表中的每个字符串都是负责组件的标识符，用于从列表中删除条目。如果对象的deletionTimestamp为非nil，则只能删除此列表中的条目。出于安全原因，更新终结器需要特殊权限。为了强制执行准入规则，我们将终结器公开为子资源，并且在更新主资源时不允许直接更改终结器。
### 新组件
* Finalizers:
  * 就像控制器一样，终结器总是在运行。
  * 第三方可以在集群中开发并运行自己的终结器。终结器不需要在API服务器上注册。
  * 监视满足两个条件的更新事件:
    1. 更新后的对象在ObjectMeta.finalizer中具有终结器的标识符;
    2. ObjectMeta.DeletionTimestamp从nil更新到非nil.
  * 将结束逻辑应用于更新事件中的对象.
  * 执行完结束逻辑之后，将自己在ObjectMeta.Finalizers中移除.
  * 当最后一个finalizer从ObjectMeta.Finalizers删除后，API服务会把对像删除。
  * 因为结束逻辑可能被多次应用（例如，终结器在应用结束逻辑之后但在从ObjectMeta.Finalizers移除之前崩溃），结束逻辑必须是幂等的。
  * 如果终结器未能及时执行，具有适当权限的用户可以从ObjectMeta.Finalizers手动删除终结器。我们将提供kubectl命令来执行此操作。

### 对现有组件的更改

* API server:
  * Deletion handler:
    * 如果要删除的对象的`ObjectMeta.Finalizers`非空，则更新DeletionTimestamp，但不删除该对象。
    * 如果`ObjectMeta.Finalizers`为空且options.GracePeriod为零，则删除该对象。如果options.GracePeriod不为零，则只更新DeletionTimestamp。
  * Update handler:
    * 如果更新删除了最后一个终结器，并且DeletionTimestamp为非零，并且DeletionGracePeriodSeconds为零，则从注册表中删除该对象。
    * 如果更新删除了最后一个终结器，并且DeletionTimestamp为非零，但DeletionGracePeriodSeconds不为零，则只更新该对象。

### The "orphan" finalizer
### API changes

```go
type DeleteOptions struct {
	…
	OrphanDependents bool
}
```
**DeleteOptions.OrphanDependents**: 允许用户表达依赖对象是否应该是孤立的。它默认为true，因为版本1.2之前的控制器期望依赖对象成为孤立对象。

### 对现有组件的更改

* API server:
处理删除请求时，根据DeleteOptions.OrphanDependents是否为真，API服务器更新对象以向/从ObjectMeta.Finalizers map添加/删除"orphan" finalizer。

### 新组件

将第四个组件添加到垃圾收集器, the"orphan" finalizer:
* 如上所述监视更新事件[The finalizer framework](#the-finalizer-framework).
* 从其依赖项的`OwnerReferences`中删除事件中的对象。
  * 依赖对象可以通过GC保存的DAG找到，或者通过重新依赖依赖资源并检查每个潜在依赖对象的OwnerReferences字段。
* 同时删除依赖对象具有的任何悬空所有者引用。
* 最后，将自己从对象的`ObjectMeta.Finalizers`中删除。

## 实现原理
GarbageCollector 中包含一个 GraphBuilder 结构体，这个结构体会以 Goroutine 的形式运行并使用 Informer 监听集群中几乎全部资源的变动，一旦发现任何的变更事件 — 增删改，就会将事件交给主循环处理，主循环会根据事件的不同选择将待处理对象加入不同的队列，与此同时 GarbageCollector 持有的另外两组队列会负责删除或者孤立目标对象。

![""](gc.png)

接下来我们会从几个关键点介绍垃圾收集器是如何删除 Kubernetes 集群中的对象以及它们的依赖的。  

## 删除策略
多个资源的 Informer 共同构成了垃圾收集器中的 Propagator，它监听所有的资源更新事件并将它们投入到工作队列中，这些事件会更新内存中的 DAG，这个 DAG 表示了集群中不同对象之间的从属关系，垃圾收集器的多个 Worker 会从两个队列中获取待处理的对象并调用 attemptToDeleteItem 和 attempteToOrphanItem 方法，这里我们主要介绍 attemptToDeleteItem 的实现：  
```go
func (gc *GarbageCollector) attemptToDeleteItem(item *node) error {
	latest, _ := gc.getObject(item.identity)
	ownerReferences := latest.GetOwnerReferences()

	solid, dangling, waitingForDependentsDeletion, _ := gc.classifyReferences(item, ownerReferences)
```	
该方法会先获取待处理的对象以及所有者的引用列表，随后使用 classifyReferences 方法将引用进行分类并按照不同的条件分别进行处理：
```go
	switch {
	case len(solid) != 0:
		ownerUIDs := append(ownerRefsToUIDs(dangling), ownerRefsToUIDs(waitingForDependentsDeletion)...)
		patch := deleteOwnerRefStrategicMergePatch(item.identity.UID, ownerUIDs...)
		gc.patch(item, patch, func(n *node) ([]byte, error) {
			return gc.deleteOwnerRefJSONMergePatch(n, ownerUIDs...)
		})
		return err
```		
如果当前对象的所有者还有存在于集群中的，那么当前的对象就不会被删除，上述代码会将已经被删除或等待删除的引用从对象中删掉。  
当正在被删除的所有者不存在任何的依赖并且该对象的 ownerReference.blockOwnerDeletion 属性为 true 时会阻止依赖方的删除，所以当前的对象会等待属性 ownerReference.blockOwnerDeletion=true 的所有对象的删除后才会被删除。
```go
	// ...
	case len(waitingForDependentsDeletion) != 0 && item.dependentsLength() != 0:
		deps := item.getDependents()
		for _, dep := range deps {
			if dep.isDeletingDependents() {
				patch, _ := item.unblockOwnerReferencesStrategicMergePatch()
				gc.patch(item, patch, gc.unblockOwnerReferencesJSONMergePatch)				
				break
			}
		}
		policy := metav1.DeletePropagationForeground
		return gc.deleteObject(item.identity, &policy)
	// ...	
```	
在默认情况下，也就是当前对象已经不包含任何依赖，那么如果当前对象可能会选择三种不同的策略处理依赖：
```go
	// ...
	default:
		var policy metav1.DeletionPropagation
		switch {
		case hasOrphanFinalizer(latest):
			policy = metav1.DeletePropagationOrphan
		case hasDeleteDependentsFinalizer(latest):
			policy = metav1.DeletePropagationForeground
		default:
			policy = metav1.DeletePropagationBackground
		}
		return gc.deleteObject(item.identity, &policy)
	}
}
```
如果当前对象有 FinalizerOrphanDependents 终结器，DeletePropagationOrphan 策略会让对象所有的依赖变成孤立的；  
如果当前对象有 FinalizerDeleteDependents 终结器，DeletePropagationBackground 策略在前台等待所有依赖被删除后才会删除，整个删除过程都是同步的；  
默认情况下会使用 DeletePropagationDefault 策略在后台删除当前对象的全部依赖；  
终结器
对象的终结器是在对象删除之前需要执行的逻辑，所有的对象在删除之前，它的终结器字段必须为空，终结器提供了一个通用的 API，它的功能不只是用于阻止级联删除，还能过通过它在对象删除之前加入钩子：  
```go
type ObjectMeta struct {
	// ...
	Finalizers []string
}
```
终结器在对象被删之前运行，每当终结器成功运行之后，就会将它自己从 Finalizers 数组中删除，当最后一个终结器被删除之后，API Server 就会删除该对象。  

在默认情况下，删除一个对象会删除它的全部依赖，但是我们在一些特定情况下我们只是想删除当前对象本身并不想造成复杂的级联删除，垃圾回收机制在这时引入了 OrphanFinalizer，它会在对象被删除之前向 Finalizers 数组添加或者删除 OrphanFinalizer。  

该终结器会监听对象的更新事件并将它自己从它全部依赖对象的 OwnerReferences 数组中删除，与此同时会删除所有依赖对象中已经失效的 OwnerReferences 并将 OrphanFinalizer 从 Finalizers 数组中删除。  
 
通过 OrphanFinalizer 我们能够在删除一个 Kubernetes 对象时保留它的全部依赖，为使用者提供一种更灵活的办法来保留和删除对象。  

## 总结 ##
Kubernetes 中垃圾收集器的主要作用就是监听集群中对象的变更事件并根据两个字段 OwnerReferences 和 Finalizers 确定对象的删除策略，其中包括同步和后台的选择、是否应该触发级联删除移除当前对象的全部依赖；在默认情况下，当我们删除 Kubernetes 集群中的 ReplicaSet、Deployment 对象时都会删除这些对象的全部依赖，不过我们也可以通过 OrphanFinalizer 终结器删除单独的对象。

友情链接：
https://blog.gmem.cc/extend-kubernetes-with-custom-resources
https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/garbage-collection.md
https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/synchronous-garbage-collection.md


