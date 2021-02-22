

# Overview

eviction模块 是Kubelet受到了磁盘或者内存压力的时候，为了包含重要的服务不受影响而驱逐其他应用的模块。

# 实例化



```go
// 代码位置 pkg/kubelet/kubelet.go
func NewMainKubelet(..) (*Kubelet, error) {
	...
	if experimentalNodeAllocatableIgnoreEvictionThreshold {
		// Do not provide kubeCfg.EnforceNodeAllocatable to eviction threshold parsing if we are not enforcing Evictions
		enforceNodeAllocatable = []string{}
	}
	thresholds, err := eviction.ParseThresholdConfig(enforceNodeAllocatable, kubeCfg.EvictionHard, kubeCfg.EvictionSoft, kubeCfg.EvictionSoftGracePeriod, kubeCfg.EvictionMinimumReclaim)
	
	evictionConfig := eviction.Config{
		PressureTransitionPeriod: kubeCfg.EvictionPressureTransitionPeriod.Duration,
		MaxPodGracePeriodSeconds: int64(kubeCfg.EvictionMaxPodGracePeriod),
		Thresholds:               thresholds,
		KernelMemcgNotification:  experimentalKernelMemcgNotification,
		PodCgroupRoot:            kubeDeps.ContainerManager.GetPodCgroupRoot(),
	}
	...
	// setup eviction manager
	evictionManager, evictionAdmitHandler := eviction.NewManager(klet.resourceAnalyzer, evictionConfig, killPodNow(klet.podWorkers, kubeDeps.Recorder), klet.podManager.GetMirrorPodByPod, klet.imageManager, klet.containerGC, kubeDeps.Recorder, nodeRef, klet.clock)

	klet.evictionManager = evictionManager
    klet.admitHandlers.AddPodAdmitHandler(evictionAdmitHandler)
	...	
}
```



## managerImple数据结构

```go
type managerImpl struct {
	//  用来跟踪时间
	clock clock.Clock	
    // manager的配置
	config Config	
    // 杀掉pod的函数
	killPodFunc KillPodFunc	
    // 通过一个静态pod获取mirror的函数
	mirrorPodFunc MirrorPodFunc
    // gc image的接口，通常是调用CRI那边的接口	
	imageGC ImageGC
    // gc container的接口	
	containerGC ContainerGC	
	sync.RWMutex
    // 一系列条件去保护node	
	nodeConditions []v1.NodeConditionType	
    // 捕捉根据满足的阈值最后观察到节点条件的时间
	nodeConditionsLastObservedAt nodeConditionsObservedAt
	nodeRef *v1.ObjectReference
	recorder record.EventRecorder
	// 用于测量系统的使用统计
	summaryProvider stats.SummaryProvider
	// 记录第一次观察到阈值的时间
	thresholdsFirstObservedAt thresholdsObservedAt
	// 记录已经满足但尚未解析的阈值集(包括宽限期)
	thresholdsMet []evictionapi.Threshold
	// signalToRankFunc将资源映射到该资源的排序函数。
	signalToRankFunc map[evictionapi.Signal]rankFunc
	// signalToNodeReclaimFuncs maps a resource to an ordered list of functions that know how to reclaim that resource.
	signalToNodeReclaimFuncs map[evictionapi.Signal]nodeReclaimFuncs	
	lastObservations signalObservations
	// dedicatedImageFs 指示imagefs是否在rootfs之外的单独设备上
	dedicatedImageFs *bool
	// thresholdNotifiers是一个内存阈值通知器列表，每个通知一个内存回收阈值
	thresholdNotifiers []ThresholdNotifier
	// thresholdsLastUpdated是thresholdNotifiers最后一次更新的时间。
	thresholdsLastUpdated time.Time
}
```



