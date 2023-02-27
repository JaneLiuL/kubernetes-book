# overview
这篇文档围绕scheduler组建的leader选举为中心，对scheduler其他逻辑并没有太多侧重点。
scheduler这个组件是有leader选举的，并且是不使用etcd来实现leader election.


# 探索&&问题
之前使用kubespray provision了一个ha cluster, 是2个node 来作为master node, 3个worker node。 
我一直以为两个master node的scheduler 是独立运行，没有哪个为leader的时候，我看了一下scheduler pod的日志，大概如下所示
```
[instance.go:274] Using reconciler: lease
[lease.go:235] Resetting endpoints for master service "kubernetes" to 10.xx.xx.xx
```
然后当时公司同事问了一下lease,以及scheduler leader选举的一些问题
使用如下命令去查看了一下endpoint, 居然发现没有endpoint, 但是有leader选举的日志
```
kubectl get ep -n kube-system
```
同事提示了一下要看lease, 发现跟我以前看的逻辑还是有点区别的, `kubectl describe lease kube-scheduler -n kube-system 
`
```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2022-11-30T15:37:15Z"
  labels:
    k8s.io/component: kube-scheduler
    kubernetes.io/hostname: kind-control-plane
  name: kube-scheduler-c4vwjftbvpc5os2vvzle4qg27a
  namespace: kube-system
  resourceVersion: "18171"
  uid: d6c68901-4ec5-4385-b1ef-2d783738da6c
spec:
  Acquire Time: 2022-11-30T18:04:27.912073Z
  holderIdentity: kube-scheduler-c4vwjftbvpc5os2vvzle4qg27a_9cbf54e5-1136-44bd-8f9a-1dcd15c346b4
  leaseDurationSeconds: 3600
  lease transitions: 1
  renewTime: "2022-11-30T18:14:27.912073Z"
```

粗略的把以前看的时候的图上传上来了，如下所示
![](./images/ep-leader-algorithm.png)

那就让我们继续带着问题出发，我们的问题分别是：
1. ha cluster, 另外一个master 的scheduler是在什么情况下怎么获取leader的
2. why use lease instead of endpoint
3. 为什么只有一个master node的时候，也要默认开启leader


# 查看配置
首先我们查看一下scheduler 启动的yaml文件，我们发现两个master node的scheduler都是默认启动了leader选举
```
--leader-elect=true
```
然后我们开始查看我们的kubernetes 源码配置，发现跟leader选举的启动参数有以下几个

```go
type LeaderElectionConfiguration struct {

    // 是否开启选举功能，默认开启
	LeaderElect bool
	// 锁的失效时间，类似于 session-timeout
	LeaseDuration metav1.Duration
	// leader 的心跳间隔，必须小于等于 lease-duration
	RenewDeadline metav1.Duration
	// leader-elect-retry-period: non-leader 
	RetryPeriod metav1.Duration
	// 用什么对象来存放选主信息, 可以用 lease, ep, configmap
	ResourceLock string
	// lease/endpoint 的名称，kube-scheduler
	ResourceName string
	// lease/ep/configmap放哪个namespace
	ResourceNamespace string
}
```

查看了源码，位置： https://github.com/kubernetes/kubernetes/blob/release-1.25/cmd/kube-scheduler/app/options/options.go#L91
可以看出，1.25版本的kubernetes scheduler组件默认是使用lease作为锁
```go
func NewOptions() *Options {
	o := &Options{
	
		LeaderElection: &componentbaseconfig.LeaderElectionConfiguration{
			LeaderElect:       true,
			LeaseDuration:     metav1.Duration{Duration: 15 * time.Second},
			RenewDeadline:     metav1.Duration{Duration: 10 * time.Second},
			RetryPeriod:       metav1.Duration{Duration: 2 * time.Second},
			ResourceLock:      "leases",
			ResourceName:      "kube-scheduler",
			ResourceNamespace: "kube-system",
		},
	}
```
当然代码也提供了覆盖功能，如果在启动参数配置`leader-elect-resource-lock`是可以配置成endpoint的，代码就不展示了，位置在https://github.com/kubernetes/kubernetes/blob/release-1.25/cmd/kube-scheduler/app/options/options.go#L166


