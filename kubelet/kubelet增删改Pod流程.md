# Overview

此篇文档主要是讲解了Kubelet 创建Pod的流程。 



# 调用关系

```
Run()
	syncLoop()
		syncLoopIteration()
	
```



# 增删改pod

## 1. SyncLoop

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

## 2. syncLoopIteration

从下方的代码可以看出，Kubelet的设计是真的非常精湛，他并非使用了A call B, B call C, C call D这种方法去让整个循环保持状态，而是解耦了各大模块，在kubelet 的Run的时候，就让各个模块使用go协程各自做各自的事情，然后写到channel里面，在syncLoopIteration里面，去读取channel里面的事件处理。

```go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
     // configCh 这个channel是监听来自例如apiserver的变化，当从apiserver中的pod有被增删改，那么这个channel就会有具体Pod的更新操作
	case u, open := <-configCh:
		...
        switch u.Op {
            // 如果是创建Pod的操作，那么调用HandlePodAdditions去执行Pod的创建， 见下方的创建Pod
		case kubetypes.ADD:
			klog.V(2).Infof("SyncLoop (ADD, %q): %q", u.Source, format.Pods(u.Pods))		
			handler.HandlePodAdditions(u.Pods)
            
		case kubetypes.UPDATE:			
			handler.HandlePodUpdates(u.Pods)
            // 删除Pod的操作，看下方的删除Pod
		case kubetypes.REMOVE:
			klog.V(2).Infof("SyncLoop (REMOVE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodRemoves(u.Pods)         
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



# 创建Pod

## 1 总流程

HandlePodAdditions是SyncHandler接口的一个方法，步骤如下：

1. 把Pod切片按创建时间排序，创建时间越早，越早被kubelet处理
2. 从`podManager`中获取现在节点上已经有的所有Pod
3. 然后把我们需要创建的Pod添加到podManager上
4. 查看是否允许创建该Pod
5. dispatchWork在一个pod worker中启动异步同步pod，也就是我们下方的调度Pod
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

## 2  调度Pod

`dispatchWork` 把需要创建/更新的Pod调度给`podWorkers` 去处理。 也就是说，当新建，或者更新Pod的时候，都会通过 `dispatchWork` 方法交给`podWorkers`--`UpdatePod` 去处理，

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

`UpdatePod` 是将新的更新应用到特定的Pod上， 只有不存在的Pod, 也就是被Kubelet新增的Pod就会创建一个新协程去处理。

```go
// 代码位置 pkg/kubelet/pod_workers.go
func (p *podWorkers) UpdatePod(options *UpdatePodOptions) {
	pod := options.Pod
	uid := pod.UID
	var podUpdates chan UpdatePodOptions
	var exists bool
	...
	if podUpdates, exists = p.podUpdates[uid]; !exists {	
		podUpdates = make(chan UpdatePodOptions, 1)
		p.podUpdates[uid] = podUpdates

		// 如果Pod不存在，也就是说这个是一个新建的Pod, 就开启一个新协程去执行managePodLoop
		go func() {
			defer runtime.HandleCrash()
			p.managePodLoop(podUpdates)
		}()
	}
	if !p.isWorking[pod.UID] {
		p.isWorking[pod.UID] = true
		podUpdates <- *options
	} else {		
		update, found := p.lastUndeliveredWorkUpdate[pod.UID]
		if !found || update.UpdateType != kubetypes.SyncPodKill {
			p.lastUndeliveredWorkUpdate[pod.UID] = *options
		}
	}
}
```



同步Pod `managePodLoop` 实际上是调用了`syncPodFn` 去执行， 这里就不放出`managePodLoop` 的代码，而`syncPodFn`在`newPodWorkers` 实例化的时候，已经指出`syncPodFn` 就是`klet.syncPod`

```go
func NewMainKubelet(...) (*Kubelet, error) {
...
	klet.podWorkers = newPodWorkers(klet.syncPod, kubeDeps.Recorder, klet.workQueue, klet.resyncInterval, backOffPeriod, klet.podCache)
}
```

## 3 准备创建Pod

`syncPod` 的工作流程是:

1. 如果正在创建pod，记录pod worker启动延迟
2. 调用generateAPIPodStatus来准备v1pod status为pod
3. 如果pod被视为第一次运行，记录pod启动的latency
4. 在状态管理器中更新pod的状态
5. 如果pod是static pod，则创建static pod
6. 如果pod不存在，请为pod创建数据目录
7. 等待卷挂载
8. 获取pod的pullSecret
9. 调用CRI的SyncPod回调函数

```go
// 代码位置 pkg/kubelet/kubelet.go
func (kl *Kubelet) syncPod(o syncPodOptions) error {
	pod := o.pod
	mirrorPod := o.mirrorPod
	podStatus := o.podStatus
	updateType := o.updateType
	
    // 删除Pod的操作，这里我们跳过
	if updateType == kubetypes.SyncPodKill {
		...
	}

	var firstSeenTime time.Time
	if firstSeenTimeStr, ok := pod.Annotations[kubetypes.ConfigFirstSeenAnnotationKey]; ok {
		firstSeenTime = kubetypes.ConvertToTimestamp(firstSeenTimeStr).Get()
	}

	if updateType == kubetypes.SyncPodCreate {
		if !firstSeenTime.IsZero() {
			// 记录latency
			metrics.PodWorkerStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
			metrics.DeprecatedPodWorkerStartLatency.Observe(metrics.SinceInMicroseconds(firstSeenTime))
		} ...
	}

	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus)
	// 设置Pod IP 
	podStatus.IPs = make([]string, 0, len(apiPodStatus.PodIPs))
	for _, ipInfo := range apiPodStatus.PodIPs {
		podStatus.IPs = append(podStatus.IPs, ipInfo.IP)
	}

	if len(podStatus.IPs) == 0 && len(apiPodStatus.PodIP) > 0 {
		podStatus.IPs = []string{apiPodStatus.PodIP}
	}


    // 查询是否可以在节点上运行该Pod
	runnable := kl.canRunPod(pod)
	if !runnable.Admit {
		...
	}

	// 在状态管理器中更新pod的状态
	kl.statusManager.SetPodStatus(pod, apiPodStatus)
	...	
    // 设置pod的资源使用参数
	pcm := kl.containerManager.NewPodContainerManager()
	if !kl.podIsTerminated(pod) {
		firstSync := true
		for _, containerStatus := range apiPodStatus.ContainerStatuses {
			if containerStatus.State.Running != nil {
				firstSync = false
				break
			}
		}
		podKilled := false
		if !pcm.Exists(pod) && !firstSync {
			if err := kl.killPod(pod, nil, podStatus, nil); err == nil {
				podKilled = true
			}
		}
		// 创建Pod的Cgroups
		if !(podKilled && pod.Spec.RestartPolicy == v1.RestartPolicyNever) {
			if !pcm.Exists(pod) {
				if err := kl.containerManager.UpdateQOSCgroups(); err != nil {
					...
				}
				...
			}
		}
	}
	
    // 为静态Pod创建mirror pod
	if kubetypes.IsStaticPod(pod) {
		podFullName := kubecontainer.GetPodFullName(pod)
		deleted := false
		...
	}	
    // 设置Pod的数据目录
	if err := kl.makePodDataDirs(pod); err != nil {
		...
	}
	if !kl.podIsTerminated(pod) {		
        // attach和mount volumes
		if err := kl.volumeManager.WaitForAttachAndMount(pod); err != nil {
			...
		}
	}	
    // 获取Pod的pullSecret
	pullSecrets := kl.getPullSecretsForPod(pod)

	// 容器运行时SyncPod去创建容器
	result := kl.containerRuntime.SyncPod(pod, podStatus, pullSecrets, kl.backOff)
	kl.reasonCache.Update(pod.UID, result)
	if err := result.Error(); err != nil {
		for _, r := range result.SyncResults {
			if r.Error != kubecontainer.ErrCrashLoopBackOff && r.Error != images.ErrImagePullBackOff {
				return err
			}
		}
		return nil
	}
	return nil
}
```

## 4 容器运行时SyncPod 创建容器

SyncPod通过执行以下步骤将正在运行的pod同步到需要的pod中:

1. 计算sandbox和容器更改。
2. 如果有必要，杀死pod sandbox。
3. 杀死任何不应该运行的容器。
4. 如有必要，创建sandbox。
5. 创建临时容器。
6. 创建初始化容器。
7. 建立正常的容器。

```go
// 代码位置 pkg/kubelet/kuberuntime/kuberuntime_manager.go
func (m *kubeGenericRuntimeManager) SyncPod(pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {
	// 计算sandbox和容器更改
    // computePodActions通过对比pod跟podstatus来返回podActions这个对象来得到需要的操作，是否需要killpod还是createPodSandbox
	podContainerChanges := m.computePodActions(pod, podStatus)
	if podContainerChanges.CreateSandbox {
		ref, err := ref.GetReference(legacyscheme.Scheme, pod)
		...		
	}

    // 如果返回的podActions是需要killPod， 那么会先停止sandbox 然后创建一个新的sandbox
	if podContainerChanges.KillPod {
		if podContainerChanges.CreateSandbox {...
		}

		killResult := m.killPodWithSyncResult(pod, kubecontainer.ConvertPodStatusToRunningPod(m.runtimeName, podStatus), nil)
		result.AddPodSyncResult(killResult)
		...

		if podContainerChanges.CreateSandbox {
			m.purgeInitContainers(pod, podStatus)
		}
	} else {		
        // 杀死这个pod里面所有运行的容器
		for containerID, containerInfo := range podContainerChanges.ContainersToKill {		
			killContainerResult := kubecontainer.NewSyncResult(kubecontainer.KillContainer, containerInfo.name)
			result.AddSyncResult(killContainerResult)
			if err := m.killContainer(pod, containerID, containerInfo.name, containerInfo.message, nil); err != nil {
				killContainerResult.Fail(kubecontainer.ErrKillContainer, err.Error())
				...
				return
			}
		}
	}

	m.pruneInitContainersBeforeStart(pod, podStatus)
	
	var podIPs []string
	if podStatus != nil {
		podIPs = podStatus.IPs
	}
	
    // 如果podActions是需要创建sandbox, 那么就创建sandbox
	podSandboxID := podContainerChanges.SandboxID
	if podContainerChanges.CreateSandbox {
		...		
		createSandboxResult := kubecontainer.NewSyncResult(kubecontainer.CreatePodSandbox, format.Pod(pod))
		result.AddSyncResult(createSandboxResult)
        // 感兴趣的可以看看下方的附: createPodSandbox到底执行了什么操作
		podSandboxID, msg, err = m.createPodSandbox(pod, podContainerChanges.Attempt)
		if err != nil {
			...
		}		
		// 获取PodSandboxStatus
		podSandboxStatus, err := m.runtimeService.PodSandboxStatus(podSandboxID)
		if err != nil {
			...
		}		
     ...	
    // 获取 podSandboxConfig 
	configPodSandboxResult := kubecontainer.NewSyncResult(kubecontainer.ConfigPodSandbox, podSandboxID)
	result.AddSyncResult(configPodSandboxResult)
	podSandboxConfig, err := m.generatePodSandboxConfig(pod, podContainerChanges.Attempt)
	if err != nil {
		...
	}

	// typeName是一个用来描述日志消息中这种类型容器的标签，目前是:"container"， "init container"或"ephemeral container"
	start := func(typeName string, container *v1.Container) error {
		startContainerResult := kubecontainer.NewSyncResult(kubecontainer.StartContainer, container.Name)
		result.AddSyncResult(startContainerResult)

		isInBackOff, msg, err := m.doBackOff(pod, container, podStatus, backOff)
		if isInBackOff {
			startContainerResult.Fail(err, msg)			
			return err
		}				
        // 启动容器，具体可见附: 启动容器
		if msg, err := m.startContainer(podSandboxID, podSandboxConfig, container, pod, podStatus, pullSecrets, podIP, podIPs); err != nil {
			startContainerResult.Fail(err, msg)
			...
			switch {
			case err == images.ErrImagePullBackOff:
				klog.V(3).Infof("%v start failed: %v: %s", typeName, err, msg)
			default:
				utilruntime.HandleError(fmt.Errorf("%v start failed: %v: %s", typeName, err, msg))
			}
			return err
		}

		return nil
	}

    // 创建临时容器
	if utilfeature.DefaultFeatureGate.Enabled(features.EphemeralContainers) {
		for _, idx := range podContainerChanges.EphemeralContainersToStart {
			c := (*v1.Container)(&pod.Spec.EphemeralContainers[idx].EphemeralContainerCommon)
			start("ephemeral container", c)
		}
	}
	
    // 启动init 容器
	if container := podContainerChanges.NextInitContainerToStart; container != nil {		
		if err := start("init container", container); err != nil {
			return
		}


    // 启动容器里面需要启动的容器
	for _, idx := range podContainerChanges.ContainersToStart {
		start("container", &pod.Spec.Containers[idx])
	}

	return
}
```

### 附: createPodSandbox

步骤如下：

1. 在节点上创建日志目录，通常来说  PodSandbox的日志目录是`/var/log/pods/<podUID>/` 容器的日志目录是 `containerName/Instance#.log`  
2. 查找启动的容器运行时，如果没有找到则报错退出
3. 找到容器运行时，例如dockershim之后，会通过CRI的接口去gRPC去call 对应的容器运行时的`RunPodSandbox`。 这部分建议大家去看看 [容器运行时CRI](../开放接口/CRI.md)

```go
func (m *kubeGenericRuntimeManager) createPodSandbox(pod *v1.Pod, attempt uint32) (string, string, error) {
	podSandboxConfig, err := m.generatePodSandboxConfig(pod, attempt)
	if err != nil {
		...
	}
	
    // 首先在节点上创建日志目录
	err = m.osInterface.MkdirAll(podSandboxConfig.LogDirectory, 0755)
	if err != nil {
		...
	}

	runtimeHandler := ""
	if utilfeature.DefaultFeatureGate.Enabled(features.RuntimeClass) && m.runtimeClassManager != nil {
		runtimeHandler, err = m.runtimeClassManager.LookupRuntimeHandler(pod.Spec.RuntimeClassName)
		if err != nil {
			...
		}
		if runtimeHandler != "" {
			...
		}
	}

    // 通过容器运行时来启动PodSandbox，不同容器运行时例如docker跟rkt都是需要实现CRI接口即可
	podSandBoxID, err := m.runtimeService.RunPodSandbox(podSandboxConfig, runtimeHandler)
	if err != nil {
		...
	}

	return podSandBoxID, "", nil
}
```



### 附：启动容器

步骤如下：

1. 拉取镜像
2. 调用容器运行时去创建容器
3. 记录容器启动次数
4. 执行PostStart hook

```go
func (m *kubeGenericRuntimeManager) startContainer(podSandboxID string, podSandboxConfig *runtimeapi.PodSandboxConfig, container *v1.Container, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, podIP string, podIPs []string) (string, error) {
	// 拉取镜像
	imageRef, msg, err := m.imagePuller.EnsureImageExists(pod, container, pullSecrets, podSandboxConfig)
	if err != nil {
		...
	}

	// 创建容器
	ref, err := kubecontainer.GenerateContainerRef(pod, container)
	if err != nil {
		...
	}
	
    // restartCount 是记录容器被启动的次数，第一次创建的容器 restartCount 是0
	restartCount := 0
	containerStatus := podStatus.FindContainerStatusByName(container.Name)
	if containerStatus != nil {
		restartCount = containerStatus.RestartCount + 1
	}

	containerConfig, cleanupAction, err := m.generateContainerConfig(container, pod, restartCount, podIP, imageRef, podIPs)
	if cleanupAction != nil {
		defer cleanupAction()
	}
	if err != nil {
		...
	} 
    // 调用容器运行时创建容器CreateContainer
	containerID, err := m.runtimeService.CreateContainer(podSandboxID, containerConfig, podSandboxConfig)
	if err != nil {
		...
	}
    // PreStartContainer是属于InternalContainerLifecycle接口下的方法之一，主要作用是在容器启动之前进行类似设备挂载，设置endpoint
	err = m.internalLifecycle.PreStartContainer(pod, container, containerID)
	if err != nil {
		....
	}

	if ref != nil {
		m.containerRefManager.SetRef(kubecontainer.ContainerID{
			Type: m.runtimeName,
			ID:   containerID,
		}, ref)
	}

	// 启动容器，同样也是调用CRI的StartContainer
	err = m.runtimeService.StartContainer(containerID)
	if err != nil {
		...
	}
	

	// 将容器日志链接到用于集群日志记录的遗留容器日志位置
	containerMeta := containerConfig.GetMetadata()
	sandboxMeta := podSandboxConfig.GetMetadata()
	legacySymlink := legacyLogSymlink(containerID, containerMeta.Name, sandboxMeta.Name,
		sandboxMeta.Namespace)
	containerLog := filepath.Join(podSandboxConfig.LogDirectory, containerConfig.LogPath)

	if _, err := m.osInterface.Stat(containerLog); !os.IsNotExist(err) {
		if err := m.osInterface.Symlink(containerLog, legacySymlink); err != nil {
			...
		}
	}
	
    // 执行PostStart hook
	if container.Lifecycle != nil && container.Lifecycle.PostStart != nil {
		kubeContainerID := kubecontainer.ContainerID{
			Type: m.runtimeName,
			ID:   containerID,
		}
		msg, handlerErr := m.runner.Run(kubeContainerID, pod, container, container.Lifecycle.PostStart)
		if handlerErr != nil {
			...
		}
	}

	return "", nil
}

```



# 删除Pod

 而用户通过`kubectl delete pod`删除`Pod` 是走`deletePod`--`HandlePodRemoves`-- `deletePod`--`removeWorker`--`podUpdates`的路线。

```go
// 代码位置 pkg/kubelet/kubelet.go
func (kl *Kubelet) HandlePodRemoves(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		kl.podManager.DeletePod(pod)
		if kubetypes.IsMirrorPod(pod) {
			kl.handleMirrorPod(pod, start)
			continue
		}
		// Deletion is allowed to fail because the periodic cleanup routine
		// will trigger deletion again.
		if err := kl.deletePod(pod); err != nil {
			klog.V(2).Infof("Failed to delete pod %q, err: %v", format.Pod(pod), err)
		}
		kl.probeManager.RemovePod(pod)
	}
}
```

