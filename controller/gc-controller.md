# Overview

我们通常会通过删除一个Deployment的方法来同时删除Deployment对象以及所属
Pod资源对象， 而K8S中是怎么把所属资源也一并删除的呢？ 
没错，就是GC Controller发挥的作用，GC Controller会将被删除对象的附属资源查询并且一并删除。
垃圾回收就是从系统中删除未使用的对象，并释放分配给它们的计算资源。

与面向对象的语言不同，在K8s对象清单定义中，我们从来没有明确定义或编写与所有者相关的关系，而是系统如何确定该关系? 
在K8s中，每个从属对象都有一个唯一的元数据字段名称metas.ownerReferences用于关系表示。

从Kubernetes 1.8开始，K8为由特定控制器(例如ReplicaSet，StatefulSet，DaemonSet，Deployment，Job和CronJob)创建或采用的对象设置ownerReferences的值。
如果需要，还可以手动设置ownerReferences。
一个对象可以有多个ownerReferences，例如在namespace中。

此篇文档主要是讲述GC Controller的工作流程，



GC Controller 是controller-manager下的一个controller之一，主要作用是删除需要删除的对象，以及该对象的下属关系。

为什么GC 需要以一个controller 的形式去运转， 我在阅读这个代码之前一直觉得应该是让kubelet 去操作才对，直到我读了这个Design proposal: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/garbage-collection.md#overview 在这里简单翻译一下：

1. 支持服务器端Cascading deletion级联删除。简单理解一下其实是利用已经建立的父子关系(ownerreference)来删除，例如当一个pod的owner已经被删除的时候，就认为这个pod是没有人管需要删除
2. 集中级联删除逻辑，而不是在控制器中扩展。
3. 允许有选择地孤立依赖对象

# 代码理解

## API 部分

在ObjectMeta 中 引入了OwnerReferences, 去列出所有该对象依赖的对象们简称父亲们。 如果所有父亲们都被删除，那么这个对象就会被GC。

另外一个是在ObjectMeta 中 引入了Finalizers 列表，去列出所有在删除这个对象之前的终结者们。 当这个对象被彻底从集群中删除之前这个列表是必须清空。列表中的每个字符串都是负责从列表中删除条目的组件的标识符。如果该对象的deletionTimestamp为非nil，则只能删除该列表中的条目。出于安全原因，更新终结器需要特殊的特权。为了实施允许规则，我们将终结器作为子资源公开，并禁止在更新主资源时直接更改终结器。

有一个特别需要注意的事情是，OwnerReference这个Struct是没有namespace字段的，也就是说，父亲们如果是namespace scope的就必须是同一个namespace。

```go
type ObjectMeta struct {
	...
	OwnerReferences []OwnerReference
    Finalizers []string
}

type OwnerReference struct {
	// Version of the referent.
	APIVersion string
	// Kind of the referent.
	Kind string
	// Name of the referent.
	Name string
	// UID of the referent.
	UID types.UID
}
```

对API Server来说，当一个对象的`ObjectMeta.Finalizers`是非空的时候，需要更新`DeletionTimestamp`。 当`ObjectMeta.Finalizers` 非空然后`options.GracePeriod` 值是0 的时候，那么需要删除该对象， 当`options.GracePeriod`  不为0的时候，只是更新`DeletionTimestamp`



另外一个API 更改的地方是`DeleteOptions`， 引入了`OrphanDependents` ，允许用户去表示依赖对象是否应该成为孤立对象。它默认为true，因为在1.2版之前的控制器期望依赖对象成为孤儿。

```go
type DeleteOptions struct {
	…
	OrphanDependents bool
}
```



## 启动GC Controller

从以下启动代码我们可以得知，该controller的启动主要做了两个事情

1. 实例化NewGarbageCollector 
2. 启动garbage collector
3. 每30秒定期执行 Sync

```go
// 代码位置 cmd/kube-controller-manager/app/core.go
func startGarbageCollectorController(ctx ControllerContext) (http.Handler, bool, error) {
    // 如果不启动GC Controller，则直接返回退出
	if !ctx.ComponentConfig.GarbageCollectorController.EnableGarbageCollector {
		return nil, false, nil
	}

	gcClientset := ctx.ClientBuilder.ClientOrDie("generic-garbage-collector")
	discoveryClient := ctx.ClientBuilder.DiscoveryClientOrDie("generic-garbage-collector")

	config := ctx.ClientBuilder.ConfigOrDie("generic-garbage-collector")
	metadataClient, err := metadata.NewForConfig(config)
	...

	ignoredResources := make(map[schema.GroupResource]struct{})
	for _, r := range ctx.ComponentConfig.GarbageCollectorController.GCIgnoredResources {
		ignoredResources[schema.GroupResource{Group: r.Group, Resource: r.Resource}] = struct{}{}
	}
    // 实例化 NewGarbageCollector
	garbageCollector, err := garbagecollector.NewGarbageCollector(
		gcClientset,
		metadataClient,
		ctx.RESTMapper,
		ignoredResources,
		ctx.ObjectOrMetadataInformerFactory,
		ctx.InformersStarted,
	)
	// 启动garbage collector.
	workers := int(ctx.ComponentConfig.GarbageCollectorController.ConcurrentGCSyncs)
	go garbageCollector.Run(workers, ctx.Stop)

    // 每30秒定期执行 Sync
	go garbageCollector.Sync(discoveryClient, 30*time.Second, ctx.Stop)

	return garbagecollector.NewDebugHandler(garbageCollector), true, nil
}
```



### 实例化NewGarbageCollector 

