# Overview

pleg模块的全称是pod lifecycle event generator，是kubelet一个非常重要的模块，是通过调用CRI接口，获取pod和容器的状态，通过比对上一次的状态来产生事件，告诉kubelet是应该启动还是删除容器等。



# 实例化GenericPLEG

pleg在kubelet的`NewMainKubelet`时候实例化的，主要是做了两个事情：

1. 实例化pleg
2. 调用pleg的Healthy进行健康检查

```go
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
	...) (*Kubelet, error) {
    ...
    // 实例化pleg
	NewGenericPLEG(klet.containerRuntime, plegChannelCapacity, plegRelistPeriod, klet.podCache, clock.RealClock{})
    // pleg的Healthy健康检查
    klet.runtimeState.addHealthCheck("PLEG", klet.pleg.Healthy)
	...	    
	return klet, nil
}
```

# 运行Start

我们启动kubelet 的主方法可以看到，他调用了`PodLifecycleEventGenerator` 接口的`Start` 方法

```go
// 代码位置 pkg/kubelet/kubelet.go
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	...	
    // pleg.Start() 会调用relist来生产事件
	kl.pleg.Start()
    // syncLoop是消费者，会消费relist的事件
	kl.syncLoop(updates, kl)
}
```

接下来我们看看，pleg模块的`Start`方法 开启一个新的go协程，定时每秒钟运行`g.relist ` （实例化GenericPLEG的时候已经可以看到plegRelistPeriod = time.Second * 1）

```go
// 代码位置 pkg/kubelet/pleg/generic.go
func (g *GenericPLEG) Start() {
	go wait.Until(g.relist, g.relistPeriod, wait.NeverStop)
}
```

# 生产者relist

relist作为event的生产者，具体工作流程如下：

1. 调用CRI接口GetPods获取所有pod
2. 更新这次relist的时间为当前时间，这个非常重要，kubelet在检查pleg模块是否健康的时候会通过读取relist的时间，一旦无法获取到该时间，或者超过30分钟，那么就认为pleg模块已经不是active状态了
3. 更新运行中的pod和容器数量
4. 对比旧的和目前pod，然后产生events
5. 获取所有容器中旧的和新的pod， 然后交给computeEvents去得出需要做出的变化
6. 如果开启了podCache，那么我们需要更新podCache
7. 把Events发送到channel

```go
func (g *GenericPLEG) relist() {	
	timestamp := g.clock.Now()
	...	
    // 调用CRI接口GetPods获取所有pod
	podList, err := g.runtime.GetPods(true)
	...
	// 更新这次relist的时间为当前时间
	g.updateRelistTime(timestamp)

	pods := kubecontainer.Pods(podList)
    // 更新运行中的pod和容器数量	
	updateRunningPodAndContainerMetrics(pods)
    // podRecords是一个map数据结构，记录着旧的pod以及目前的pod信息
	g.podRecords.setCurrent(pods)
	
    // 对比旧的和目前pod，然后产生events
	eventsByPodID := map[types.UID][]*PodLifecycleEvent{}
	for pid := range g.podRecords {
		oldPod := g.podRecords.getOld(pid)
		pod := g.podRecords.getCurrent(pid)
		
        // 获取所有容器中旧的和新的pod， 然后交给computeEvents去得出需要做出的变化， 然后我们先看computeEvents 等会再回到主路线
		allContainers := getContainersFromPods(oldPod, pod)
		for _, container := range allContainers {
			events := computeEvents(oldPod, pod, &container.ID)
			for _, e := range events {
				updateEvents(eventsByPodID, e)
			}
		}
	}
	var needsReinspection map[types.UID]*kubecontainer.Pod
	if g.cacheEnabled() {
		needsReinspection = make(map[types.UID]*kubecontainer.Pod)
	}
	
	for pid, events := range eventsByPodID {
		pod := g.podRecords.getCurrent(pid)
        // 如果开启了podCache，那么我们需要更新podCache
		if g.cacheEnabled() {
			if err := g.updateCache(pod, pid); err != nil {
				...
				needsReinspection[pid] = pod
				continue
			} else {
				delete(g.podsToReinspect, pid)
			}
		}		
        // 更新podRecords
		g.podRecords.update(pid)
		for i := range events {
		
			if events[i].Type == ContainerChanged {
				continue
			}
			select {
              // 把Events发送到channel
			case g.eventChannel <- events[i]:
			default:
				metrics.PLEGDiscardEvents.WithLabelValues().Inc()
				klog.Error("event channel is full, discard this relist() cycle event")
			}
		}
	}

	if g.cacheEnabled() {
		if len(g.podsToReinspect) > 0 {
			// 更新cache
			for pid, pod := range g.podsToReinspect {
				if err := g.updateCache(pod, pid); err != nil {
					...
					needsReinspection[pid] = pod
				}
			}
		}

		// 更新cache的timestamp
		g.cache.UpdateTime(timestamp)
	}
	g.podsToReinspect = needsReinspection
}
```