# 原理
## 锁结构
```go
type LeaderElectionRecord struct {
	// HolderIdentity is the ID that owns the lease. If empty, no one owns this lease and
	// all callers may acquire. Versions of this library prior to Kubernetes 1.14 will not
	// attempt to acquire leases with empty identities and will wait for the full lease
	// interval to expire before attempting to reacquire. This value is set to empty when
	// a client voluntarily steps down.
	HolderIdentity       string      `json:"holderIdentity"`
	LeaseDurationSeconds int         `json:"leaseDurationSeconds"`
    // Leader 第一次成功获得租约时的时间戳
	AcquireTime          metav1.Time `json:"acquireTime"`
	RenewTime            metav1.Time `json:"renewTime"`
    // leader 更换
	LeaderTransitions    int         `json:"leaderTransitions"`
}
```

## 发起选leader
现在我们有 lease, endpoint, configmap等resourcelock, 这些锁需要实现以下接口, 代码链接 https://github.com/kubernetes/client-go/blob/release-1.25/tools/leaderelection/resourcelock/interface.go
```go
type Interface interface {
	// Get returns the LeaderElectionRecord
	Get(ctx context.Context) (*LeaderElectionRecord, []byte, error)

	// Create attempts to create a LeaderElectionRecord
	Create(ctx context.Context, ler LeaderElectionRecord) error

	// Update will update and existing LeaderElectionRecord
	Update(ctx context.Context, ler LeaderElectionRecord) error

	// RecordEvent is used to record events
	RecordEvent(string)

	// Identity will return the locks Identity
	Identity() string

	// Describe is used to convert details on current resource lock
	// into a string
	Describe() string
}
```

以 LeaseResourceLock 的选举过程为例：
```go
func (ll *LeaseLock) Create(ctx context.Context, ler LeaderElectionRecord) error {
	var err error
	ll.lease, err = ll.Client.Leases(ll.LeaseMeta.Namespace).Create(ctx, &coordinationv1.Lease{
		ObjectMeta: metav1.ObjectMeta{
			Name:      ll.LeaseMeta.Name,
			Namespace: ll.LeaseMeta.Namespace,
		},
		Spec: LeaderElectionRecordToLeaseSpec(&ler),
	}, metav1.CreateOptions{})
	return err
}
func LeaderElectionRecordToLeaseSpec(ler *LeaderElectionRecord) coordinationv1.LeaseSpec {
	leaseDurationSeconds := int32(ler.LeaseDurationSeconds)
	leaseTransitions := int32(ler.LeaderTransitions)
	return coordinationv1.LeaseSpec{
		HolderIdentity:       &ler.HolderIdentity,
		LeaseDurationSeconds: &leaseDurationSeconds,
		AcquireTime:          &metav1.MicroTime{ler.AcquireTime.Time},
		RenewTime:            &metav1.MicroTime{ler.RenewTime.Time},
		LeaseTransitions:     &leaseTransitions,
	}
}

// Update will update an existing Lease spec.
func (ll *LeaseLock) Update(ctx context.Context, ler LeaderElectionRecord) error {
	if ll.lease == nil {
		return errors.New("lease not initialized, call get or create first")
	}
	ll.lease.Spec = LeaderElectionRecordToLeaseSpec(&ler)

	lease, err := ll.Client.Leases(ll.LeaseMeta.Namespace).Update(ctx, ll.lease, metav1.UpdateOptions{})
	if err != nil {
		return err
	}

	ll.lease = lease
	return nil
}

```



### scheduler Run 
位置 https://github.com/kubernetes/kubernetes/blob/release-1.25/cmd/kube-scheduler/app/server.go#L146
首先scheduler 在通过启动参数，构建了cc这个对象，如果启动了leader election, 他就开始进入构建 leaderElector对象，并且通过`leaderElector.Run(ctx)` 来开始进入选举
```go
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
	// remove some useless logic here...

	// If leader election is enabled, runCommand via LeaderElector until done and exit.
	if cc.LeaderElection != nil {
		cc.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
			OnStartedLeading: func(ctx context.Context) {
				close(waitingForLeader)
				sched.Run(ctx)
			},
			OnStoppedLeading: func() {
				select {
				case <-ctx.Done():
					// We were asked to terminate. Exit 0.
					klog.InfoS("Requested to terminate, exiting")
					os.Exit(0)
				default:
					// We lost the lock.
					klog.ErrorS(nil, "Leaderelection lost")
					klog.FlushAndExit(klog.ExitFlushTimeout, 1)
				}
			},
		}
		leaderElector, err := leaderelection.NewLeaderElector(*cc.LeaderElection)
		if err != nil {
			return fmt.Errorf("couldn't create leader elector: %v", err)
		}

		leaderElector.Run(ctx)

		return fmt.Errorf("lost lease")
	}

	// Leader election is disabled, so runCommand inline until done.
	close(waitingForLeader)
	sched.Run(ctx)
	return fmt.Errorf("finished without leader elect")
}

```
现在我们来看看 `leaderElector.Run(ctx)` 发生了什么事情， 会先判断是否获得锁， 如果