GarbageCollector 的数据结构里面，有两个个队列，分别是
`attemptToDelete`： 当时机成熟时，垃圾收集器尝试删除队列attemptToDelete中的项。
`attemptToOrphan`： 垃圾收集器尝试使attemptToOrphan队列中的项的依赖项成为孤儿，然后删除这些项

`dependencyGraphBuilder` 里面也有一个隐藏的队列，那就是`graphChanges`。在介绍 `graphChanges` 这个队列之前我们先理解下`GraphBuilder` 这个Struct的作用。 

`GraphBuilder` 这个Structural其实是使用了Informer监听所有资源的增加删除修改，一旦发现之后会将对象加入`Dirty Queue` 队列。

```go
func NewGarbageCollector(...) (*GarbageCollector, error) {
	...
	attemptToDelete := workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "garbage_collector_attempt_to_delete")
	attemptToOrphan := workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "garbage_collector_attempt_to_orphan")
	absentOwnerCache := NewReferenceCache(500)
	gc := &GarbageCollector{
		metadataClient:   metadataClient,
		restMapper:       mapper,
		attemptToDelete:  attemptToDelete,
		attemptToOrphan:  attemptToOrphan,
		absentOwnerCache: absentOwnerCache,
	}
	gc.dependencyGraphBuilder = &GraphBuilder{
		eventRecorder:    eventRecorder,
		metadataClient:   metadataClient,
		informersStarted: informersStarted,
		restMapper:       mapper,
		graphChanges:     workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "garbage_collector_graph_changes"),
		uidToNode: &concurrentUIDToNode{
			uidToNode: make(map[types.UID]*node),
		},
		attemptToDelete:  attemptToDelete,
		attemptToOrphan:  attemptToOrphan,
		absentOwnerCache: absentOwnerCache,
		sharedInformers:  sharedInformers,
		ignoredResources: ignoredResources,
	}

	return gc, nil
}
```



### 启动garbage collector

开启一个新线程执行`dependencyGraphBuilder.Run`

在确保所有的监控都以及存在并且这些监控的controllers HasSynced 函数都返回true

开启一个新的线程**定时每秒**执行 ` runAttemptToDeleteWorker`

开启一个新的线程**定时每秒**执行 `runAttemptToOrphanWorker`

```go
func (gc *GarbageCollector) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer gc.attemptToDelete.ShutDown()
	defer gc.attemptToOrphan.ShutDown()
	defer gc.dependencyGraphBuilder.graphChanges.ShutDown()

    // 开启一个新线程执行dependencyGraphBuilder.Run
    go gc.dependencyGraphBuilder.Run(stopCh)

	if !cache.WaitForNamedCacheSync("garbage collector", stopCh, gc.dependencyGraphBuilder.IsSynced) {
		return
	}

	klog.Infof("Garbage collector: all resource monitors have synced. Proceeding to collect garbage")

	// gc workers
	for i := 0; i < workers; i++ {
		go wait.Until(gc.runAttemptToDeleteWorker, 1*time.Second, stopCh)
		go wait.Until(gc.runAttemptToOrphanWorker, 1*time.Second, stopCh)
	}

	<-stopCh
}

```



`runAttemptToDeleteWorker` 

该方法主要是死循环执行 `attemptToDeleteWorker` 函数。工作流程如下：

1. 从`attemptToDelete` 队列获取obj , 最后执行从队列中删除该obj

2. 获取该obj 的node 。 

3. 执行`attemptToDeleteItem`从node 节点中删除该obj。逻辑主要是获取该obj, 获取该obj的ownerreference， 如果没有ownerreference则直接返回。

   当ownerrefernce 还是存在的时候，这个obj就不会被删除。

4. 判断执行的返回错误

```go
func (gc *GarbageCollector) runAttemptToDeleteWorker() {
	for gc.attemptToDeleteWorker() {
	}
}

func (gc *GarbageCollector) attemptToDeleteWorker() bool {
	item, quit := gc.attemptToDelete.Get()
	gc.workerLock.RLock()
	defer gc.workerLock.RUnlock()
	if quit {
		return false
	}
	defer gc.attemptToDelete.Done(item)
	n, ok := item.(*node)
	...


	err := gc.attemptToDeleteItem(n)
	if err == enqueuedVirtualDeleteEventErr {		
		return true
	} else if err == namespacedOwnerOfClusterScopedObjectErr {
		// a cluster-scoped object referring to a namespaced owner is an error that will not resolve on retry, no need to requeue this node
		return true
	} else if err != nil {
		if _, ok := err.(*restMappingError); ok {			
			klog.V(5).Infof("error syncing item %s: %v", n, err)
		} else {
			utilruntime.HandleError(fmt.Errorf("error syncing item %s: %v", n, err))
		}		
		gc.attemptToDelete.AddRateLimited(item)
	} else if !n.isObserved() {
		klog.V(5).Infof("item %s hasn't been observed via informer yet", n.identity)
		gc.attemptToDelete.AddRateLimited(item)
	}
	return true
}

```







### 每30秒定期执行 Sync

```go

```










## 注意事项

今天在写代码的时候加了这一段`obj.SetOwnerReferences(append(obj.GetOwnerReferences(), ownerRef))` 到代码中，需求是希望创建keycloak client同时创建secret到不同namespace, 然后给这些secret添加ownerreference，但发现没有任何error捕获到，查了event有如下warning。

```
ownerRef [xxx/xxx, namespace: xxx, name: client, uid: 866ef543-22fb-4ec5-b253-63809d3abd22] does not exist in namespace "xx"
```

然后翻查代码后发现

```
namespace-scoped的资源添加namespace-scope级别的owner reference的时候，owner reference只能添加在同一个namespace，不能跨namespace添加； 或者可以添加owner reference为cluster-scoped级别的资源
cluster-scoped的资源只能添加cluster scope级别的owner references
```