```go
func NewManager(
	summaryProvider stats.SummaryProvider,
	config Config,
	killPodFunc KillPodFunc,
	mirrorPodFunc MirrorPodFunc,
	imageGC ImageGC,
	containerGC ContainerGC,
	recorder record.EventRecorder,
	nodeRef *v1.ObjectReference,
	clock clock.Clock,
) (Manager, lifecycle.PodAdmitHandler) {
	manager := &managerImpl{
		clock:                        clock,
		killPodFunc:                  killPodFunc,
		mirrorPodFunc:                mirrorPodFunc,
		imageGC:                      imageGC,
		containerGC:                  containerGC,
		config:                       config,
		recorder:                     recorder,
		summaryProvider:              summaryProvider,
		nodeRef:                      nodeRef,
		nodeConditionsLastObservedAt: nodeConditionsObservedAt{},
		thresholdsFirstObservedAt:    thresholdsObservedAt{},
		dedicatedImageFs:             nil,
		thresholdNotifiers:           []ThresholdNotifier{},
	}
	return manager, manager
}
```



# 运行

调用链关系如下

```bash
# 代码位置 pkg/kubelet/kubelet.go
Run()
	initializeRuntimeDependentModules()
		kl.evictionManager.Start(kl.StatsProvider, kl.GetActivePods, kl.podResourcesAreReclaimed, evictionMonitoringPeriod)
```

`eviction`的启动工作流程如下：

1. 定义`thresholdHandler` 也就是执行`synchronize`的函数
2. 使用一个新的go协程，去死循环eviction manager的监控，一旦监控到有需要驱逐的pods，就调用`waitForPodsCleanup`去驱逐清理pod

```go
func (m *managerImpl) Start(diskInfoProvider DiskInfoProvider, podFunc ActivePodsFunc, podCleanedUpFunc PodCleanedUpFunc, monitoringInterval time.Duration) {
	thresholdHandler := func(message string) {
		klog.Infof(message)
		m.synchronize(diskInfoProvider, podFunc)
	}
	...	
    // 使用一个新的go协程，去死循环eviction manager的监控
	go func() {
		for {
			if evictedPods := m.synchronize(diskInfoProvider, podFunc); evictedPods != nil 					m.waitForPodsCleanup(podCleanedUpFunc, evictedPods)
			} else {
				time.Sleep(monitoringInterval)
			}
		}
	}()
}
```

## synchronize 监控

`synchronize` 是一个强制驱逐的阈值

