# Overview

这篇文章是基于Kubernetes的master  commitid: 写下的源码分析文档。

此篇文档主要是围绕namespace controller的介绍以及工作原理。

代码位置 `pkg/controller/replicaset`



# 数据结构

在ReplicaSetController的数据结构定义中，最重要的字段是rsLister 和 podLister， rsLister是ReplicaSets的存储对象，而podLister是pods的存储对象，这两个都是通过share informer传递给NewReplicaSetController工作

```go
type ReplicaSetController struct {
...
	kubeClient clientset.Interface
	podControl controller.PodControlInterface
    // syncHander是一个自定义的函数
    syncHandler func(rsKey string) error
	rsLister appslisters.ReplicaSetLister
	rsListerSynced cache.InformerSynced

	// A store of pods, populated by the shared informer passed to NewReplicaSetController
	podLister corelisters.PodLister
	podListerSynced cache.InformerSynced    
...
}
```

# 实例化ReplicaSetController

```go
func NewReplicaSetController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {
    // 开启k8s的event监听，把事件记录到event广播
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartStructuredLogging(0)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
    // NewBaseController是把clientset和事件广播传递去初始化ReplicaSetController实例
	return NewBaseController(rsInformer, podInformer, kubeClient, burstReplicas,
		apps.SchemeGroupVersion.WithKind("ReplicaSet"),
		"replicaset_controller",
		"replicaset",
		controller.RealPodControl{
			KubeClient: kubeClient,
			Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "replicaset-controller"}),
		},
	)
}

func NewBaseController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int,
	gvk schema.GroupVersionKind, metricOwnerName, queueName string, podControl controller.PodControlInterface) *ReplicaSetController {
...

	rsc := &ReplicaSetController{
		GroupVersionKind: gvk,
		kubeClient:       kubeClient,
		podControl:       podControl,
		burstReplicas:    burstReplicas,
		expectations:     controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
		queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), queueName),
	}

    // Replicaset controller监听了replica资源对象的增删改事件的变化
	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    rsc.addRS,
		UpdateFunc: rsc.updateRS,
		DeleteFunc: rsc.deleteRS,
	})
	rsc.rsLister = rsInformer.Lister()
	rsc.rsListerSynced = rsInformer.Informer().HasSynced

    // 同时， Replicaset controller监听了pod资源对象的增删改事件的变化
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: rsc.addPod,
		UpdateFunc: rsc.updatePod,
		DeleteFunc: rsc.deletePod,
	})
	rsc.podLister = podInformer.Lister()
	rsc.podListerSynced = podInformer.Informer().HasSynced

    // 这里非常重要！！ 记住！
	rsc.syncHandler = rsc.syncReplicaSet

	return rsc
}
```



## rsc的增删改

```go

func (rsc *ReplicaSetController) addRS(obj interface{}) {
	rs := obj.(*apps.ReplicaSet)
	klog.V(4).Infof("Adding %s %s/%s", rsc.Kind, rs.Namespace, rs.Name)
	rsc.enqueueRS(rs)
}

func (rsc *ReplicaSetController) updateRS(old, cur interface{}) {
	oldRS := old.(*apps.ReplicaSet)
	curRS := cur.(*apps.ReplicaSet)

    // 如果两个replica的UID不一致的话就不可以再保留旧的replicaset了， 把新的replicaset的对象插入到Workqueue里面	
	if curRS.UID != oldRS.UID {
		key, err := controller.KeyFunc(oldRS)
		if err != nil {
			utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", oldRS, err))
			return
		}
        // call deleteRS去把旧的replicaset的对象删掉操作插入到workqueue里面
		rsc.deleteRS(cache.DeletedFinalStateUnknown{
			Key: key,
			Obj: oldRS,
		})
	}

	if *(oldRS.Spec.Replicas) != *(curRS.Spec.Replicas) {
		klog.V(4).Infof("%v %v updated. Desired pod count change: %d->%d", rsc.Kind, curRS.Name, *(oldRS.Spec.Replicas), *(curRS.Spec.Replicas))
	}
    // 把新的replicaset的对象插入到Workqueue里面
	rsc.enqueueRS(curRS)
}

func (rsc *ReplicaSetController) enqueueRS(rs *apps.ReplicaSet) {
	key, err := controller.KeyFunc(rs)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", rs, err))
		return
	}

    // 把资源对象加到workqueue里面
	rsc.queue.Add(key)
}
```