```go
// Run starts the leader election loop. Run will not return
// before leader election loop is stopped by ctx or it has
// stopped holding the leader lease
func (le *LeaderElector) Run(ctx context.Context) {
	defer runtime.HandleCrash()
	defer func() {
		le.config.Callbacks.OnStoppedLeading()
	}()

	if !le.acquire(ctx) {
		return // ctx signalled done
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	go le.config.Callbacks.OnStartedLeading(ctx)
	le.renew(ctx)
}

// acquire loops calling tryAcquireOrRenew and returns true immediately when tryAcquireOrRenew succeeds.
// Returns false if ctx signals done.
func (le *LeaderElector) acquire(ctx context.Context) bool {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	succeeded := false
	desc := le.config.Lock.Describe()
	klog.Infof("attempting to acquire leader lease %v...", desc)
	wait.JitterUntil(func() {
		succeeded = le.tryAcquireOrRenew(ctx)
		le.maybeReportTransition()
		if !succeeded {
			klog.V(4).Infof("failed to acquire lease %v", desc)
			return
		}
		le.config.Lock.RecordEvent("became leader")
		le.metrics.leaderOn(le.config.Name)
		klog.Infof("successfully acquired lease %v", desc)
		cancel()
	}, le.config.RetryPeriod, JitterFactor, true, ctx.Done())
	return succeeded
}

// tryAcquireOrRenew tries to acquire a leader lease if it is not already acquired,
// else it tries to renew the lease if it has already been acquired. Returns true
// on success else returns false.
func (le *LeaderElector) tryAcquireOrRenew(ctx context.Context) bool {
	now := metav1.Now()
	leaderElectionRecord := rl.LeaderElectionRecord{
		HolderIdentity:       le.config.Lock.Identity(),
		LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
		RenewTime:            now,
		AcquireTime:          now,
	}

	// 1. obtain or create the ElectionRecord
	oldLeaderElectionRecord, oldLeaderElectionRawRecord, err := le.config.Lock.Get(ctx)
	if err != nil {
		if !errors.IsNotFound(err) {
			klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
			return false
		}
		if err = le.config.Lock.Create(ctx, leaderElectionRecord); err != nil {
			klog.Errorf("error initially creating leader election record: %v", err)
			return false
		}

		le.setObservedRecord(&leaderElectionRecord)

		return true
	}

	// 2. Record obtained, check the Identity & Time
	if !bytes.Equal(le.observedRawRecord, oldLeaderElectionRawRecord) {
		le.setObservedRecord(oldLeaderElectionRecord)

		le.observedRawRecord = oldLeaderElectionRawRecord
	}
	if len(oldLeaderElectionRecord.HolderIdentity) > 0 &&
		le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
		!le.IsLeader() {
		klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
		return false
	}

	// 3. We're going to try to update. The leaderElectionRecord is set to it's default
	// here. Let's correct it before updating.
	if le.IsLeader() {
		leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
	} else {
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
	}

	// update the lock itself
	if err = le.config.Lock.Update(ctx, leaderElectionRecord); err != nil {
		klog.Errorf("Failed to update lock: %v", err)
		return false
	}

	le.setObservedRecord(&leaderElectionRecord)
	return true
}

```

## 一开始怎么判断谁是leader
# Q1: 另外一个master 的scheduler是在什么情况下怎么获取leader的
# Q2: why use lease instead of endpoint
# Q3: 为什么只有一个master node的时候，也要默认开启leader