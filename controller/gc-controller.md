# Overview

我们通常会通过删除一个Deployment的方法来同时删除Deployment对象以及所属
Pod资源对象， 而K8S中是怎么把所属资源也一并删除的呢？ 
没错，就是GC Controller发挥的作用，GC Controller会将被删除对象的附属资源查询并且一并删除。
垃圾回收就是从系统中删除未使用的对象，并释放分配给它们的计算资源。


与面向对象的语言不同，在K8s对象清单定义中，我们从来没有明确定义或编写与所有者相关的关系，而是系统如何确定该关系? 
在K8s中，每个从属对象都有一个唯一的元数据字段名称metas.ownerReferences用于关系表示。

从Kubernetes 1.8开始，K8为由特定控制器(例如ReplicaSet，StatefulSet，DaemonSet，Deployment，Job和CronJob)创建或采用的对象设置ownerReferences的值。
如果需要，还可以手动设置ownerReferences。
一个对象可以有多个ownerReferences，例如在namespace中。


在用于未使用对象GC的K8中，有两大类：
级联：在级联之一中，所有者的删除导致从群集中删除从属对象。
孤儿：顾名思义，对所有者对象的删除操作只会将其从集群中删除，并使所有从属对象处于"孤儿"状态。

工作原理：
dependencyGraphBuilder: 这个是一个用来查询检查K8S集群里面的资源，会无线循环执行monitor，查询集群中的资源，并且添加到dirty_queue中

runAttemptToDeleteWorker和runAttemptToOrphanWorker: 定期运行，从dirty_queue中获取，检查OwnerReference是否为空，
如果为空则取下一个继续处理，否则检查OwnerReferences元数据中的每个条目， 如果OwnerReferences中列出的所有者都不存在，那么worker会删除这个对象


数据结构
GarbageCollector里面有两个队列，一个是`attemptToDelete`，另外一个是`attemptToOrphan`
attemptToDelete： 当时机成熟时，垃圾收集器尝试删除队列attemptToDelete中的项。
attemptToOrphan： 垃圾收集器尝试使attemptToOrphan队列中的项的依赖项成为孤儿，然后删除这些项。

type GarbageCollector struct {
	restMapper     resettableRESTMapper
	metadataClient metadata.Interface
	// garbage collector attempts to delete the items in attemptToDelete queue when the time is ripe.
	attemptToDelete workqueue.RateLimitingInterface
	// garbage collector attempts to orphan the dependents of the items in the attemptToOrphan queue, then deletes the items.
	attemptToOrphan        workqueue.RateLimitingInterface
	dependencyGraphBuilder *GraphBuilder
	// GC caches the owners that do not exist according to the API server.
	absentOwnerCache *ReferenceCache

	workerLock sync.RWMutex
}

运行
func (gc *GarbageCollector) Run(workers int, stopCh <-chan struct{}) {
	// 开启go协程执行dependencyGraphBuilder.Run
	go gc.dependencyGraphBuilder.Run(stopCh)

	if !cache.WaitForNamedCacheSync("garbage collector", stopCh, gc.dependencyGraphBuilder.IsSynced) {
		return
	}
	
	klog.Infof("Garbage collector: all resource monitors have synced. Proceeding to collect garbage")
	
	// gc workers
	for i := 0; i < workers; i++ {
		// 定时执行runAttemptToDeleteWorker和runAttemptToOrphanWorker 方法
		go wait.Until(gc.runAttemptToDeleteWorker, 1*time.Second, stopCh)
		go wait.Until(gc.runAttemptToOrphanWorker, 1*time.Second, stopCh)
	}
	
	<-stopCh
}



# gc.dependencyGraphBuilder.Run


func (gb *GraphBuilder) Run(stopCh <-chan struct{}) {	
	// Set up the stop channel.
	gb.monitorLock.Lock()
	gb.stopCh = stopCh
	gb.running = true
	gb.monitorLock.Unlock()


	// 启动 monitors	 直到stop channel被关闭
	gb.startMonitors()
	// 定期执行runProcessGraphChanges
	wait.Until(gb.runProcessGraphChanges, 1*time.Second, stopCh)

}

runProcessGraphChanges是无线死循环执行processGraphChanges
func (gb *GraphBuilder) runProcessGraphChanges() {
	for gb.processGraphChanges() {
	}
}


runProcessGraphChanges 是从graphChanges队列里面出队，更新graph，填充dirty_queue。