```go
// 代码位置 pkg/kubelet/eviction_manager.go
func (m *managerImpl) synchronize(diskInfoProvider DiskInfoProvider, podFunc ActivePodsFunc) []*v1.Pod {
	
	klog.V(3).Infof("eviction manager: synchronize housekeeping")
	// build the ranking functions (if not yet known)
	// TODO: have a function in cadvisor that lets us know if global housekeeping has completed
	if m.dedicatedImageFs == nil {
		hasImageFs, ok := diskInfoProvider.HasDedicatedImageFs()
		if ok != nil {
			return nil
		}
		m.dedicatedImageFs = &hasImageFs
		m.signalToRankFunc = buildSignalToRankFunc(hasImageFs)
		m.signalToNodeReclaimFuncs = buildSignalToNodeReclaimFuncs(m.imageGC, m.containerGC, hasImageFs)
	}

	activePods := podFunc()
	updateStats := true
	summary, err := m.summaryProvider.Get(updateStats)
	
	if m.clock.Since(m.thresholdsLastUpdated) > notifierRefreshInterval {
		m.thresholdsLastUpdated = m.clock.Now()
		for _, notifier := range m.thresholdNotifiers {
			if err := notifier.UpdateThreshold(summary); err != nil {
				klog.Warningf("eviction manager: failed to update %s: %v", notifier.Description(), err)
			}
		}
	}

	// make observations and get a function to derive pod usage stats relative to those observations.
	observations, statsFunc := makeSignalObservations(summary)
	debugLogObservations("observations", observations)

	// determine the set of thresholds met independent of grace period
	thresholds = thresholdsMet(thresholds, observations, false)
	debugLogThresholdsWithObservation("thresholds - ignoring grace period", thresholds, observations)

	// determine the set of thresholds previously met that have not yet satisfied the associated min-reclaim
	if len(m.thresholdsMet) > 0 {
		thresholdsNotYetResolved := thresholdsMet(m.thresholdsMet, observations, true)
		thresholds = mergeThresholds(thresholds, thresholdsNotYetResolved)
	}
	debugLogThresholdsWithObservation("thresholds - reclaim not satisfied", thresholds, observations)

	// track when a threshold was first observed
	now := m.clock.Now()
	thresholdsFirstObservedAt := thresholdsFirstObservedAt(thresholds, m.thresholdsFirstObservedAt, now)

	// the set of node conditions that are triggered by currently observed thresholds
	nodeConditions := nodeConditions(thresholds)
	if len(nodeConditions) > 0 {
		klog.V(3).Infof("eviction manager: node conditions - observed: %v", nodeConditions)
	}

	// track when a node condition was last observed
	nodeConditionsLastObservedAt := nodeConditionsLastObservedAt(nodeConditions, m.nodeConditionsLastObservedAt, now)

	// node conditions report true if it has been observed within the transition period window
	nodeConditions = nodeConditionsObservedSince(nodeConditionsLastObservedAt, m.config.PressureTransitionPeriod, now)
	if len(nodeConditions) > 0 {
		klog.V(3).Infof("eviction manager: node conditions - transition period not met: %v", nodeConditions)
	}

	// determine the set of thresholds we need to drive eviction behavior (i.e. all grace periods are met)
	thresholds = thresholdsMetGracePeriod(thresholdsFirstObservedAt, now)
	debugLogThresholdsWithObservation("thresholds - grace periods satisfied", thresholds, observations)

	// update internal state
	m.Lock()
	m.nodeConditions = nodeConditions
	m.thresholdsFirstObservedAt = thresholdsFirstObservedAt
	m.nodeConditionsLastObservedAt = nodeConditionsLastObservedAt
	m.thresholdsMet = thresholds

	// determine the set of thresholds whose stats have been updated since the last sync
	thresholds = thresholdsUpdatedStats(thresholds, observations, m.lastObservations)
	debugLogThresholdsWithObservation("thresholds - updated stats", thresholds, observations)

	m.lastObservations = observations
	m.Unlock()

	// evict pods if there is a resource usage violation from local volume temporary storage
	// If eviction happens in localStorageEviction function, skip the rest of eviction action
	if utilfeature.DefaultFeatureGate.Enabled(features.LocalStorageCapacityIsolation) {
		if evictedPods := m.localStorageEviction(summary, activePods); len(evictedPods) > 0 {
			return evictedPods
		}
	}

	if len(thresholds) == 0 {
		klog.V(3).Infof("eviction manager: no resources are starved")
		return nil
	}

	// rank the thresholds by eviction priority
	sort.Sort(byEvictionPriority(thresholds))
	thresholdToReclaim, resourceToReclaim, foundAny := getReclaimableThreshold(thresholds)
	if !foundAny {
		return nil
	}
	klog.Warningf("eviction manager: attempting to reclaim %v", resourceToReclaim)

	// record an event about the resources we are now attempting to reclaim via eviction
	m.recorder.Eventf(m.nodeRef, v1.EventTypeWarning, "EvictionThresholdMet", "Attempting to reclaim %s", resourceToReclaim)

	// check if there are node-level resources we can reclaim to reduce pressure before evicting end-user pods.
	if m.reclaimNodeLevelResources(thresholdToReclaim.Signal, resourceToReclaim) {
		klog.Infof("eviction manager: able to reduce %v pressure without evicting pods.", resourceToReclaim)
		return nil
	}

	klog.Infof("eviction manager: must evict pod(s) to reclaim %v", resourceToReclaim)

	// rank the pods for eviction
	rank, ok := m.signalToRankFunc[thresholdToReclaim.Signal]
	if !ok {
		klog.Errorf("eviction manager: no ranking function for signal %s", thresholdToReclaim.Signal)
		return nil
	}

	// the only candidates viable for eviction are those pods that had anything running.
	if len(activePods) == 0 {
		klog.Errorf("eviction manager: eviction thresholds have been met, but no pods are active to evict")
		return nil
	}

	// rank the running pods for eviction for the specified resource
	rank(activePods, statsFunc)

	klog.Infof("eviction manager: pods ranked for eviction: %s", format.Pods(activePods))

	//record age of metrics for met thresholds that we are using for evictions.
	for _, t := range thresholds {
		timeObserved := observations[t.Signal].time
		if !timeObserved.IsZero() {
			metrics.EvictionStatsAge.WithLabelValues(string(t.Signal)).Observe(metrics.SinceInSeconds(timeObserved.Time))
			metrics.DeprecatedEvictionStatsAge.WithLabelValues(string(t.Signal)).Observe(metrics.SinceInMicroseconds(timeObserved.Time))
		}
	}

	// we kill at most a single pod during each eviction interval
	for i := range activePods {
		pod := activePods[i]
		gracePeriodOverride := int64(0)
		if !isHardEvictionThreshold(thresholdToReclaim) {
			gracePeriodOverride = m.config.MaxPodGracePeriodSeconds
		}
		message, annotations := evictionMessage(resourceToReclaim, pod, statsFunc)
		if m.evictPod(pod, gracePeriodOverride, message, annotations) {
			metrics.Evictions.WithLabelValues(string(thresholdToReclaim.Signal)).Inc()
			return []*v1.Pod{pod}
		}
	}
	klog.Infof("eviction manager: unable to evict any pods from the node")
	return nil
}
```