这里我们复习下workqueue, workqueue是存储了资源对象以及他的操作方法的queue

## pod的增删改

```go
func (rsc *ReplicaSetController) addPod(obj interface{}) {
	pod := obj.(*v1.Pod)

    // 如果pod的DeletionTimestamp是非空，说明这个pod被清理/清理中，那么更新expectations， 然后插入Workqueue中，
	if pod.DeletionTimestamp != nil {		
		rsc.deletePod(pod)
		return
	}

	// If it has a ControllerRef, that's all that matters.
	if controllerRef := metav1.GetControllerOf(pod); controllerRef != nil {
		rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
		if rs == nil {
			return
		}
		rsKey, err := controller.KeyFunc(rs)
		if err != nil {
			return
		}
		klog.V(4).Infof("Pod %s created: %#v.", pod.Name, pod)
		rsc.expectations.CreationObserved(rsKey)
		rsc.queue.Add(rsKey)
		return
	}

	rss := rsc.getPodReplicaSets(pod)
	if len(rss) == 0 {
		return
	}
	klog.V(4).Infof("Orphan Pod %s created: %#v.", pod.Name, pod)
	for _, rs := range rss {
		rsc.enqueueRS(rs)
	}
}
```



## Run

我们从官网的client-go的图去看controller，现在上方我们是确定了pod跟Replicaset对象的增删改会把资源对象插入到workqueue, 我们进入到Run函数看它如何去实现ProcessItem

![1598519716510](C:\Users\EZLIUJA\AppData\Roaming\Typora\typora-user-images\1598519716510.png)

```go
// Run begins watching and syncing.
func (rsc *ReplicaSetController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer rsc.queue.ShutDown()

	controllerName := strings.ToLower(rsc.Kind)	
	if !cache.WaitForNamedCacheSync(rsc.Kind, stopCh, rsc.podListerSynced, rsc.rsListerSynced) {
		return
	}
    // 使用协程启动worker，然后跳到下方worker()  方法
	for i := 0; i < workers; i++ {       
		go wait.Until(rsc.worker, time.Second, stopCh)
	}

	<-stopCh
}

func (rsc *ReplicaSetController) worker() {
    // 死循环进入最重要的processNextWorkItem
	for rsc.processNextWorkItem() {
	}
}

func (rsc *ReplicaSetController) processNextWorkItem() bool {
	key, quit := rsc.queue.Get()
	if quit {
		return false
	}
	defer rsc.queue.Done(key)

    // 进入最重要的syncHandler方法
	err := rsc.syncHandler(key.(string))
	if err == nil {
		rsc.queue.Forget(key)
		return true
	}

	utilruntime.HandleError(fmt.Errorf("sync %q failed with %v", key, err))
	rsc.queue.AddRateLimited(key)

	return true
}

```



## syncHandler --> syncReplicaSet

从上方的Run的方法，我们可以看到从workqueue里面去调到syncHandler方法，因为从最开始的数据结构以及实例化，我们已经确定了rsc.syncHandler = rsc.syncReplicaSet

所以我们现在分析下syncReplicaSet方法做了什么动作

```go
func (rsc *ReplicaSetController) syncReplicaSet(key string) error {
	startTime := time.Now()
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
...
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
...

	rsNeedsSync := rsc.expectations.SatisfiedExpectations(key)
	selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
...

	// list all pods to include the pods that don't match the rs`s selector
	// anymore but has the stale controller ref.
	// TODO: Do the List and Filter in a single pass, or use an index.
	allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
	if err != nil {
		return err
	}
	// Ignore inactive pods.
	filteredPods := controller.FilterActivePods(allPods)

	// NOTE: filteredPods are pointing to objects from cache - if you need to
	// modify them, you need to copy it first.
	filteredPods, err = rsc.claimPods(rs, selector, filteredPods)
	if err != nil {
		return err
	}

	var manageReplicasErr error
    // 把所有active的pod， 并且(从selector看)owner reference 是该replica的pod资源对象，跟该replicaset对象交给manageReplicas方法处理
	if rsNeedsSync && rs.DeletionTimestamp == nil {
		manageReplicasErr = rsc.manageReplicas(filteredPods, rs)
	}
	rs = rs.DeepCopy()
	newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)
	
    // 通过manageReplicasErr处理结果的比对去更详细replicaSet的状态
	updatedRS, err := updateReplicaSetStatus(rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)
