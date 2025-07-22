# 问题

架构师询问：我们尝试使用PVC和Deployment的时候，当我们这个Deployment需要挂载PVC的时候，是否出现了一个问题，Deployment的Pod被调度到节点A, 而节点A是属于zone 1的，PVC的磁盘却是zone 2，导致无法绑定该PVC？

这篇文章是基于Kubernetes的master  commitid:  8e8b6a01cf6bf55dea5e2e4f554597a95c82988a写下的源码分析文档。

# 问题是如何引起的

我们需要每一次部署Deployment的时候，都能正确绑定的PVC是同属于一个zone的。而PVC是在我们kubectl apply pvc.yaml的时候已经创建完成，调度器看起来没有感知到卷的位置，把Deployment调度去了另外一个zone的节点。如果真的出现这个问题的话，猜测问题应该出在调度器上。

## 调度器过程

调度器overall过程可以看[scheluler overall](/scheduler/scheduler阅读理解上.md)。 而决定一个node节点是否能满足Pod的需求的很重要的阶段是**过滤**。



### 过滤

对调度器来说，如何调用每一个enable插件在过滤阶段的过滤方法，调用链如下，也就是说在调度器过滤的阶段，每个支持过滤阶段执行的调度器插件都会被执行`Filter`方法， 

```bash
RunFilterPlugins()
	 f.runFilterPlugin(ctx, pl, state, pod, nodeInfo)
	 	pl.Filter(ctx, state, pod, nodeInfo)
```



而涉及到PVC和PV的调度器插件是`VolumeBinding`和`volumezone`，接下来我们会先看看`volumebinding `插件。

#### volumebinding 插件

`volumebinding`插件只参与了调度器的过滤环节，而`volumebinding`的`Filter`方法是调用了`predicate`方法。

```go
// 代码位置 pkg/scheduler/framework/plugins/volumebinding/volume_binding.go
func (pl *VolumeBinding) Filter(ctx context.Context, cs *framework.CycleState, pod *v1.Pod, nodeInfo *schedulernodeinfo.NodeInfo) *framework.Status {
	_, reasons, err := pl.predicate(pod, nil, nodeInfo)
	return migration.PredicateResultToFrameworkStatus(reasons, err)
}
```

`volumebinding`  插件的`predicate`方法是一个在调度器过滤阶段检查节点是否满足Pod的Volume绑定，  如果满足的话函数返回的`true`和`nil`, `nil`,  如果不满足，那么返回`false`，以及不满足的原因和错误。 插件的工作流程如下：

1. 如果pod没有PVC，就直接返回
2. 调用`FindPodVolumes` 检查节点是否满足pod的PVC要求。具体检查算法跟PVC Controller使用了同一个算法，只判断大小大于等于PVC大小的PV，并且access mode一致，并不检查disk zone。

```go
// 代码位置 pkg/scheduler/algorithm/predicates/predicates.go
func (c *VolumeBindingChecker) predicate(pod *v1.Pod, meta Metadata, nodeInfo *schedulernodeinfo.NodeInfo) (bool, []PredicateFailureReason, error) {	
    // 如果pod没有PVC，就直接返回
	if !podHasPVCs(pod) {
		return true, nil, nil
	}
	node := nodeInfo.Node()
	// 调用FindPodVolumes 检查节点是否满足pod的PVC要求
	unboundSatisfied, boundSatisfied, err := c.binder.Binder.FindPodVolumes(pod, node)
	...
	failReasons := []PredicateFailureReason{}
	if !boundSatisfied {
		failReasons = append(failReasons, ErrVolumeNodeConflict)
	}

    // 找不到匹配该pod的PV
	if !unboundSatisfied {		
		failReasons = append(failReasons, ErrVolumeBindConflict)
	}

	if len(failReasons) > 0 {
		return false, failReasons, nil
	}
    // 找到的所有的PVC都满足pod
	return true, nil, nil
}
```



`FindPodVolumes`  的工作流程如下：

查询  没有绑定的Volume和已经绑定Volume 是否满足Pod的PVC要求