## 驱逐清理Pod资源

在`eviction`模块运行的时候（具体可以看运行的调用链），已经指出`PodCleanedUpFunc`就是`podResourcesAreReclaimed`

`waitForPodsCleanup`的工作流程如下：

1. 每分钟对传入的要驱逐的pod列表执行`podCleanedUpFunc`
2. 如果30分钟内都无法对所有要驱逐的pod执行完`podCleanedUpFunc`，则超时退出

```go
// 代码位置 pkg/kubelet/eviction_manager.go
func (m *managerImpl) waitForPodsCleanup(podCleanedUpFunc PodCleanedUpFunc, pods []*v1.Pod) {
    // 设置timeout的时间，默认是30分钟，超过timeout之后则退出
	timeout := m.clock.NewTimer(podCleanupTimeout)    
	defer timeout.Stop()
    // 设置定时器1分钟    
	ticker := m.clock.NewTicker(podCleanupPollFreq)
	defer ticker.Stop()
	for {
		select {
         // 当超时，则直接退出
		case <-timeout.C():
			klog.Warningf("eviction manager: timed out waiting for pods %s to be cleaned up", format.Pods(pods))
			return
         // 每分钟执行一次   
		case <-ticker.C():
            // 轮询所有pod执行podCleanedUpFunc，直到所有pod都被清理完成才返回
			for i, pod := range pods {
				if !podCleanedUpFunc(pod) {
					break
				}
				if i == len(pods)-1 {
					klog.Infof("eviction manager: pods %s successfully cleaned up", format.Pods(pods))
					return
				}
			}
		}
	}
}
```

`podResourcesAreReclaimed`是一个调用`    return kl.PodResourcesAreReclaimed(pod, status)` 尝试去回收一个pod的所有资源， `PodResourcesAreReclaimed` 主要是在删除一个pod之前去回收pod的资源。 如果一个pod正在消耗的所有所需的节点级资源都已被kubelet回收则返回true，否则返回false。 

`PodResourcesAreReclaimed` 的工作流程如下：

1. 查询pod的container状态是否是running，如果是running则返回false
2. 获取pod的ContainerStatuses，如果还能获取到container的状态，说明container没有被删除，那么返回false
3. 如果pod的卷仍然存在那么也是一样返回false
4. 尝试新建一个pod cgroup sanbox，如果存在，说明之前的pod cgroup sandbox没有被清理，一样返回false
5. 以上都不是，则返回true，也就是说没有running的container, 也获取不到任何container， volume被回收，cgroup sandbox也被清理完成，就是pod的资源都被回收完成，就返回true

