# Overview

这篇文章是基于Kubernetes的master  commitid: 写下的源码分析文档。

此篇文档主要是围绕deployment controller的介绍以及工作原理。

代码位置 `pkg/controller/deployment`



# 数据结构

```go
type DeploymentController struct {
	// replica sets controller
	rsControl     controller.RSControlInterface
	client        clientset.Interface
    // 事件广播
	eventRecorder record.EventRecorder

	
	// used for unit testing
	enqueueDeployment func(deployment *apps.Deployment)
	
    // deployment informer去从store里面里面list/get deployment信息
	dLister appslisters.DeploymentLister
	// replicaset informer去从store里面里面list/get replicaset 信息
	rsLister appslisters.ReplicaSetLister
	// pod informer去从store里面里面list/get pod 信息
	podLister corelisters.PodLister

	...
    // workqueue队列
	queue workqueue.RateLimitingInterface
}
```



# 实例化Deployment controller

实例化deploymentcontroller，接收来自deployment Informer和 replicaset Informer的增删改事件以及pod Informer的删除事件

```go
func NewDeploymentController(dInformer appsinformers.DeploymentInformer, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
    // 事件广播器
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartStructuredLogging(0)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: client.CoreV1().Events("")})

	if client != nil && client.CoreV1().RESTClient().GetRateLimiter() != nil {
		if err := ratelimiter.RegisterMetricAndTrackRateLimiterUsage("deployment_controller", client.CoreV1().RESTClient().GetRateLimiter()); err != nil {
			return nil, err
		}
	}
    // 实例化
	dc := &DeploymentController{
		client:        client,
		eventRecorder: eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "deployment-controller"}),
		queue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "deployment"),
	}
	dc.rsControl = controller.RealRSControl{
		KubeClient: client,
		Recorder:   dc.eventRecorder,
	}

	dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addDeployment,
		UpdateFunc: dc.updateDeployment,
		DeleteFunc: dc.deleteDeployment,
	})
	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addReplicaSet,
		UpdateFunc: dc.updateReplicaSet,
		DeleteFunc: dc.deleteReplicaSet,
	})
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		DeleteFunc: dc.deletePod,
	})

	dc.syncHandler = dc.syncDeployment
	dc.enqueueDeployment = dc.enqueue

	dc.dLister = dInformer.Lister()
	dc.rsLister = rsInformer.Lister()
	dc.podLister = podInformer.Lister()
	dc.dListerSynced = dInformer.Informer().HasSynced
	dc.rsListerSynced = rsInformer.Informer().HasSynced
	dc.podListerSynced = podInformer.Informer().HasSynced
	return dc, nil
}
```



# Run

```go
func (dc *DeploymentController) Run(workers int, stopCh <-chan struct{}) {	
...

	for i := 0; i < workers; i++ {
        // 使用go协程启动worker
		go wait.Until(dc.worker, time.Second, stopCh)
	}
	<-stopCh
}
```

## worker

```go
func (dc *DeploymentController) worker() {
    // 死循环进入processNextWorkItem方法
	for dc.processNextWorkItem() {
	}
}


func (dc *DeploymentController) processNextWorkItem() bool {
	key, quit := dc.queue.Get()
	if quit {
		return false
	}
	defer dc.queue.Done(key)

    // 进入syncHandler方法处理每一个进入workqueue的事件，跟replicaset Controller类似，deployment controller在实例化的时候通过操作dc.syncHandler = dc.syncDeployment
	err := dc.syncHandler(key.(string))
	dc.handleErr(err, key)

	return true
}

```



## syncHandler --> syncDeployment

```go
func (dc *DeploymentController) syncDeployment(key string) error {
	startTime := time.Now()
...
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	...
	deployment, err := dc.dLister.Deployments(namespace).Get(name)
	...
	// Deep-copy otherwise we are mutating our cache.
	// TODO: Deep-copy only when needed.
	d := deployment.DeepCopy()

	everything := metav1.LabelSelector{}
	if reflect.DeepEqual(d.Spec.Selector, &everything) {
		dc.eventRecorder.Eventf(d, v1.EventTypeWarning, "SelectingAll", "This deployment is selecting all pods. A non-empty selector is required.")
		if d.Status.ObservedGeneration < d.Generation {
			d.Status.ObservedGeneration = d.Generation
			dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(context.TODO(), d, metav1.UpdateOptions{})
		}
		return nil
	}
	
    // 获取owner reference 属于该Deployment对象的replicaset, 在下方有详细理解getReplicaSetsForDeployment方法
	rsList, err := dc.getReplicaSetsForDeployment(d)

    // 查找属于该deployment对象的pod
	podMap, err := dc.getPodMapForDeployment(d, rsList)

...
    // 返回的是bool和error, 如果true则说明Deployment.Spec.Replicas跟实际的owner reference 的Replica数量不一致，那么需要sync去执行scale事件
	scalingEvent, err := dc.isScalingEvent(d, rsList)
	if err != nil {
		return err
	}
	if scalingEvent {
		return dc.sync(d, rsList)
	}

    // 根据deployment对象的Spec.Strategy去决定要做什么操作，是要创建replicaSet呢还是要删除replicaSet
	switch d.Spec.Strategy.Type {
	case apps.RecreateDeploymentStrategyType:
		return dc.rolloutRecreate(d, rsList, podMap)
	case apps.RollingUpdateDeploymentStrategyType:
		return dc.rolloutRolling(d, rsList)
	}
	return fmt.Errorf("unexpected deployment strategy type: %s", d.Spec.Strategy.Type)
}

```



### getReplicaSetsForDeployment

```go
func (dc *DeploymentController) getReplicaSetsForDeployment(d *apps.Deployment) ([]*apps.ReplicaSet, error) {
	// List all ReplicaSets to find those we own but that no longer match our
	// selector. They will be orphaned by ClaimReplicaSets().
    // replicaset informer去从store里面里面list 所有的replicaset 
	rsList, err := dc.rsLister.ReplicaSets(d.Namespace).List(labels.Everything())
    // 拿到该deployment对象的Spec.Selector
	deploymentSelector, err := metav1.LabelSelectorAsSelector(d.Spec.Selector)


    // 这一步是为了保证我们现在拿到的d的deployment对象是最新的（并检查它的deletionTimestamp，而不是检查它的本地缓存）
	canAdoptFunc := controller.RecheckDeletionTimestamp(func() (metav1.Object, error) {
		fresh, err := dc.client.AppsV1().Deployments(d.Namespace).Get(context.TODO(), d.Name, metav1.GetOptions{})
		
		if fresh.UID != d.UID {
			return nil, fmt.Errorf("original Deployment %v/%v is gone: got uid %v, wanted %v", d.Namespace, d.Name, fresh.UID, d.UID)
		}
		return fresh, nil
	})
    // 调用ClaimReplicaSets去返回owner reference属于该deployment对象的replicaset的list
	cm := controller.NewReplicaSetControllerRefManager(dc.rsControl, d, deploymentSelector, controllerKind, canAdoptFunc)
	return cm.ClaimReplicaSets(rsList)
}
```





# 总结

Deployment Controller是通过Spec中的描述，跟实际Status比对去更改。







从deployment controller的代码里面，我们可以看到，这又是一个典型的controller的flow: replicaset/deployment的创建/删除/更改都会写入workqueue, 然后从workqueue里面读取去执行processNextWorkItem, 去查找owner reference属于该deployment的Replicaset比对，是创建还是删除Replicaset。