## 计算computeEvents

```go
func computeEvents(oldPod, newPod *kubecontainer.Pod, cid *kubecontainer.ContainerID) []*PodLifecycleEvent {
	var pid types.UID
	if oldPod != nil {
		pid = oldPod.ID
	} else if newPod != nil {
		pid = newPod.ID
	}
	oldState := getContainerState(oldPod, cid)
	newState := getContainerState(newPod, cid)
    // 获取old pod的state跟目前pod的state, 交给generateEvents去得出需要做出的变化事件
	return generateEvents(pid, cid.ID, oldState, newState)
}
```

通过对比old pod的state跟目前pod的state，如果一致，直接返回，否则会出现如下几种情况：

1. 当前podState 容器是"running"，那么返回事件，type是ContainerStarted
2. 当前podState 容器是"exited"，也就是说容器是死亡状态，返回报告事件，type是ContainerDied
3. 当前podState 容器状态是"unknown"， 那么返回报告事件, type是ContainerChanged
4. 当前podState 容器状态"non-existent", 不存在，也就说明之前已经报告过死亡了，返回报告事件，type是ContainerRemoved
5. 都不命中得情况下，就说明无法从CRI获取到容器状态了

```go
func generateEvents(podID types.UID, cid string, oldState, newState plegContainerState) []*PodLifecycleEvent {
    // 如果old pod的state跟目前pod的state一样，说明不需要变化，直接返回
	if newState == oldState {
		return nil
	}

	switch newState {
	case plegContainerRunning:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerStarted, Data: cid}}
	case plegContainerExited:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}}
	case plegContainerUnknown:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerChanged, Data: cid}}
	case plegContainerNonExistent:
		switch oldState {
		case plegContainerExited:			
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerRemoved, Data: cid}}
		default:
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}, {ID: podID, Type: ContainerRemoved, Data: cid}}
		}
	default:
		panic(fmt.Sprintf("unrecognized container state: a%v", newState))
	}
}
```

# 消费者syncLoop

`syncLoop` 是作为消费者去消费pleg模块产生的消息，主要工作流程如下:

1. 调用pleg的`Watch`方法，去获取`eventChannel`
2. 进入`syncLoopIteration` 循环，一旦可以从`eventChannel`消费消息，那么进入`HandlePodSyncs`去同步操作

```go
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	...
    // pleg的Watch方法是返回eventChannel，也就是上面relist生产的Events消息
	plegCh := kl.pleg.Watch()
    // 看下方的syncLoopIteration
    if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
    ...
}
```



```go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
	...
    // pleg的channel只要可以消费到消息，那么就进入该循环
	case e := <-plegCh:
		if isSyncPodWorthy(e) {
			// PLEG event for a pod; sync it.
            // 调用handler接口的HandlePodSyncs去同步，例如是要启动或者删除操作
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				klog.V(2).Infof("SyncLoop (PLEG): %q, event: %#v", format.Pod(pod), e)
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} ...
		}
		// 如果是容器死亡的消息，那么调用cleanUpContainersInPod来清理pod里面的容器
		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
	...
}
```



# Healthy健康检查

在上方的生产者relist中，我们看到每次执行relist的时候，都会调用`g.updateRelistTime(timestamp)`更新relist的时间，而kubelet在检查pleg模块是否健康的时候会通过读取relist的时间，一旦无法获取到该时间，或者超过30分钟，那么就认为pleg模块已经是不健康，也就是非active状态。

```go
func (g *GenericPLEG) Healthy() (bool, error) {
    // 获取到pleg模块调用relist的时间
	relistTime := g.getRelistTime()
    // 如果获取到的时间是空，也就说明从来没有进行过relist
	if relistTime.IsZero() {
		return false, fmt.Errorf("pleg has yet to be successful")
	}
	elapsed := g.clock.Since(relistTime)
    // 如果获取到relist的时间跟现在的时间差别超过relistThreshold，就认为pleg模块不健康
	if elapsed > relistThreshold {
		return false, fmt.Errorf("pleg was last seen active %v ago; threshold is %v", elapsed, relistThreshold)
	}
	return true, nil
}

```



# 总结

PLEG是通过定时每秒钟调取CRI获取所有Pod，然后对比容器状态来写入事件到channel里面，例如ContainerRemoved或者是启动容器。 然后kubelet在`syncLoop`  来消费消息，完成从actual state到desire state的转变。