```go
// 代码位置 pkg/kubelet/kubelet_pods.go
func (kl *Kubelet) podResourcesAreReclaimed(pod *v1.Pod) bool {
	status, ok := kl.statusManager.GetPodStatus(pod.UID)
	if !ok {
		status = pod.Status
	}
	return kl.PodResourcesAreReclaimed(pod, status)
}
// PodResourcesAreReclaimed 主要是在删除一个pod之前去回收pod的资源
// 如果一个pod正在消耗的所有所需的节点级资源都已被kubelet回收则返回true，否则返回false
func (kl *Kubelet) PodResourcesAreReclaimed(pod *v1.Pod, status v1.PodStatus) bool {
	if !notRunning(status.ContainerStatuses) {
		// 如果container还在运行的话，就不能删除pod
		klog.V(3).Infof("Pod %q is terminated, but some containers are still running", format.Pod(pod))
		return false
	}
	runtimeStatus, err := kl.podCache.Get(pod.UID)	
    // 获取pod的ContainerStatuses，如果还能获取到container的状态，说明container没有被删除，那么返回false
	if len(runtimeStatus.ContainerStatuses) > 0 {
		var statusStr string
		for _, status := range runtimeStatus.ContainerStatuses {
			statusStr += fmt.Sprintf("%+v ", *status)
		}
		klog.V(3).Infof("Pod %q is terminated, but some containers have not been cleaned up: %s", format.Pod(pod), statusStr)
		return false
	}
    // 如果pod的卷仍然存在那么也是一样返回false
	if kl.podVolumesExist(pod.UID) && !kl.keepTerminatedPodVolumes {
		klog.V(3).Infof("Pod %q is terminated, but some volumes have not been cleaned up", format.Pod(pod))
		return false
	}
    // 尝试新建一个pod cgroup sanbox，如果存在，说明之前的pod cgroup sandbox没有被清理，一样返回false
	if kl.kubeletConfiguration.CgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
		if pcm.Exists(pod) {
			klog.V(3).Infof("Pod %q is terminated, but pod cgroup sandbox has not been cleaned up", format.Pod(pod))
			return false
		}
	}
	return true
}
```



## Admit 

我一开始想了很久，为什么kubelet 的`eviction`模块有一个Admit的方法，一开始有100个pod被调度到节点a上，(调度是一次性的，调度完成之后这个pod会一直在这个节点运行，因此需要kubelet自身的监控，为了保护node节点本身)后来节点a出现内存压力或者磁盘压力的时候，为了保护node节点的稳定性需要驱逐低优先级的pod，而驱逐什么pod，会使用Admit去查询pod的优先级，pod是属于best effort还是guarantee，以及通过pod的污点容忍情况来决定

工作流程如下：

1. 查询`nodeConditions`是否为0，如果是，也就是说不管是收到内存还是磁盘或者其他压力，都对所有的pod进行admit放行
2. 如果是静态pod获取mirror pod或者pod Priority是大于`scheduling.SystemCriticalPriority` （也就是设置的priority的值大于scheduling.SystemCriticalPriority），则对pod进行admit放行
3. 如果是只有内存压力的情况下，best effort以外的所有pod都admit放行，或者有带上污点是key = `TaintNodeMemoryPressure`, value = `TaintEffectNoSchedule`都admit放行。其他则都不放行，也就是说，如果是best effort，而且没有打上容忍内存污点的就不放行

这里突然想起我之前遇到的kubernetes的一个逻辑的BUG：

当我创建了一个replicaset ，是属于guaranteed的，一开始集群的节点正常，我的replicaset 的pod被调度到节点a上，后节点出现磁盘压力，驱逐pod之后，在节点a就出现很多个状态是evicted的pod，然后replicaset get到filterpod的数量跟声明的数量不一致，就去创建pod，调度器就再调度该pod, 而由于驱逐清理pod的节点后又有了磁盘空间，又被调度到该节点，调度之后不久，又由于磁盘空间被驱逐，导致节点a上越来越多的evicted的pod