1. 获取pod的所有volumes， 返回pod的PVC，返回三个参数：已经绑定的PVC， 设置延迟绑定的PVC（也就是没有被绑定的）， 没有绑定的PVC（也就是属于Immediate模式的PVC）
2. 如果`unboundClaimsImmediate`列表大于0，也就是有PVC没有绑定，那么返回false并且错误原因是pod有没有绑定的模式是immediate的PVC
3. 检查已经绑定的PV节点的亲和性，`checkBoundClaims`首先往pv缓存里面获取到pv的名字来判断PVC已经绑定的PV是否存在，然后检查节点亲和性是否所有的volumes都符合node的亲和性
4. 对于延迟绑定的PVC,也就是没有绑定的PVC列表如果大于0
   1. 首先会检查PVC的annotation，如果包含了`"volume.kubernetes.io/selected-node"`的key，那么会检查value是否就是该node的name，如果是则说明满足，反则返回false
   2. 如果`claimsToFindMatching`列表大于0，也就是说明该node满足PVC的annotation，并且有没有绑定的PVC需要等待绑定，那么使用`findMatchingVolumes`去尝试查找匹配的卷。而`findMatchingVolumes`实际上是调用了下方的PVC Controller的`FindMatchingVolume`，使用了同一个方法来作为算法，只比对了PV的容量大小，access mode是否一致。
5. 重置数据，为下一次调度做准备，包括更新绑定的缓存等

`getPodVolumes` 会查询pod的spec.volumes，来获取pod的所有volumes，然后查询这下volumes是否已经被绑定。 如果没有被绑定的Volumes会查询是否设置了`DelayBindingMode` 延时绑定模式，如果是，那么添加到延时绑定的PVC列表去返回。

```go
// 代码位置 pkg/controller/volume/scheduling/scheduler_binder.go
func (b *volumeBinder) FindPodVolumes(pod *v1.Pod, node *v1.Node) (unboundVolumesSatisfied, boundVolumesSatisfied bool, err error) {
	podName := getPodName(pod)
	unboundVolumesSatisfied = true
	boundVolumesSatisfied = true
	start := time.Now()

	var (
		matchedBindings   []*bindingInfo
		provisionedClaims []*v1.PersistentVolumeClaim
	)
	defer func() {
		// 我们为每个新的调度循环重新创建绑定。
		if len(matchedBindings) == 0 && len(provisionedClaims) == 0 {
			// 如果没有对该节点进行绑定的PV，则清除缓存。
			b.podBindingCache.ClearBindings(pod, node.Name)
			return
		}		
		if len(matchedBindings) == 0 {
			matchedBindings = nil
		}
		if len(provisionedClaims) == 0 {
			provisionedClaims = nil
		}
		// 更新cache的binding
		b.podBindingCache.UpdateBindings(pod, node.Name, matchedBindings, provisionedClaims)
	}()	
    
    // 获取pod的所有volumes， 返回pod的PVC，返回四个参数：已经绑定的PVC， 设置延迟绑定的PVC（也就是没有被绑定的）， 没有绑定的PVC（也就是属于Immediate模式的PVC）和error
	boundClaims, claimsToBind, unboundClaimsImmediate, err := b.getPodVolumes(pod)
		
    // 如果unboundClaimsImmediate列表大于0，也就是有PVC没有绑定，那么返回false并且错误原因是pod有没有绑定的模式是immediate的PVC
	if len(unboundClaimsImmediate) > 0 {
		return false, false, fmt.Errorf("pod has unbound immediate PersistentVolumeClaims")
	}	
    // 检查已经绑定的PV节点的亲和性，`checkBoundClaims`首先往pv缓存里面获取到pv的名字来判断PVC已经绑定的PV是否存在，然后检查节点亲和性是否所有的volumes都符合node的亲和性
	if len(boundClaims) > 0 {
		boundVolumesSatisfied, err = b.checkBoundClaims(boundClaims, node, podName)
		if err != nil {
			return false, false, err
		}
	}
	
    // 延迟绑定的PVC,也就是没有绑定的PVC列表如果大于0
	if len(claimsToBind) > 0 {
		var (
			claimsToFindMatching []*v1.PersistentVolumeClaim
			claimsToProvision    []*v1.PersistentVolumeClaim
		)
		
        // 首先会检查PVC的annotation，如果包含了"volume.kubernetes.io/selected-node"的key，那么会检查value是否就是该node的name，如果是则说明满足并且添加到claimsToFindMatching列表里面，反则返回false
		for _, claim := range claimsToBind {
			if selectedNode, ok := claim.Annotations[pvutil.AnnSelectedNode]; ok {
				if selectedNode != node.Name {					
					return false, boundVolumesSatisfied, nil
				}
				claimsToProvision = append(claimsToProvision, claim)
			} else {
				claimsToFindMatching = append(claimsToFindMatching, claim)
			}
		}

		// 如果claimsToFindMatching列表大于0，也就是说明该node满足PVC的annotation，并且有没有绑定的PVC需要等待绑定，那么使用findMatchingVolumes去尝试查找匹配的卷，这里特地把findMatchingVolumes详细流程在下方放出来。
		if len(claimsToFindMatching) > 0 {
			var unboundClaims []*v1.PersistentVolumeClaim
			unboundVolumesSatisfied, matchedBindings, unboundClaims, err = b.findMatchingVolumes(pod, claimsToFindMatching, node)
			if err != nil {
				return false, false, err
			}
			claimsToProvision = append(claimsToProvision, unboundClaims...)
		}

		// Check for claims to provision
		if len(claimsToProvision) > 0 {
			unboundVolumesSatisfied, provisionedClaims, err = b.checkVolumeProvisions(pod, claimsToProvision, node)
			if err != nil {
				return false, false, err
			}
		}
	}

	return unboundVolumesSatisfied, boundVolumesSatisfied, nil
}
```



