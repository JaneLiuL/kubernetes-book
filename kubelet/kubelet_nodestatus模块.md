

# 运行

在`Kubelet`运行程序的时候，其中有一步是开启go协程**定期**同步节点状态，此篇文档是讲解同步节点状态以及nodestatus模块。

```go
// 代码位置 pkg/kubelet/kubelet.go
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	...	
    // 开启go协程定期同步节点状态
	go wait.Until(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, wait.NeverStop)
	...
}
```



`syncNodeStatus` 主要做了两个事情

1. 如果节点没有注册，那么往APIServer注册节点信息
2. 更新节点状态

```go
// 代码位置 pkg/kubelet/kubelet_node_status.go
func (kl *Kubelet) syncNodeStatus() {
	...
	// 往APIServer注册节点
	if kl.registerNode {
		kl.registerWithAPIServer()
	}
    // 更新节点信息
	if err := kl.updateNodeStatus(); err != nil {
		klog.Errorf("Unable to update node status: %v", err)
	}
}
```

# 注册节点

注册节点的工作流程如下

1. 查看注册状态是否是已经完成状态，如果是则退出注册节点
2. 进入循环，尝试构造v1.Node对象，并且使用该对象尝试往API Server注册节点信息
   1. 尝试使用client去create节点信息，如果创建成功则直接返回
   2.  如果返回错是节点已经存在，那么直接返回
   3. 尝试使用nodeName往API Server获取是否有已经存在，如果有错误或者无法获取到该节点信息，说明获取不到，也就是注册失败
   4. 如果可以获取到跟nodeName同名的节点对象信息，那么复制该对象，并且开始进入调和，使用patch更新节点信息
3. 如果注册成功设置registrationCompleted是true，否则继续循环tryRegisterWithAPIServer

```go
func (kl *Kubelet) registerWithAPIServer() {
	if kl.registrationCompleted {
		return
	}
	step := 100 * time.Millisecond

	for {
		time.Sleep(step)
		step = step * 2
		if step >= 7*time.Second {
			step = 7 * time.Second
		}
		// initialNode是构造初始v1.Node对象，会默认带上os的label，是否有污点的label, 初始化node的时候，都会设置NodeNetworkUnavailable是true等
		node, err := kl.initialNode(context.TODO())
		...
		klog.Infof("Attempting to register node %s", node.Name)
        // 尝试注册node信息，具体可以看下方
		registered := kl.tryRegisterWithAPIServer(node)
        // 如果返回true，也就说明注册成功，设置registrationCompleted是true，否则继续循环tryRegisterWithAPIServer
		if registered {
			kl.registrationCompleted = true
			return
		}
	}
}

func (kl *Kubelet) tryRegisterWithAPIServer(node *v1.Node) bool {
    // 尝试使用client去create节点信息，如果创建成功则直接返回
	_, err := kl.kubeClient.CoreV1().Nodes().Create(node)
	if err == nil {
		return true
	}
	// 如果返回错是节点已经存在，那么直接返回
	if !apierrors.IsAlreadyExists(err) {
		klog.Errorf("Unable to register node %q with API server: %v", kl.nodeName, err)
		return false
	}
    // 进入下方说明没有创建节点成功，错误信息并非已经存在该节点，尝试查看是否能获取到同nodeName的节点
	// 尝试使用nodeName往API Server获取是否有已经存在，如果有错误或者无法获取到该节点信息，说明获取不到，也就是注册失败
	existingNode, err := kl.kubeClient.CoreV1().Nodes().Get(string(kl.nodeName), metav1.GetOptions{})
	if err != nil {
		klog.Errorf("Unable to register node %q with API server: error getting existing node: %v", kl.nodeName, err)
		return false
	}    
	if existingNode == nil {
		klog.Errorf("Unable to register node %q with API server: no node instance returned", kl.nodeName)
		return false
	}

    // 如果可以获取到跟nodeName同名的节点对象信息，那么复制该对象
	originalNode := existingNode.DeepCopy()
	klog.Infof("Node %s was previously registered", kl.nodeName)
	
    // 尝试调和annotation的值
	requiresUpdate := kl.reconcileCMADAnnotationWithExistingNode(node, existingNode)
	requiresUpdate = kl.updateDefaultLabels(node, existingNode) || requiresUpdate
	requiresUpdate = kl.reconcileExtendedResource(node, existingNode) || requiresUpdate
	if requiresUpdate {
		if _, _, err := nodeutil.PatchNodeStatus(kl.kubeClient.CoreV1(), types.NodeName(kl.nodeName), originalNode, existingNode); err != nil {
			klog.Errorf("Unable to reconcile node %q with API server: error updating node: %v", kl.nodeName, err)
			return false
		}
	}

	return true
}

```