```go
// 代码位置 pkg/kubelet/eviction_manager.go
func (m *managerImpl) Admit(attrs *lifecycle.PodAdmitAttributes) lifecycle.PodAdmitResult {	
	if len(m.nodeConditions) == 0 {
		return lifecycle.PodAdmitResult{Admit: true}
	}
	if kubelettypes.IsCriticalPod(attrs.Pod) {
		return lifecycle.PodAdmitResult{Admit: true}
	}
	// 如果是只有内存压力的情况下
	nodeOnlyHasMemoryPressureCondition := hasNodeCondition(m.nodeConditions, v1.NodeMemoryPressure) && len(m.nodeConditions) == 1
	if nodeOnlyHasMemoryPressureCondition {
		notBestEffort := v1.PodQOSBestEffort != v1qos.GetPodQOS(attrs.Pod)
		if notBestEffort {
			return lifecycle.PodAdmitResult{Admit: true}
		}

		// 通过检查pod的污点确定是否放行
		if v1helper.TolerationsTolerateTaint(attrs.Pod.Spec.Tolerations, &v1.Taint{
			Key:    v1.TaintNodeMemoryPressure,
			Effect: v1.TaintEffectNoSchedule,
		}) {
			return lifecycle.PodAdmitResult{Admit: true}
		}
	}

	// 如果是best effort，而且没有打上容忍内存污点的就不放行
	return lifecycle.PodAdmitResult{
		Admit:   false,
		Reason:  Reason,
		Message: fmt.Sprintf(nodeConditionMessageFmt, m.nodeConditions),
	}
}

```



# 单元测试

今天看了`eviction` 模块的单元测试，下方是测试受到内存压力的单元测试案例：

1. 定义一组测试数据，5个pod 不同priority，不同资源限制和使用率
2. 创建这5个pod
3. 定义要驱逐的第五个pod, podToEvict
4. 实例化`managerImpl`，包括压力过度时间是5分钟等
5. 首先创建两个best effort的pod， 这个时候应该没有任何的内存压力
6. 然后更改时间，mock内存压力的情况下，而还没到5分钟，不应该有任何pod被kill
7. 然后过了GracePeriod之后，应该因为内存压力开始驱逐pod， 第一个被驱逐的pod应该是第五个pod
8. 减少内存压力，（但仍然有一定的内存压力），现在应该没有pod被kill， 也就是`podKiller.pod`应该等于nil
9. 现在best-effort的pod应该还是不应该被创建，而burstable和guaranteed的还是可以被创建
10. 移除所有内存压力，现在在应该没有任何内存压力，也应该没有pod被kill
11. 现在所有pod都可以被admit