#### volumezone插件

`volumezone`插件只参与了调度器的过滤环节。

过滤环节会返回三个参数：

1. 是否满足pod的需求，是返回true
2. 不满足的原因
3. 错误

工作流程如下：

1. 如果pod不需要volumes，直接返回满足过滤条件true，不再做其他检查
2. 轮询node节点的所有label， 获取key是`"failure-domain.beta.kubernetes.io/zone"`  和key是`"failure-domain.beta.kubernetes.io/region"` 的label， 添加到`nodeConstraints` map中
3. 如果`nodeConstraints`长度是空，也就是说明这个node节点没有约束，可以直接被调度。 也就是说，当我们使用zone的时候，所有的node节点都需要打上zone的label
4. 检查PVC是否存在, 如果PVC设置了延迟绑定，那么跳过该PVC
5. 获取PV的label，获取key是`"failure-domain.beta.kubernetes.io/zone"`  和key是`"failure-domain.beta.kubernetes.io/region"` 的label，跟`nodeConstraints`这个map对比，如果PV的label里面并没有node的zone label 的值的话，那么就说明该node不符合Pod的PV的要求，返回不满足false以及原因

```go
// 代码位置 pkg/scheduler/framework/plugins/volumezone/volume_zone.go
func (pl *VolumeZone) Filter(ctx context.Context, _ *framework.CycleState, pod *v1.Pod, nodeInfo *nodeinfo.NodeInfo) *framework.Status {
	// 运行 predicate去执行过滤
	_, reasons, err := pl.predicate(pod, nil, nodeInfo)
	return migration.PredicateResultToFrameworkStatus(reasons, err)
}


// 代码位置 pkg/scheduler/algorithm/predicates/predicates.go
func (c *VolumeZoneChecker) predicate(pod *v1.Pod, meta Metadata, nodeInfo *schedulernodeinfo.NodeInfo) (bool, []PredicateFailureReason, error) {
	// 如果pod不需要volumes，直接返回即可
	if len(pod.Spec.Volumes) == 0 {
		return true, nil, nil
	}

	node := nodeInfo.Node()

	nodeConstraints := make(map[string]string)
    // 轮询node节点的所有label，
    // 获取key是`"failure-domain.beta.kubernetes.io/zone"`
    // 和key是`"failure-domain.beta.kubernetes.io/region"` 的label的。添加到nodeConstraints map中
	for k, v := range node.ObjectMeta.Labels {
		if k != v1.LabelZoneFailureDomain && k != v1.LabelZoneRegion {
			continue
		}
		nodeConstraints[k] = v
	}

    // 如果nodeConstraints长度是空，也就是说明这个node节点没有约束，可以直接被调度
	if len(nodeConstraints) == 0 {		
        // 也就是说，当我们使用zone的时候，所有的node节点都需要打上zone的label
		return true, nil, nil
	}

	namespace := pod.Namespace
	manifest := &(pod.Spec)
	for i := range manifest.Volumes {
		volume := &manifest.Volumes[i]
		if volume.PersistentVolumeClaim != nil {
			pvcName := volume.PersistentVolumeClaim.ClaimName
			if pvcName == "" {
				return false, nil, fmt.Errorf("PersistentVolumeClaim had no name")
			}
			pvc, err := c.pvcLister.PersistentVolumeClaims(namespace).Get(pvcName)
			if err != nil {
				return false, nil, err
			}

			if pvc == nil {
				return false, nil, fmt.Errorf("PersistentVolumeClaim was not found: %q", pvcName)
			}

			pvName := pvc.Spec.VolumeName
            // 检查PVC是否存在
			if pvName == "" {
				scName := v1helper.GetPersistentVolumeClaimClass(pvc)
				if len(scName) > 0 {
					class, _ := c.scLister.Get(scName)
					if class != nil {
						if class.VolumeBindingMode == nil {
							return false, nil, fmt.Errorf("VolumeBindingMode not set for StorageClass %q", scName)
						}
                        // 如果设置了延迟绑定，那么跳过该PVC
						if *class.VolumeBindingMode == storage.VolumeBindingWaitForFirstConsumer {
							continue
						}
					}
				}
				return false, nil, fmt.Errorf("PersistentVolumeClaim was not found: %q", pvcName)
			}

			pv, err := c.pvLister.Get(pvName)
			if err != nil {
				return false, nil, err
			}

			if pv == nil {
				return false, nil, fmt.Errorf("PersistentVolume was not found: %q", pvName)
			}
			// 获取PV的label，获取key是`"failure-domain.beta.kubernetes.io/zone"`  和key是`"failure-domain.beta.kubernetes.io/region"` 的label，然后调用LabelZonesToSet解析pv的label
			for k, v := range pv.ObjectMeta.Labels {
				if k != v1.LabelZoneFailureDomain && k != v1.LabelZoneRegion {
					continue
				}
				nodeV, _ := nodeConstraints[k]
                // 从包含要设置的zone的分隔列表的字符串转换PV标签值
				volumeVSet, err := volumehelpers.LabelZonesToSet(v)
				if err != nil {
					klog.Warningf("Failed to parse label for %q: %q. Ignoring the label. err=%v. ", k, v, err)
					continue
				}
				// 如果PV的label里面并没有node的zone label 的值的话，那么就说明该node不符合Pod的PV的要求，返回不满足false以及原因
				if !volumeVSet.Has(nodeV) {
					klog.V(10).Infof("Won't schedule pod %q onto node %q due to volume %q (mismatch on %q)", pod.Name, node.Name, pvName, k)
					return false, []PredicateFailureReason{ErrVolumeZoneConflict}, nil
				}
			}
		}
	}

	return true, nil, nil
}
```