# 更新节点状态

调用链：

```go
func (kl *Kubelet) updateNodeStatus() error {
	klog.V(5).Infof("Updating node status")
    // 循环5次尝试更新节点状态
	for i := 0; i < nodeStatusUpdateRetry; i++ {
		if err := kl.tryUpdateNodeStatus(i); err != nil {
			if i > 0 && kl.onRepeatedHeartbeatFailure != nil {
				kl.onRepeatedHeartbeatFailure()
			}
			klog.Errorf("Error updating node status, will retry: %v", err)
		} else {
			return nil
		}
	}
	return fmt.Errorf("update node status exceeds retry count")
}

func (kl *Kubelet) tryUpdateNodeStatus(tryNumber int) error {	
	opts := metav1.GetOptions{}
    // 如果是第一次尝试更新节点状态，那么从APIServer的缓存获取信息
	if tryNumber == 0 {
		util.FromApiserverCache(&opts)
	}     
    // 获取节点对象
	node, err := kl.heartbeatClient.CoreV1().Nodes().Get(string(kl.nodeName), opts)
	// 将节点对象深度复制
	originalNode := node.DeepCopy()	
	podCIDRChanged := false
	if len(node.Spec.PodCIDRs) != 0 {
   // 将node.Spec.PodCIDRs更新到podCIDRs
		podCIDRs := strings.Join(node.Spec.PodCIDRs, ",")
		if podCIDRChanged, err = kl.updatePodCIDR(podCIDRs); err != nil {
			klog.Errorf(err.Error())
		}
	}
	// 设置节点状态
	kl.setNodeStatus(node)

	now := kl.clock.Now()
	if now.Before(kl.lastStatusReportTime.Add(kl.nodeStatusReportFrequency)) {
		if !podCIDRChanged && !nodeStatusHasChanged(&originalNode.Status, &node.Status) {
			// 
			kl.volumeManager.MarkVolumesAsReportedInUse(node.Status.VolumesInUse)
			return nil
		}
	}

	// Patch the current status on the API server
	updatedNode, _, err := nodeutil.PatchNodeStatus(kl.heartbeatClient.CoreV1(), types.NodeName(kl.nodeName), originalNode, node)
	if err != nil {
		return err
	}
	kl.lastStatusReportTime = now
	kl.setLastObservedNodeAddresses(updatedNode.Status.Addresses)
	// If update finishes successfully, mark the volumeInUse as reportedInUse to indicate
	// those volumes are already updated in the node's status
	kl.volumeManager.MarkVolumesAsReportedInUse(updatedNode.Status.VolumesInUse)
	return nil
}
```

## 设置节点状态

调用链关系如下：

```bash
updateNodeStatus()
	tryUpdateNodeStatus()
		setNodeStatus()
			setNodeStatusFuncs()
				defaultNodeStatusFuncs() # 在NewMainKubelet实例化的时候指出klet.setNodeStatusFuncs = klet.defaultNodeStatusFuncs()
```

接下来让我们看看`defaultNodeStatusFuncs`是如何设置节点状态的， 

`nodestatus.NodeAddress`是返回一个更新地址相关的func

`nodestatus.MachineInfo`是调用了`cAdvisor`等返回一个更新机器信息相关例如`ResourceCPU`  `ResourceMemory`等的func

`nodestatus.VersionInfo`是一个返回更新版本信息相关包括`KernelVersion` `KubeletVersion`等的func

`nodestatus.DaemonEndpoints` 是一个返回更新`DaemonEndpoints`的func， 也就是更新`node.Status.DaemonEndpoints`

`nodestatus.Images`是返回一个更新节点上所有的image名字，tag的切片的func，也就是更新`node.Status.Images`