...
	// Resync the ReplicaSet after MinReadySeconds as a last line of defense to guard against clock-skew.
	if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
		updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
		updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
		rsc.queue.AddAfter(key, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
	}
	return manageReplicasErr
}
```

从上方的代码我们可以看到，syncReplicaSet这个方法主要是找到该replicaset对象以及owner reference是属于该replicaset 的pod，去交给manageRelicas去处理，通过处理结果的比对去更详细replicaSet的状态



## manageReplicas



```go
func (rsc *ReplicaSetController) manageReplicas(filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
    // 比对owner reference是我们replicaSet的pods 跟 replica.Spec.Replicas的数量差
	diff := len(filteredPods) - int(*(rs.Spec.Replicas))
	rsKey, err := controller.KeyFunc(rs)
	...
    // 如果是小于0，则代表pod不够，需要增加
	if diff < 0 {
		diff *= -1
		if diff > rsc.burstReplicas {
			diff = rsc.burstReplicas
		}
		rsc.expectations.ExpectCreations(rsKey, diff)
		
		successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
            // 交给pod controller去创建owner reference是该replicaset的pod
			err := rsc.podControl.CreatePodsWithControllerRef(rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
			if err != nil {
				if errors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
					return nil
				}
			}
			return err
		})

		// Any skipped pods that we never attempted to start shouldn't be expected.
		// The skipped pods will be retried later. The next controller resync will
		// retry the slow start process.
		if skippedPods := diff - successfulCreations; skippedPods > 0 {
			klog.V(2).Infof("Slow-start failure. Skipping creation of %d pods, decrementing expectations for %v %v/%v", skippedPods, rsc.Kind, rs.Namespace, rs.Name)
			for i := 0; i < skippedPods; i++ {
				// Decrement the expected number of creates because the informer won't observe this pod
				rsc.expectations.CreationObserved(rsKey)
			}
		}
		return err
        // 如果多，那么就删除pod
	} else if diff > 0 {
		if diff > rsc.burstReplicas {
			diff = rsc.burstReplicas
		}
		klog.V(2).InfoS("Too many replicas", "replicaSet", klog.KObj(rs), "need", *(rs.Spec.Replicas), "deleting", diff)

		relatedPods, err := rsc.getIndirectlyRelatedPods(rs)
		utilruntime.HandleError(err)

		// Choose which Pods to delete, preferring those in earlier phases of startup.
		podsToDelete := getPodsToDelete(filteredPods, relatedPods, diff)

		rsc.expectations.ExpectDeletions(rsKey, getPodKeys(podsToDelete))

		errCh := make(chan error, diff)
		var wg sync.WaitGroup
		wg.Add(diff)
		for _, pod := range podsToDelete {
			go func(targetPod *v1.Pod) {
				defer wg.Done()
				if err := rsc.podControl.DeletePod(rs.Namespace, targetPod.Name, rs); err != nil {
					// Decrement the expected number of deletes because the informer won't observe this deletion
					podKey := controller.PodKey(targetPod)
					rsc.expectations.DeletionObserved(rsKey, podKey)
					if !apierrors.IsNotFound(err) {
						klog.V(2).Infof("Failed to delete %v, decremented expectations for %v %s/%s", podKey, rsc.Kind, rs.Namespace, rs.Name)
						errCh <- err
					}
				}
			}(pod)
		}
		wg.Wait()

		select {
		case err := <-errCh:
			// all errors have been reported before and they're likely to be the same, so we'll only return the first one we hit.
			if err != nil {
				return err
			}
		default:
		}
	}

	return nil
}

```



# 总结

replicaSet controller做的事情比较简单，就是比对owner reference 的pod是自己的数量跟 replicaset.Spec.replica的数量差，去决定是增加还是删除pod