## PVC Controller

PVC Controller的流程如下（这里我们重点是讲Add/ Update操作）:

1. 首先从队列里面获取PVC对象
2. 如果PVC对象能从API Server获取到，就说明是属于Add/Update操作，进入下方的sync操作，否则说明是Delete操作
3. 如果PVC没有跟PV绑定（是否绑定查annotation是否有bind-completed即可查询）：
   1. 如果指定PV名称
      1. 那么会先查询PV是否存在，存在的情况下检查大小，access mode是否一致，一致即可`bind`绑定，否则返回`pending`状态
   2. 如果没有指定PV名称
      1. 首先使用`findBestMatchForClaim`查找一个已经创建的PV，在找不到的情况下会尝试通过`provisionClaim`动态创建一个PV来绑定，涉及的查找算法是`findByClaim`，判断大小大于等于PVC大小的PV，并且access mode一致。**如果有延迟绑定的PVC，不会做任何操作，直接返回。**
4. 如果PVC已经绑定PV：
   1. 如果不能获取到`Spec.VolumeName` 说明PV已经删除，那么设置PVC状态为`Lost`
   2. 如果找到PV对象，并且没有绑定，就执行`bind`重新绑定，否则说明PVC跟PV关系已经好，直接返回

以下是PVC查找PV使用的算法代码块，可以看到判断在存在PV的情况下PVC能否跟PV绑定是查询access mode以及存储大小和污点容忍情况。

