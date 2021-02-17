# Overview

pleg模块的全称是pod lifecycle event generator，也就是pod生命周期的事件产生



# 实例化GenericPLEG

pleg在kubelet的`NewMainKubelet`时候实例化的

```go
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
	...) (*Kubelet, error) {
    ...
	NewGenericPLEG(klet.containerRuntime, plegChannelCapacity, plegRelistPeriod, klet.podCache, clock.RealClock{})
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
2. 更新运行中的pod和容器数量
3. 对比旧的和目前pod，然后产生events
4. 获取所有容器中旧的和新的pod， 然后交给computeEvents去得出需要做出的变化
5. 如果开启了podCache，那么我们需要更新podCache
6. 把Events发送到channel

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

```

```



# 总结

PLEG是通过定时每秒钟调取CRI获取所有Pod，然后对比容器状态来写入事件到channel里面，例如ContainerRemoved或者是启动容器。 然后kubelet在`syncLoop` 会调用`pleg.Watch`