```go
func (kl *Kubelet) defaultNodeStatusFuncs() []func(*v1.Node) error {
	// if cloud is not nil, we expect the cloud resource sync manager to exist
	var nodeAddressesFunc func() ([]v1.NodeAddress, error)
    // 如果是使用例如AWS/azure之类的cloud，那么使用cloudResourceSyncManager去获取node的地址
	if kl.cloud != nil {
		nodeAddressesFunc = kl.cloudResourceSyncManager.NodeAddresses
	}
    
	var validateHostFunc func() error
	if kl.appArmorValidator != nil {
		validateHostFunc = kl.appArmorValidator.ValidateHost
	}
	var setters []func(n *v1.Node) error
	// setters是一个保存func的切片
	setters = append(setters,
		nodestatus.NodeAddress(kl.nodeIP, kl.nodeIPValidator, kl.hostname, kl.hostnameOverridden, kl.externalCloudProvider, kl.cloud, nodeAddressesFunc),
		nodestatus.MachineInfo(string(kl.nodeName), kl.maxPods, kl.podsPerCore, kl.GetCachedMachineInfo, kl.containerManager.GetCapacity,
			kl.containerManager.GetDevicePluginResourceCapacity, kl.containerManager.GetNodeAllocatableReservation, kl.recordEvent),
		nodestatus.VersionInfo(kl.cadvisor.VersionInfo, kl.containerRuntime.Type, kl.containerRuntime.Version),
		nodestatus.DaemonEndpoints(kl.daemonEndpoints),
		nodestatus.Images(kl.nodeStatusMaxImages, kl.imageManager.GetImageList),
		nodestatus.GoRuntime(),
	)	
    // 返回volumeLimits的func
	setters = append(setters, nodestatus.VolumeLimits(kl.volumePluginMgr.ListVolumePluginWithLimits))

	setters = append(setters,
		nodestatus.MemoryPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderMemoryPressure, kl.recordNodeStatusEvent),
		nodestatus.DiskPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderDiskPressure, kl.recordNodeStatusEvent),
		nodestatus.PIDPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderPIDPressure, kl.recordNodeStatusEvent),
		nodestatus.ReadyCondition(kl.clock.Now, kl.runtimeState.runtimeErrors, kl.runtimeState.networkErrors, kl.runtimeState.storageErrors, validateHostFunc, kl.containerManager.Status, kl.recordNodeStatusEvent),
		nodestatus.VolumesInUse(kl.volumeManager.ReconcilerStatesHasBeenSynced, kl.volumeManager.GetVolumesInUse),		
		kl.recordNodeSchedulableEvent,
	)
	return setters
}
```

我们这里放一个`nodestatus.Images` 的具体代码来看看， 在上方调用的时候已经指出`imageListFunc = kl.imageManager.GetImageList`, 也就是从`imageCache`里面获取所有的images， 然后把该切片信息更新到`node.Status.Images`

```go
func Images(nodeStatusMaxImages int32,
	imageListFunc func() ([]kubecontainer.Image, error), // typically Kubelet.imageManager.GetImageList
) Setter {
	return func(node *v1.Node) error {
		// Update image list of this node
		var imagesOnNode []v1.ContainerImage
		containerImages, err := imageListFunc()
		if err != nil {
			node.Status.Images = imagesOnNode
			return fmt.Errorf("error getting image list: %v", err)
		}		
		if int(nodeStatusMaxImages) > -1 &&
			int(nodeStatusMaxImages) < len(containerImages) {
			containerImages = containerImages[0:nodeStatusMaxImages]
		}

		for _, image := range containerImages {
			names := append(image.RepoDigests, image.RepoTags...)
			if len(names) > MaxNamesPerImageInNodeStatus {
				names = names[0:MaxNamesPerImageInNodeStatus]
			}
			imagesOnNode = append(imagesOnNode, v1.ContainerImage{
				Names:     names,
				SizeBytes: image.Size,
			})
		}

		node.Status.Images = imagesOnNode
		return nil
	}
}
```

### 更新节点状态

上述的setters返回的是更新node.status.xx的func切片，然后回到`setNodeStatus`路线，循环返回的setter, 让f(node)去执行操作去更新节点状态

```go
func (kl *Kubelet) setNodeStatus(node *v1.Node) {
	for i, f := range kl.setNodeStatusFuncs {
		klog.V(5).Infof("Setting node status at position %v", i)
		if err := f(node); err != nil {
			klog.Errorf("Failed to set some node status fields: %s", err)
		}
	}
}
```

# 总结

nodestatus模块是一个负责注册节点，更新node.status相关信息的模块。