```go
// 代码位置 pkg/controller/volume/persistentvolume/index.go
func (pvIndex *persistentVolumeOrderedIndex) findByClaim(claim *v1.PersistentVolumeClaim, delayBinding bool) (*v1.PersistentVolume, error) {
	// 查找匹配的access mode
	allPossibleModes := pvIndex.allPossibleMatchingAccessModes(claim.Spec.AccessModes)

	for _, modes := range allPossibleModes {
		volumes, err := pvIndex.listByAccessModes(modes)
		if err != nil {
			return nil, err
		}
		// 查找符合的Volume， FindMatchingVolume是PV controller和调度器都会调取此函数
		bestVol, err := pvutil.FindMatchingVolume(claim, volumes, nil /* node for topology binding*/, nil /* exclusion map */, delayBinding)
		if err != nil {
			return nil, err
		}

		if bestVol != nil {
			return bestVol, nil
		}
	}
	return nil, nil
}
// 代码位置 pkg/controller/volume/persistentvolume/util/util.go
func FindMatchingVolume(
	claim *v1.PersistentVolumeClaim,
	volumes []*v1.PersistentVolume,
	node *v1.Node,
	excludedVolumes map[string]*v1.PersistentVolume,
	delayBinding bool) (*v1.PersistentVolume, error) {

	var smallestVolume *v1.PersistentVolume
	var smallestVolumeQty resource.Quantity
    // 获取PVC的spec.resources.requests.storage存储大小值
	requestedQty := claim.Spec.Resources.Requests[v1.ResourceName(v1.ResourceStorage)]
	requestedClass := v1helper.GetPersistentVolumeClaimClass(claim)
		...
	// 轮询PV， 排除exclude list里面的volumes
	for _, volume := range volumes {
		if _, ok := excludedVolumes[volume.Name]; ok {			
			continue
		}
		volumeQty := volume.Spec.Capacity[v1.ResourceStorage]

		// 通过检查PV里面的spec.capacity.storage的大小，通过大小比对，我们只获取比PVC定义的storage大于等于的PV
		if CheckVolumeModeMismatches(&claim.Spec, &volume.Spec) {
			continue
		}

        // 跳过DeletionTimeStamp不等于空的PV(DeletionTimeStamp不为空说明该PV正在删除中)
		if utilfeature.DefaultFeatureGate.Enabled(features.StorageObjectInUseProtection) {
			if volume.ObjectMeta.DeletionTimestamp != nil {
				continue
			}
		}

		nodeAffinityValid := true
		if node != nil {
			// 检查污点容忍
			err := volumeutil.CheckNodeAffinity(volume, node.Labels)
			if err != nil {
				nodeAffinityValid = false
			}
		}

		if IsVolumeBoundToClaim(volume, claim) {
			if volumeQty.Cmp(requestedQty) < 0 {
				continue
			}
			if !nodeAffinityValid {
				return nil, nil
			}

			return volume, nil
		}
		// 如果是延迟绑定，直接跳过
		if node == nil && delayBinding {			
			continue
		}		
        // 检查PV状态是否可用
		if volume.Status.Phase != v1.VolumeAvailable {
			continue
		} else if volume.Spec.ClaimRef != nil {
			continue
		} else if selector != nil && !selector.Matches(labels.Set(volume.Labels)) {
			continue
		}
		if v1helper.GetPersistentVolumeClass(volume) != requestedClass {
			continue
		}
		if !nodeAffinityValid {
			continue
		}

		if node != nil {
			// j检查access mode是否match
			if !CheckAccessModes(claim, volume) {
				continue
			}
		}

		if volumeQty.Cmp(requestedQty) >= 0 {
			if smallestVolume == nil || smallestVolumeQty.Cmp(volumeQty) > 0 {
				smallestVolume = volume
				smallestVolumeQty = volumeQty
			}
		}
	}

	if smallestVolume != nil {
		// 返回符合的volume
		return smallestVolume, nil
	}

	return nil, nil
}

```



# 总结