```go
func TestMemoryPressure(t *testing.T) {
	podMaker := makePodWithMemoryStats
	summaryStatsMaker := makeMemoryStats
	podsToMake := []podToMake{
		{name: "guaranteed-low-priority-high-usage", priority: lowPriority, requests: newResourceList("100m", "1Gi", ""), limits: newResourceList("100m", "1Gi", ""), memoryWorkingSet: "900Mi"},
		{name: "burstable-below-requests", priority: defaultPriority, requests: newResourceList("100m", "100Mi", ""), limits: newResourceList("200m", "1Gi", ""), memoryWorkingSet: "50Mi"},
		{name: "burstable-above-requests", priority: defaultPriority, requests: newResourceList("100m", "100Mi", ""), limits: newResourceList("200m", "1Gi", ""), memoryWorkingSet: "400Mi"},
		{name: "best-effort-high-priority-high-usage", priority: highPriority, requests: newResourceList("", "", ""), limits: newResourceList("", "", ""), memoryWorkingSet: "400Mi"},
		{name: "best-effort-low-priority-low-usage", priority: lowPriority, requests: newResourceList("", "", ""), limits: newResourceList("", "", ""), memoryWorkingSet: "100Mi"},
	}
	pods := []*v1.Pod{}
	podStats := map[*v1.Pod]statsapi.PodStats{}
	for _, podToMake := range podsToMake {
		pod, podStat := podMaker(podToMake.name, podToMake.priority, podToMake.requests, podToMake.limits, podToMake.memoryWorkingSet)
		pods = append(pods, pod)
		podStats[pod] = podStat
	}
	podToEvict := pods[4]
	activePodsFunc := func() []*v1.Pod {
		return pods
	}

	fakeClock := clock.NewFakeClock(time.Now())
	podKiller := &mockPodKiller{}
	diskInfoProvider := &mockDiskInfoProvider{dedicatedImageFs: false}
	diskGC := &mockDiskGC{err: nil}
	nodeRef := &v1.ObjectReference{Kind: "Node", Name: "test", UID: types.UID("test"), Namespace: ""}

	config := Config{
		MaxPodGracePeriodSeconds: 5,
		PressureTransitionPeriod: time.Minute * 5,
		Thresholds: []evictionapi.Threshold{
			{
				Signal:   evictionapi.SignalMemoryAvailable,
				Operator: evictionapi.OpLessThan,
				Value: evictionapi.ThresholdValue{
					Quantity: quantityMustParse("1Gi"),
				},
			},
			{
				Signal:   evictionapi.SignalMemoryAvailable,
				Operator: evictionapi.OpLessThan,
				Value: evictionapi.ThresholdValue{
					Quantity: quantityMustParse("2Gi"),
				},
				GracePeriod: time.Minute * 2,
			},
		},
	}
	summaryProvider := &fakeSummaryProvider{result: summaryStatsMaker("2Gi", podStats)}
	manager := &managerImpl{
		clock:                        fakeClock,
		killPodFunc:                  podKiller.killPodNow,
		imageGC:                      diskGC,
		containerGC:                  diskGC,
		config:                       config,
		recorder:                     &record.FakeRecorder{},
		summaryProvider:              summaryProvider,
		nodeRef:                      nodeRef,
		nodeConditionsLastObservedAt: nodeConditionsObservedAt{},
		thresholdsFirstObservedAt:    thresholdsObservedAt{},
	}

	// create a best effort pod to test admission
	bestEffortPodToAdmit, _ := podMaker("best-admit", defaultPriority, newResourceList("", "", ""), newResourceList("", "", ""), "0Gi")
	burstablePodToAdmit, _ := podMaker("burst-admit", defaultPriority, newResourceList("100m", "100Mi", ""), newResourceList("200m", "200Mi", ""), "0Gi")

	// synchronize
	manager.synchronize(diskInfoProvider, activePodsFunc)

	// we should not have memory pressure
	if manager.IsUnderMemoryPressure() {
		t.Errorf("Manager should not report memory pressure")
	}

	// try to admit our pods (they should succeed)
	expected := []bool{true, true}
	for i, pod := range []*v1.Pod{bestEffortPodToAdmit, burstablePodToAdmit} {
		if result := manager.Admit(&lifecycle.PodAdmitAttributes{Pod: pod}); expected[i] != result.Admit {
			t.Errorf("Admit pod: %v, expected: %v, actual: %v", pod, expected[i], result.Admit)
		}
	}

	// induce soft threshold
	fakeClock.Step(1 * time.Minute)
	summaryProvider.result = summaryStatsMaker("1500Mi", podStats)
	manager.synchronize(diskInfoProvider, activePodsFunc)

	// we should have memory pressure
	if !manager.IsUnderMemoryPressure() {
		t.Errorf("Manager should report memory pressure since soft threshold was met")
	}

	// verify no pod was yet killed because there has not yet been enough time passed.
	if podKiller.pod != nil {
		t.Errorf("Manager should not have killed a pod yet, but killed: %v", podKiller.pod.Name)
	}

	// step forward in time pass the grace period
	fakeClock.Step(3 * time.Minute)
	summaryProvider.result = summaryStatsMaker("1500Mi", podStats)
	manager.synchronize(diskInfoProvider, activePodsFunc)

	// we should have memory pressure
	if !manager.IsUnderMemoryPressure() {
		t.Errorf("Manager should report memory pressure since soft threshold was met")
	}

	// verify the right pod was killed with the right grace period.
	if podKiller.pod != podToEvict {
		t.Errorf("Manager chose to kill pod: %v, but should have chosen %v", podKiller.pod.Name, podToEvict.Name)
	}
	if podKiller.gracePeriodOverride == nil {
		t.Errorf("Manager chose to kill pod but should have had a grace period override.")
	}
	observedGracePeriod := *podKiller.gracePeriodOverride
	if observedGracePeriod != manager.config.MaxPodGracePeriodSeconds {
		t.Errorf("Manager chose to kill pod with incorrect grace period.  Expected: %d, actual: %d", manager.config.MaxPodGracePeriodSeconds, observedGracePeriod)
	}
	// reset state
	podKiller.pod = nil
	podKiller.gracePeriodOverride = nil

	// remove memory pressure
	fakeClock.Step(20 * time.Minute)
	summaryProvider.result = summaryStatsMaker("3Gi", podStats)
	manager.synchronize(diskInfoProvider, activePodsFunc)

	// we should not have memory pressure
	if manager.IsUnderMemoryPressure() {
		t.Errorf("Manager should not report memory pressure")
	}

	// induce memory pressure!
	fakeClock.Step(1 * time.Minute)
	summaryProvider.result = summaryStatsMaker("500Mi", podStats)
	manager.synchronize(diskInfoProvider, activePodsFunc)

	// we should have memory pressure
	if !manager.IsUnderMemoryPressure() {
		t.Errorf("Manager should report memory pressure")
	}

	// check the right pod was killed
	if podKiller.pod != podToEvict {
		t.Errorf("Manager chose to kill pod: %v, but should have chosen %v", podKiller.pod.Name, podToEvict.Name)
	}
	observedGracePeriod = *podKiller.gracePeriodOverride
	if observedGracePeriod != int64(0) {
		t.Errorf("Manager chose to kill pod with incorrect grace period.  Expected: %d, actual: %d", 0, observedGracePeriod)
	}

	// the best-effort pod should not admit, burstable should
	expected = []bool{false, true}
	for i, pod := range []*v1.Pod{bestEffortPodToAdmit, burstablePodToAdmit} {
		if result := manager.Admit(&lifecycle.PodAdmitAttributes{Pod: pod}); expected[i] != result.Admit {
			t.Errorf("Admit pod: %v, expected: %v, actual: %v", pod, expected[i], result.Admit)
		}
	}

	// reduce memory pressure
	fakeClock.Step(1 * time.Minute)
	summaryProvider.result = summaryStatsMaker("2Gi", podStats)
	podKiller.pod = nil // reset state
	manager.synchronize(diskInfoProvider, activePodsFunc)

	// we should have memory pressure (because transition period not yet met)
	if !manager.IsUnderMemoryPressure() {
		t.Errorf("Manager should report memory pressure")
	}

	// no pod should have been killed
	if podKiller.pod != nil {
		t.Errorf("Manager chose to kill pod: %v when no pod should have been killed", podKiller.pod.Name)
	}

	// the best-effort pod should not admit, burstable should
	expected = []bool{false, true}
	for i, pod := range []*v1.Pod{bestEffortPodToAdmit, burstablePodToAdmit} {
		if result := manager.Admit(&lifecycle.PodAdmitAttributes{Pod: pod}); expected[i] != result.Admit {
			t.Errorf("Admit pod: %v, expected: %v, actual: %v", pod, expected[i], result.Admit)
		}
	}

	// move the clock past transition period to ensure that we stop reporting pressure
	fakeClock.Step(5 * time.Minute)
	summaryProvider.result = summaryStatsMaker("2Gi", podStats)
	podKiller.pod = nil // reset state
	manager.synchronize(diskInfoProvider, activePodsFunc)

	// we should not have memory pressure (because transition period met)
	if manager.IsUnderMemoryPressure() {
		t.Errorf("Manager should not report memory pressure")
	}

	// no pod should have been killed
	if podKiller.pod != nil {
		t.Errorf("Manager chose to kill pod: %v when no pod should have been killed", podKiller.pod.Name)
	}

	// all pods should admit now
	expected = []bool{true, true}
	for i, pod := range []*v1.Pod{bestEffortPodToAdmit, burstablePodToAdmit} {
		if result := manager.Admit(&lifecycle.PodAdmitAttributes{Pod: pod}); expected[i] != result.Admit {
			t.Errorf("Admit pod: %v, expected: %v, actual: %v", pod, expected[i], result.Admit)
		}
	}
}

```