总结：
gb这个结构体的作用是监控kubernetes中所有资源的各种事件，包括创建、更新、删除，然后将这些事件放入事件队列eventQueue中，然后启动一个worker，这个worker从事件队列eventQueue中取出事件，然后进行处理，处理的目的是维护正确的对象依赖关系。


管理队列dirtyQueue:回收控制器会将kubernetes系统中的所有对象都放入这个管理队列中，然后回收控制器会从这个dirtyQueue队列中取出对象，然后进行资源回收处理，资源回收处理的过程是并发运行的，在kube-controller-manager中默认启动5个worker来进行处理。
在比如当对象存在删除事件时，Propagator会从对象依赖关系中删除这个对象，并且把这个对象所依赖的所有对象都放入dirtyQueue队列中等待处理


比如当对象存在创建或者更新事件时，processGraphChanges方法会判断这个对象是否存在所有者，也就是说判断这个对象是否存在Owner，如果存在Owner，并且这个Owner已经不存在了，那么将这个对象放入dirtyQueue队列中等待处理。


结构体node通过这个结构体，从Owner的角度出发，可以查询到所有dependent对象

func (gb *GraphBuilder) processGraphChanges() bool {
	//  从graphChanges 队列中出队，获取队头的对象
	item, quit := gb.graphChanges.Get()
	// 获取成功后，将该对象从队列里面移除
	defer gb.graphChanges.Done(item)
	// 获取该对象的event， 这个event不是从informer里面直接过来的，是GC controller再次构造了Event信息
	event, ok := item.(*event)
	
	// 获取到event的对象
	obj := event.obj
	
	accessor, err := meta.Accessor(obj)
	
	klog.V(5).Infof("GraphBuilder process object: %s/%s, namespace %s, name %s, uid %s, event type %v, virtual=%v", event.gvk.GroupVersion().String(), event.gvk.Kind, accessor.GetNamespace(), accessor.GetName(), string(accessor.GetUID()), event.eventType, event.virtual)	
	// 检查node是否存在， 这里的node 其实是gb里面的一个数据结构
	existingNode, found := gb.uidToNode.Read(accessor.GetUID())
	if found && !event.virtual && !existingNode.isObserved() {
		// this marks the node as having been observed via an informer event
		// 1. this depends on graphChanges only containing add/update events from the actual informer
		// 2. this allows things tracking virtual nodes' existence to stop polling and rely on informer events
		// identityFromEvent 返回对象的属主OwnerReference
		observedIdentity := identityFromEvent(event, accessor)
		if observedIdentity != existingNode.identity {
			// 找到与我们观察到的身份不匹配的依赖者			
			_, potentiallyInvalidDependents := partitionDependents(existingNode.getDependents(), observedIdentity)
			// add those potentially invalid dependents to the attemptToDelete queue.
			// if their owners are still solid the attemptToDelete will be a no-op.
			// this covers the bad child -> good parent observation sequence.
			// the good parent -> bad child observation sequence is handled in addDependentToOwners
			for _, dep := range potentiallyInvalidDependents {
				if len(observedIdentity.Namespace) > 0 && dep.identity.Namespace != observedIdentity.Namespace {
					// Namespace mismatch, this is definitely wrong
					klog.V(2).Infof("node %s references an owner %s but does not match namespaces", dep.identity, observedIdentity)
					gb.reportInvalidNamespaceOwnerRef(dep, observedIdentity.UID)
				}
				gb.attemptToDelete.Add(dep)
			}
	
			// make a copy (so we don't modify the existing node in place), store the observed identity, and replace the virtual node			
			klog.V(2).Infof("replacing virtual node %s with observed node %s", existingNode.identity, observedIdentity)
			existingNode = existingNode.clone()
			existingNode.identity = observedIdentity
			gb.uidToNode.Write(existingNode)
		}
		existingNode.markObserved()
	}
	
	switch {
	// 注意后面的!found, 也就是一开始gb.uidToNode.Read(accessor.GetUID()) 里面没有找到这个node
	case (event.eventType == addEvent || event.eventType == updateEvent) && !found:
		newNode := &node{
			identity:           identityFromEvent(event, accessor),
			dependents:         make(map[*node]struct{}),
			owners:             accessor.GetOwnerReferences(),
			deletingDependents: beingDeleted(accessor) && hasDeleteDependentsFinalizer(accessor),
			beingDeleted:       beingDeleted(accessor),
		}
		gb.insertNode(newNode)
		// the underlying delta_fifo may combine a creation and a deletion into
		// one event, so we need to further process the event.
		gb.processTransitions(event.oldObj, accessor, newNode)
	case (event.eventType == addEvent || event.eventType == updateEvent) && found:
		// handle changes in ownerReferences
		added, removed, changed := referencesDiffs(existingNode.owners, accessor.GetOwnerReferences())
		if len(added) != 0 || len(removed) != 0 || len(changed) != 0 {
			// check if the changed dependency graph unblock owners that are
			// waiting for the deletion of their dependents.
			gb.addUnblockedOwnersToDeleteQueue(removed, changed)
			// update the node itself
			existingNode.owners = accessor.GetOwnerReferences()
			// Add the node to its new owners' dependent lists.
			gb.addDependentToOwners(existingNode, added)
			// remove the node from the dependent list of node that are no longer in
			// the node's owners list.
			gb.removeDependentFromOwners(existingNode, removed)
		}
	
		if beingDeleted(accessor) {
			existingNode.markBeingDeleted()
		}
		gb.processTransitions(event.oldObj, accessor, existingNode)
	case event.eventType == deleteEvent:
		if !found {
			klog.V(5).Infof("%v doesn't exist in the graph, this shouldn't happen", accessor.GetUID())
			return true
		}
	
		removeExistingNode := true
	
		if event.virtual {
			// this is a virtual delete event, not one observed from an informer
			deletedIdentity := identityFromEvent(event, accessor)
			if existingNode.virtual {
	
				// our existing node is also virtual, we're not sure of its coordinates.
				// see if any dependents reference this owner with coordinates other than the one we got a virtual delete event for.
				if matchingDependents, nonmatchingDependents := partitionDependents(existingNode.getDependents(), deletedIdentity); len(nonmatchingDependents) > 0 {
	
					// some of our dependents disagree on our coordinates, so do not remove the existing virtual node from the graph
					removeExistingNode = false
	
					if len(matchingDependents) > 0 {
						// mark the observed deleted identity as absent
						gb.absentOwnerCache.Add(deletedIdentity)
						// attempt to delete dependents that do match the verified deleted identity
						for _, dep := range matchingDependents {
							gb.attemptToDelete.Add(dep)
						}
					}
	
					// if the delete event verified existingNode.identity doesn't exist...
					if existingNode.identity == deletedIdentity {
						// find an alternative identity our nonmatching dependents refer to us by
						replacementIdentity := getAlternateOwnerIdentity(nonmatchingDependents, deletedIdentity)
						if replacementIdentity != nil {
							// replace the existing virtual node with a new one with one of our other potential identities
							replacementNode := existingNode.clone()
							replacementNode.identity = *replacementIdentity
							gb.uidToNode.Write(replacementNode)
							// and add the new virtual node back to the attemptToDelete queue
							gb.attemptToDelete.AddRateLimited(replacementNode)
						}
					}
				}
	
			} else if existingNode.identity != deletedIdentity {
				// do not remove the existing real node from the graph based on a virtual delete event
				removeExistingNode = false
	
				// our existing node which was observed via informer disagrees with the virtual delete event's coordinates
				matchingDependents, _ := partitionDependents(existingNode.getDependents(), deletedIdentity)
	
				if len(matchingDependents) > 0 {
					// mark the observed deleted identity as absent
					gb.absentOwnerCache.Add(deletedIdentity)
					// attempt to delete dependents that do match the verified deleted identity
					for _, dep := range matchingDependents {
						gb.attemptToDelete.Add(dep)
					}
				}
			}
		}
	
		if removeExistingNode {
			// removeNode updates the graph
			gb.removeNode(existingNode)
			existingNode.dependentsLock.RLock()
			defer existingNode.dependentsLock.RUnlock()
			if len(existingNode.dependents) > 0 {
				gb.absentOwnerCache.Add(identityFromEvent(event, accessor))
			}
			for dep := range existingNode.dependents {
				gb.attemptToDelete.Add(dep)
			}
			for _, owner := range existingNode.owners {
				ownerNode, found := gb.uidToNode.Read(owner.UID)
				if !found || !ownerNode.isDeletingDependents() {
					continue
				}
				// this is to let attempToDeleteItem check if all the owner's
				// dependents are deleted, if so, the owner will be deleted.
				gb.attemptToDelete.Add(ownerNode)
			}
		}
	}
	return true
}


runAttemptToDeleteWorker


runAttemptToOrphanWorker





