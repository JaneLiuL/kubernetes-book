# 概述

之前写过的DaemonSet controller, StatefulSet controller阅读理解，实际上只是存在于etcd的一个数据，而真正的把Pod资源在一个节点上启动，是离不开Kubelet组件的。Kubelet组件可以说是最后的执行者。



我们首先来看一个Pod被创建的流程：

1. 用户使用`Kubectl` 创建了一个Replicas 
2. Replicas Controller根据提供的数据创建了一组Pod资源
3. Scheduler监听到Pod资源的变动，把该Pod对象加到调度器队列中等待调度
4. Kubelet 监听到Pod资源的变动，但由于Pod对象的NodeName 为空，则跳过
5. Scheduler使用一系列算法，预选打分，为该Pod选取一个最优节点Node，并且更改Pod对象，添加.Status.NodeName字段的值
6. Kubelet 监听到Pod的变动，这个时候Pod对象的NodeName 是自身所在的Node，进入Kubelet 循环



# 问题

从Pod启动讲述下Kubelet具体的执行流程？

为什么一个Pod里面的几个container的网络、存储是可以互相通？

从Pod删除讲述下Kubelet具体的执行流程？

Kubelet是怎么上报Node自身的资源等数据？

Kubelet是如何执行垃圾回收？





# 我想象中的设计

Kubelet 需要统筹Pod的生命周期，需要考虑到服务、网络、存储、GC、日志、Metrics。

1. 提供一个HTTP Server，暴露自身的数据，例如Pod的信息，自身的health check

2. 存储方面

   1. 需要管理Volume的组件，管理Volume在Node节点的状态，以及上报状态
   2. 需要管理Image的组件，Image的size， GC等
   3. 需要管理Container的组件，GC等

3. 网络方面

   1. 需要上报Node的地址
   2. 需要管理Pod的地址
   3. 需要管理Iptables/IPVS的组件

4. 监控

   1. 需要监控并且上报Pod状态
   2. 需要监控Node状态，包括CPU/Memory/Process

5. 日志

   1. Pod/Container日志
   
   
   
   

# 目录结构

```bash
tree kubernetes/pkg/kubelet/
```





| Folder            | Description                                     |
| ----------------- | ----------------------------------------------- |
| apis              |                                                 |
| cadvisor          |                                                 |
| certificate       |                                                 |
| checkpointmanager |                                                 |
| client            |                                                 |
| cloudresource     |                                                 |
| cm                |                                                 |
| config            |                                                 |
| configmap         |                                                 |
| container         |                                                 |
| cri               |                                                 |
| custommetrics     |                                                 |
| dockershim        | CRI 部分， dockershim作为docker的CRI 接口的实现 |
| envvars           |                                                 |
| events            |                                                 |
| eviction          | kubelet node节点将Pod驱逐                       |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |
|                   |                                                 |



## <加个架构图>





# Run

QA: 等会找找创建Pod的channel  updates的数据从哪里来

```go
// 代码位置 `pkg/kubelet/kubelet.go`
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	// 启动 cloudResourceSyncManager
	if kl.cloudResourceSyncManager != nil {
		go kl.cloudResourceSyncManager.Run(wait.NeverStop)
	}
	// 初始化container runtime之外的其他models
	if err := kl.initializeModules(); err != nil {
	..
	}

	// 启动 volume manager
	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

	if kl.kubeClient != nil {		
        // 启动 syncNodeStatus去同步节点状态
		go wait.Until(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, wait.NeverStop)
		go kl.fastStatusUpdateOnce()

		// 启动 syncing lease
		go kl.nodeLeaseController.Run(wait.NeverStop)
	}
    // 启动container runtime模块
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

	// 设置 iptables规则
	if kl.makeIPTablesUtilChains {
		kl.initNetworkUtil()
	}

    // 启动podKiller 
	go wait.Until(kl.podKiller.PerformPodKillingWork, 1*time.Second, wait.NeverStop)

	
    // 启动statusManager 和 probeManager
	kl.statusManager.Start()
	kl.probeManager.Start()

	// Start syncing RuntimeClasses if enabled.
	if kl.runtimeClassManager != nil {
		kl.runtimeClassManager.Start(wait.NeverStop)
	}
	
    // pleg是Kubelet最核心的模块，启动pleg模块
	kl.pleg.Start()
	kl.syncLoop(updates, kl)
}
```



# 模块

