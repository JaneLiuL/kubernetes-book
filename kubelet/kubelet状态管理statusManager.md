# Overview

statusManager模块

# 实例化

```go
// 代码位置 pkg/kubelet/kubelet.go
func NewMainKubelet(...) (*Kubelet, error) {
	...
    // 实例化statusManager
	klet.statusManager = status.NewManager(klet.kubeClient, klet.podManager, klet)	
}

// 代码位置 pkg/kubelet/status/status_manager.go
func NewManager(kubeClient clientset.Interface, podManager kubepod.Manager, podDeletionSafety PodDeletionSafetyProvider) Manager {
	return &manager{
		kubeClient:        kubeClient,
		podManager:        podManager,
		podStatuses:       make(map[types.UID]versionedPodStatus),
		podStatusChannel:  make(chan podStatusSyncRequest, 1000), // Buffer up to 1000 statuses
		apiStatusVersions: make(map[kubetypes.MirrorPodUID]uint64),
		podDeletionSafety: podDeletionSafety,
	}
}
```

# 启动

```go
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	..	
    // 启动statusManager
	kl.statusManager.Start()
	...
}
```





```go
func (m *manager) Start() {
	...
    // 使用ticker定时器
	syncTicker := time.Tick(syncPeriod)	
    // 启动一个go协程永远运行
	go wait.Forever(func() {
		select {
         // 如果可以从podStatusChannel channel消费消息，那么进入syncPod去同步
		case syncRequest := <-m.podStatusChannel:			
			m.syncPod(syncRequest.podUID, syncRequest.status)
        // 定时批量同步    
		case <-syncTicker:
			m.syncBatch()
		}
	}, 0)
}
```















