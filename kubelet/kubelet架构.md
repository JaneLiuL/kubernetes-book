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





# 问题回答

从Pod启动讲述下Kubelet具体的执行流程？

为什么一个Pod里面的几个container的网络、存储是可以互相通？

答： Pod的几个Container网络存储可以互通，具体在开放接口--CRI中提到，因为是使用了container模式

从Pod删除讲述下Kubelet具体的执行流程？

Kubelet是怎么上报Node自身的资源等数据？

Kubelet是如何执行垃圾回收？