PLEG(Pod Lifecycle Event Generator）核心模块 : call container runtime-→get container info→compare pod cache→send info to eventChannel

syncLop(consumer)-→consum eventChannel→sync Pod

evictionManager -->驱逐pod保证集群文档(when disk/memory presssure) 按posClass顺序驱逐

probeManager --> monitor container health, if livenessProbe fail, kubelet kill pod

statusManager  --> maintain status and update to api server (status info come from like probeManager )

imageManager-->包括pull image等去配合运行Pod

volumeManager-->挂载/卸载卷等操作

containerManager-->负责cgroup配置信息

runtimerManager -->底层使用docker/rkt就是runtime实现对接

podManager-->访问Pod信息

imageGC-->回收image

containerGC-->清理container

containerRefManager --> report container info like fail, map containerID and v1.ObjectReference

cAdvisor 模块 → monitor container info

OOMWatcher →Watch cAdvisor and generate Event





# SyncLoop

为什么需要两个定时器呢，是因为我们除了要监听Pod的消息，也就是当用户Add/Delete/Modify Pod的消息之外，我们还需要保持自身kubelet的工作，去比对Pod跟节点Node的Pod的区别，例如当etcd中没有了Pod的数据，可是Node节点上却有该Pod数据，则需要清理Pod。

```go
// 代码位置pkg/kubelet/kubelet.go
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	klog.Info("Starting kubelet main sync loop.")	
    // 执行了两个定时器，一个是syncTicker  一个是housekeepingTicker
	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()
    
    // pleg.Watch()返回的是一个channel, 这个channel的数据是从pleg的event录入的
	plegCh := kl.pleg.Watch()
	const (
		base   = 100 * time.Millisecond
		max    = 5 * time.Second
		factor = 2
	)
	duration := base
	
	if kl.dnsConfigurer != nil && kl.dnsConfigurer.ResolverConfig != "" {
		kl.dnsConfigurer.CheckLimitsForResolvConf()
	}
    	for {	
		duration = base

		kl.syncLoopMonitor.Store(kl.clock.Now())
         // 执行最重要的逻辑，处理plegCh channel里面的数据
		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```



SyncLoop实际上是无限循环做了一个最重要的逻辑：`syncLoopIteration`

## syncLoopIteration

从下方的代码可以看出，Kubelet的设计是真的非常精湛，他并非使用了A call B, B call C, C call D这种方法去让整个循环保持状态，而是解耦了各大模块，在kubelet 的Run的时候，就让各个模块使用go协程各自做各自的事情，然后写到channel里面，在syncLoopIteration里面，去读取channel里面的事件处理。

```go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
     // configCh 这个channel是监听来自例如apiserver的变化，当从apiserver中的pod有被增删改，那么这个channel就会有具体Pod的更新操作
	case u, open := <-configCh:
		...
        switch u.Op {
            // 如果是创建Pod的操作，那么调用HandlePodAdditions去执行Pod的创建
		case kubetypes.ADD:
			klog.V(2).Infof("SyncLoop (ADD, %q): %q", u.Source, format.Pods(u.Pods))			
			handler.HandlePodAdditions(u.Pods)
     // plegCh 这个channel是来自pleg 模块的消息，也就是当容器状态发生变化则会写入这个chanenl， 那么我们从这个channel就是接受容器状态变化的事件   
	case e := <-plegCh:
		...
     // 定时，这个是一个保存pod状态的Channel, 类似cache一样去保存pod现在的状态   
	case <-syncCh:
		...
    // pod健康检查如果有问题，那么就执行重启等操作            
    case update := <-kl.livenessManager.Updates():            
         ...            
     // housekeeping是负责清理pod   
	case <-housekeepingCh:
		...
	}
	return true
}
```

### 清理Pod:  housekeepingCh

我们从Kubelet `syncLoopIteration` 的`housekeepingCh` channel开始跟着Pod清理的路线开始跟下去。 从代码可以看出，`housekeepingCh` 这条channel是定时在`SyncLoop`函数被触发的`time.NewTicker(housekeepingPeriod)`，然后跟着`HandlePodCleanups`看看具体清理Pod的操作

```go
func (kl *Kubelet) syncLoopIteration(housekeepingCh <-chan time.Time...) bool {	
	case <-housekeepingCh:
		...
			if err := handler.HandlePodCleanups(); err != nil {
				klog.Errorf("Failed cleaning pods: %v", err)
			}		
	}
```

HandlePodCleanups执行一系列的清理工作，包括Terminate Pod，杀死不需要的Pod，并删除孤立的卷/Pod/ 录。、

该函数是被定时触发的。

```go
// 代码位置 pkg/kubelet/kubelet_pods.go
func (kl *Kubelet) HandlePodCleanups() error {	
	...
    // 获取本节点上的所有pod
	allPods, mirrorPods := kl.podManager.GetPodsAndMirrorPods()	
    // 找到没有被terminate的pod
	activePods := kl.filterOutTerminatedPods(allPods)

	desiredPods := make(map[types.UID]sets.Empty)
	for _, pod := range activePods {
		desiredPods[pod.UID] = sets.Empty{}
	}
	kl.podWorkers.ForgetNonExistingPodWorkers(desiredPods)
	kl.probeManager.CleanupPods(desiredPods)

	runningPods, err := kl.runtimeCache.GetPods()
	...
	for _, pod := range runningPods {
		if _, found := desiredPods[pod.ID]; !found {
            // 第一步把Pod从running状态kill掉
			kl.podKillingCh <- &kubecontainer.PodPair{APIPod: nil, RunningPod: pod}
		}
	}
	// 然后开始移除这些Pod
	kl.removeOrphanedPodStatuses(allPods, mirrorPods)
	runningPods, err = kl.containerRuntime.GetPods(false)
	...

    // 然后移除volume
	err = kl.cleanupOrphanedPodDirs(allPods, runningPods)
	...

	// 移除所有孤立的mirror pod
	kl.podManager.DeleteOrphanedMirrorPods()

	// 移除Pod中不再运行的任何cgroup
	if kl.cgroupsPerQOS {
		kl.cleanupOrphanedPodCgroups(cgroupPods, activePods)
	}

	kl.backOff.GC()
	return nil
}
```





### 创建Pod: HandlePodAdditions

HandlePodAdditions是SyncHandler接口的一个方法，步骤如下：

1. 把Pod切片按创建时间排序，创建时间越早，越早被kubelet处理
2. 从`podManager`中获取现在节点上已经有的所有Pod
3. 然后把我们需要创建的Pod添加到podManager上
4. 查看是否允许创建该Pod
5. dispatchWork在一个pod worker中启动异步同步pod
6. 把Pod添加到probeManager 中

我刚开始看代码的时候被`podManager`和`probeManager` 搞混淆了，傻傻分不清。 在这里特地high light下：

`podManager` ：是负责管理在该节点上的Pod， 任何在这台node节点上创建/更新的pod都会在`podManager`上

`probeManager`:  是负责管理Pod的健康检查，例如我们Pod定义的liveness 是监控自身某个页面，那么这个`probeManager` 就是不断监控这个Pod的健康状态

```go
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
	start := kl.clock.Now()
    // 首先我们把列表（切片才对）里面的Pod按照创建时间排序，也就是说我们会按Pod创建时间来让kubelet创建Pod
	sort.Sort(sliceutils.PodsByCreationTime(pods))
	for _, pod := range pods {
        // 从podManager中获取现在节点上已经有的所有Pod
		existingPods := kl.podManager.GetPods()
		// 然后把我们需要创建的Pod添加到podManager上
		kl.podManager.AddPod(pod)		
		...
		if !kl.podIsTerminated(pod) {
			// 把已经存在的Pod列表过滤出状态不是terminate的Pod列表
			activePods := kl.filterOutTerminatedPods(existingPods)
			
            // 查看我们是否可以允许这个Pod，如果不是就拒绝
            // 这个是根据pod admission 来决定的，如果一个Pod是不允许被创建的，那么就拒绝该Pod
			if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
				kl.rejectPod(pod, reason, message)
				continue
			}
		}
		mirrorPod, _ := kl.podManager.GetMirrorPodByPod(pod)
        // dispatchWork在一个pod worker中启动异步同步pod
		kl.dispatchWork(pod, kubetypes.SyncPodCreate, mirrorPod, start)
        // 把Pod添加到probeManager 中
		kl.probeManager.AddPod(pod)
	}
}
```

`dispatchWork` 把需要创建/更新的Pod调度给`podWorkers` 去处理。 也就是说，当新建，或者更新Pod的时候，都会通过 `dispatchWork` 方法交给`podWorkers` 去处理， 而用户通过`kubectl delete pod`删除`Pod` 是走`deletePod`--`HandlePodRemoves`-- `deletePod`的路线。

```go
func (kl *Kubelet) dispatchWork(pod *v1.Pod, syncType kubetypes.SyncPodType, mirrorPod *v1.Pod, start time.Time) {
 	...
    // 使用podWorkers 去更新pod， 由于我们在 HandlePodAdditions 里面已经告诉dispatchWork syncType就是SyncPodCreate 也就是2
	kl.podWorkers.UpdatePod(&UpdatePodOptions{
		Pod:        pod,
		MirrorPod:  mirrorPod,
		UpdateType: syncType,
		OnCompleteFunc: func(err error) {
			...
		},
	})
	...
}
```





```go
func (p *podWorkers) UpdatePod(options *UpdatePodOptions) {
	pod := options.Pod
	uid := pod.UID
	var podUpdates chan UpdatePodOptions
	var exists bool
	...
	if podUpdates, exists = p.podUpdates[uid]; !exists {	
		podUpdates = make(chan UpdatePodOptions, 1)
		p.podUpdates[uid] = podUpdates

		// Creating a new pod worker either means this is a new pod, or that the
		// kubelet just restarted. In either case the kubelet is willing to believe
		// the status of the pod for the first pod worker sync. See corresponding
		// comment in syncPod.
		go func() {
			defer runtime.HandleCrash()
			p.managePodLoop(podUpdates)
		}()
	}
	if !p.isWorking[pod.UID] {
		p.isWorking[pod.UID] = true
		podUpdates <- *options
	} else {
		// if a request to kill a pod is pending, we do not let anything overwrite that request.
		update, found := p.lastUndeliveredWorkUpdate[pod.UID]
		if !found || update.UpdateType != kubetypes.SyncPodKill {
			p.lastUndeliveredWorkUpdate[pod.UID] = *options
		}
	}
}
```



# 问题回答

从Pod启动讲述下Kubelet具体的执行流程？

为什么一个Pod里面的几个container的网络、存储是可以互相通？

答： Pod的几个Container网络存储可以互通，具体在开放接口--CRI中提到，因为是使用了container模式

从Pod删除讲述下Kubelet具体的执行流程？

Kubelet是怎么上报Node自身的资源等数据？

Kubelet是如何执行垃圾回收？