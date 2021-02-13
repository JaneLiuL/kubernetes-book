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

从Pod启动讲述下Kubelet具体的执行流程？为什么一个Pod里面的几个container的网络、存储是可以互相通？

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





# 端口监听

kubelet 默认监听四个端口，分别为 10250 、10255、10248、4194。

- 10250（kubelet API）：kubelet server 与 apiserver 通信的端口，定期请求 apiserver 获取自己所应当处理的任务，通过该端口可以访问获取 node 资源以及状态。

- 10248（健康检查端口）：通过访问该端口可以判断 kubelet 是否正常工作, 通过 kubelet 的启动参数 `--healthz-port` 和 `--healthz-bind-address` 来指定监听的地址和端口。

  ```
  $ curl http://127.0.0.1:10248/healthzok
  ```

- 4194（cAdvisor 监听）：kublet 通过该端口可以获取到该节点的环境信息以及 node 上运行的容器状态等内容，访问 [http://localhost:4194](http://localhost:4194/) 可以看到 cAdvisor 的管理界面,通过 kubelet 的启动参数 `--cadvisor-port` 可以指定启动的端口。

```
  $ curl  http://127.0.0.1:4194/metrics
```

- 10255 （readonly API）：提供了 pod 和 node 的信息，接口以只读形式暴露出去，访问该端口不需要认证和鉴权。

  ```
  
  ```



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

从下方的代码可以看出，Kubelet的设计是真的非常精湛，他并非使用了A call B, B call C, C call D这种方法去让整个循环保持状态，而是解耦了各大模块，在kubelet 的Run的时候，就让各个模块使用go协程各自做各自的事情，然后写到channel里面，在syncLoopIteration里面，去读取channel里面的事件处理。

```go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
	case u, open := <-configCh:
		...
	case e := <-plegCh:
		...
	case <-syncCh:
		...
	case <-housekeepingCh:
		...
	}
	return true
}
```