持久卷PV是cluster scope的，可以提前由管理员手工provision，也可以使用`storage class`动态provision。以下是一个PV的具体输出，代码里面也经常出现轮询所有PV去查是`spec.claimRef`为空来判定该PV是否已经被PVC所绑定。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: foo-pv
spec:
  storageClassName: ""
  claimRef:
    name: foo-pvc
    namespace: foo
  ...
```

PVC是namespace scope的，在PVC中，我们同样也可以声明PVC和PV的绑定关系。对于PVC Controller来说，他会使用`FindMatchingVolume` 来检查PVC和PV的容量大小，access mode是否一致来绑定。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: foo-pvc
  namespace: foo
spec:
  storageClassName: "" # 此处须显式设置空字符串，否则会被设置为默认的 StorageClass
  volumeName: foo-pv
```

对PVC来说，当没有指定PV的名称去绑定的时候，也可以通过`storageClass`来动态provision，而这里一种非常值得注意的情况是，PVC Controller和调度器里面都会检查该PVC是否使用了延迟绑定，也就是`volumeBindingMode: WaitForFirstConsumer`，下面是一个设置延迟绑定的`storageClass`的例子。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-csi
provisioner: disk.csi.azure.com
parameters:
  skuname: StandardSSD_LRS  
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

当`storageClass`是使用延迟绑定的时候，调度器和PVC Controller直接跳过该PVC，直到调度完成，`Kubelet`将pod启动的时候会根据`volumes.kubernetes.io/controller-managed-attach-detach: "true"`  annotation去判断是由pvc controller去attach，保证挂载的卷和机器处于同一个zone的情况。



按照上面`volumezone`插件提到的node节点的label，下面我们使用AKS来看一个具体使用了zone的node节点信息。 这里有几个注意点，当一个node节点出现了`volumes.kubernetes.io/controller-managed-attach-detach: "true"` 的annotation的时候，也就是告诉PVC Controller，是PVC Controller来负责attach volume。而在labels里面我们看到两个重要的label: `failure-domain.beta.kubernetes.io/region: westus2` 和  `failure-domain.beta.kubernetes.io/zone: westus2-1 `  。调度器会检查annotation和label来决定该节点是否符合Pod的PVC需求。

```yaml
# kubectl get node aks-default-xx -o yaml
apiVersion: v1
kind: Node
metadata:
  annotations:
    node.alpha.kubernetes.io/ttl: "0"
    volumes.kubernetes.io/controller-managed-attach-detach: "true"
  creationTimestamp: "2020-08-03Txx"
  labels:
    agentpool: default
    ...    
    failure-domain.beta.kubernetes.io/region: westus2
    failure-domain.beta.kubernetes.io/zone: westus2-1    
  name: aks-default-xx
  resourceVersion: "133xx"
  selfLink: /api/v1/nodes/aks-default-xx
  uid: c1xx
spec:
  providerID: azure:///subscriptions/xx
status:
  addresses:
  - address: aks-default-7xx
    type: Hostname
  - address: 10.xx
    type: InternalIP  
  capacity:
    attachable-volumes-azure-disk: "8"
    cpu: "4"
    ephemeral-storage: 30xxKi
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 1xx6Ki
    pods: "30"
 ..  
```



# 如何解决问题

从上述看完调度器涉及到PV和PVC插件的代码以及PVC Controller ，在调度器里面的`volumebinding` 插件是负责检查Pod需要使用的PV大小大于等于PVC容量，并且access mode一致，并不检查disk zone，而`volumezone`插件则是保证PV的label zone是包含了节点zone，也就是说如果要保证Deployment的Pod被调度到节点A, 而节点A是属于zone 1的，PVC的磁盘也是zone 1的话，可以通过调度器的`volumezone`插件来保证。

而`volumezone`插件是从k8s  1.17的版本之后开始出现volumezone的插件，而在v1.16以及之前的版本，都没有volumezone ，具体看<https://github.com/kubernetes/kubernetes/tree/release-1.16/pkg/scheduler/algorithm/predicates>。

另外一种思路是，PVC Controller里面也写明了在PVC没有指明PV name的情况下，**如果设置延迟绑定的PVC，不会做任何操作，直接返回**， 直到`Kubelet` 开始在node节点上创建Pod的时候才开始创建卷来保证zone一致。



