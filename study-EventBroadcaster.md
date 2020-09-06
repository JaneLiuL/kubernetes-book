# Overview

这篇文章主要是学习EventBroadcaster的原理以及设计。

# 概念

Event在Kubernetes是一种资源对象，跟Pod, Deployment一样，都是存储在etcd中的一种资源对象。Event跟其他资源对象不同的是，它是负责记录集群中发生的事件的一种资源对象，例如一个Pod的增加，这在Kubernetes中是一个事件，会以Event记录并且记录到etcd中。我们如果执行`kubectl describe pod <pod-name>`可以看到该pod对象发生了哪些事件。

注意： 为了不挤爆etcd， 在Kubernetes中只保存近一个小时的Event.



# 数据结构

代码位置`staging/src/k8s.io/api/core/v1/types.go`， 我们从Event的数据结构可以看出，主要是存储了在哪个时间点哪个资源对象发生了什么事情。

这里的`Type`是分成两种，一种是Normal， 一种是Warning

```go
type Event struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ObjectMeta `json:"metadata" protobuf:"bytes,1,opt,name=metadata"`
	
	InvolvedObject ObjectReference `json:"involvedObject" protobuf:"bytes,2,opt,name=involvedObject"`	
	Reason string `json:"reason,omitempty" protobuf:"bytes,3,opt,name=reason"`
	Message string `json:"message,omitempty" protobuf:"bytes,4,opt,name=message"`
	Source EventSource `json:"source,omitempty" protobuf:"bytes,5,opt,name=source"`
	FirstTimestamp metav1.Time `json:"firstTimestamp,omitempty" protobuf:"bytes,6,opt,name=firstTimestamp"`
	LastTimestamp metav1.Time `json:"lastTimestamp,omitempty" protobuf:"bytes,7,opt,name=lastTimestamp"`
	Count int32 `json:"count,omitempty" protobuf:"varint,8,opt,name=count"`
	Type string `json:"type,omitempty" protobuf:"bytes,9,opt,name=type"`
	EventTime metav1.MicroTime `json:"eventTime,omitempty" protobuf:"bytes,10,opt,name=eventTime"`
	Series *EventSeries `json:"series,omitempty" protobuf:"bytes,11,opt,name=series"`
	Action string `json:"action,omitempty" protobuf:"bytes,12,opt,name=action"`
	Related *ObjectReference `json:"related,omitempty" protobuf:"bytes,13,opt,name=related"`
	ReportingController string `json:"reportingComponent" protobuf:"bytes,14,opt,name=reportingComponent"`
	ReportingInstance string `json:"reportingInstance" protobuf:"bytes,15,opt,name=reportingInstance"`
}
```

我们可以使用`kubectl get events`看看在这个小时，`default`的namespace发生了哪些`Event`。

```bash
# kubectl get events
LAST SEEN   TYPE      REASON                    OBJECT                                        MESSAGE
2m13s       Warning   FailedGetResourceMetric   horizontalpodautoscaler/azure-vote-back-hpa   missing request for cpu
7m34s       Warning   ErrImageNeverPull         pod/discovery                                 Container image "webtest" is not present with pull policy of Never
2m34s       Warning   Failed                    pod/discovery                                 Error: ErrImageNeverPull

```



# EventBroadcaster事件管理机制

代码包位置`staging/src/k8s.io/client-go/tools/record`

我们先来学习在EventBroadcaster中几个关键的部分

## EventRecorder

Event生产者， Kubernetes 通过EventRecorder记录关键性事件。

我们来看看EventRecorder的接口

```go
type EventRecorder interface {
	// 记录刚刚发生的事件
	Event(object runtime.Object, eventtype, reason, message string)	
    // 类似printf，是用来格式化输出Event
	Eventf(object runtime.Object, eventtype, reason, messageFmt string, args ...interface{})
	// 类似Eventf， 只是增加注解Annotations	
	AnnotatedEventf(object runtime.Object, annotations map[string]string, eventtype, reason, messageFmt string, args ...interface{})
}
```



## EventBroadcaster

Event消费者， 也称事件广播器，EventBroadcaster消费EventRecorder记录的事件，并且将它分发给目前已经连接的broadcasterWatcher。



```go
type EventBroadcaster interface {
	// StartEventWatcher starts sending events received from this EventBroadcaster to the given
	// event handler function. The return value can be ignored or used to stop recording, if
	// desired.
	StartEventWatcher(eventHandler func(*v1.Event)) watch.Interface

	// StartRecordingToSink starts sending events received from this EventBroadcaster to the given
	// sink. The return value can be ignored or used to stop recording, if desired.
	StartRecordingToSink(sink EventSink) watch.Interface

	// StartLogging starts sending events received from this EventBroadcaster to the given logging
	// function. The return value can be ignored or used to stop recording, if desired.
	StartLogging(logf func(format string, args ...interface{})) watch.Interface

	// StartStructuredLogging starts sending events received from this EventBroadcaster to the structured
	// logging function. The return value can be ignored or used to stop recording, if desired.
	StartStructuredLogging(verbosity klog.Level) watch.Interface

	// NewRecorder returns an EventRecorder that can be used to send events to this EventBroadcaster
	// with the event source set to the given event source.
	NewRecorder(scheme *runtime.Scheme, source v1.EventSource) EventRecorder

	// Shutdown shuts down the broadcaster
	Shutdown()
}
```



EventBroadcaster 通过NewBroadcaster函数进行实例化

```go
func NewBroadcaster() EventBroadcaster {
	return &eventBroadcasterImpl{
		Broadcaster:   watch.NewBroadcaster(maxQueuedEvents, watch.DropIfChannelFull),
		sleepDuration: defaultSleepDuration,
	}
```



## broadcasterWatcher

观察者管理，用来定义事件的处理方式，例如上报事件到API Server







下面是一段来自statefulset controller的代码

```go
	// 实例化EventBroadcaster，用来消费Event
    eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartStructuredLogging(0)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})

	// 事件记录器，Kubernetes 通过EventRecorder记录关键性事件
	recorder := eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "statefulset-controller"})
```





