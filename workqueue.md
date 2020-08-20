# Overview

这篇文章是基于Kubernetes的v1.14.10 分支写下的源码分析文档。

此篇文档主要是围绕WorkQueue组件在`Client-go`中的介绍以及工作原理。



# 概念

WorkQueue称为工作队列，比FIFO略复杂，主要功能是**标记**和**去重**。

从下图可以看出，client-go里面的WorkQueue，起的作用类似一个chan， 当资源发生变化的时候通过回调函数将资源写入队列，由controller的worker消费者完成业务处理

![1597560260584](C:\Users\EZLIUJA\AppData\Roaming\Typora\typora-user-images\1597560260584.png)



## 特性

1. 公平原则，先进先出
2. 去重，一个工作即使被多次加入，也只会被处理一次
3. 多个消费者和多个生产者
4. 关闭通知

# 通用队列

## 数据结构

代码块`staging/src/k8s.io/client-go/util/workqueue/queue.go`

```go
type Type struct {
    // queue是一个工作队列，可以看得出是一个slice，主要作用是有序处理
	queue []t
    // dirty定义了所有需要被processed处理的items，是一个map
	dirty set
    // 标记是否正在被处理，是一个map
	processing set

	cond *sync.Cond

	shuttingDown bool

	metrics queueMetrics

	unfinishedWorkUpdatePeriod time.Duration
	clock                      clock.Clock
}

type empty struct{}
type t interface{}
type set map[t]empty
```

## 接口

`WorkQueue`的接口提供的方法如下，基本概况为可以插入元素，计算长度，获取元素等

```go
type Interface interface {
    // 给队列添加元素
	Add(item interface{})
    // 计算队列长度
	Len() int
    // 获取队列头部的一个元素
	Get() (item interface{}, shutdown bool)
    // 标记队列中该元素已被处理
	Done(item interface{})
    // 关闭队列
	ShutDown()
    // 查询是否正在被关闭
	ShuttingDown() bool
}
```

### Get方法与总结

接下来我们通过其中一个方法`Get`来看看发生了什么事情。

```go
func (q *Type) Get() (item interface{}, shutdown bool) {
    // 通过锁保证同时只有一个元素从队列头部被取出
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
...
    // 每次只取一个元素
	item, q.queue = q.queue[0], q.queue[1:]

	q.metrics.get(item)
    // 把这个元素插入入processing，通过下方的insert也可以看出，(因为是map)同一个元素只会被插入一次
    // 也可以看出，queue, processing和dirty都是在维护各自的队列中的相同元素
	q.processing.insert(item)
    // 从dirty里面去除item
	q.dirty.delete(item)

	return item, false
}

func (s set) insert(item t) {
	s[item] = empty{}
}
```

看了`Get`方法，如上所示，可以得知queue, processing和dirty都是在维护各自的队列中的相同元素

### Add方法与总结

接下来我们通过其中一个方法`Add`来看看发生了什么事情。

```go
func (q *Type) Add(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	if q.shuttingDown {
		return
	}
    // 从Get里面我们已经确认queue, processing和dirty都是在维护各自的队列中的相同元素
    // 当每次往queue里面Add一个元素的时候，下方代码都会检查dirty里面是否有这个元素，如果有直接返回，也就是标记去重的作用
	if q.dirty.has(item) {
		return
	}
    // 如果queue里面没有这个元素，那么追加到queue, dirty队列，但仍然会检查这个元素是否在processing队列中正在被处理
	q.metrics.add(item)

	q.dirty.insert(item)
	if q.processing.has(item) {
		return
	}

	q.queue = append(q.queue, item)
	q.cond.Signal()
}
```

也就是说，每次往Workqueue里面追加元素，都会检查，标记去重，保证每个元素只会被处理一次。

## 初始化

```go

func newQueue(c clock.Clock, metrics queueMetrics, updatePeriod time.Duration) *Type {
	t := &Type{
		clock:                      c,
		dirty:                      set{},
		processing:                 set{},
		cond:                       sync.NewCond(&sync.Mutex{}),
		metrics:                    metrics,
		unfinishedWorkUpdatePeriod: updatePeriod,
	}
    // 启动协程，其实作用是队列没有关闭的时候 定时同步metrics信息
	go t.updateUnfinishedWorkLoop()
	return t
}


func (q *Type) updateUnfinishedWorkLoop() {
	t := q.clock.NewTicker(q.unfinishedWorkUpdatePeriod)
	defer t.Stop()
	for range t.C() {
		if !func() bool {
			q.cond.L.Lock()
			defer q.cond.L.Unlock()
			if !q.shuttingDown {
				q.metrics.updateUnfinishedWork()
				return true
			}
			return false

		}() {
			return
		}
	}
}
```



# 延时队列

## 数据结构

代码块`staging/src/k8s.io/client-go/util/workqueue/delaying_queue.go`

从数据结构可以看出，延时队列是基于通用队列的基础上封装的

```go
type delayingType struct {
	Interface
	// clock tracks time for delayed firing
	clock clock.Clock
    // 一个缓冲的通道，提供等待添加的元素的chann
    waitingForAddCh chan *waitFor
...
}
```



## 接口

从接口可以看出，延时队列是基于通用队列的基础上封装的，加了AddAfter的方法

```go
type DelayingInterface interface {
	Interface
    // 延时添加元素
	AddAfter(item interface{}, duration time.Duration)
}
```

### AddAfter方法与总结

下面我们可以看到AddAfter的方法，根据传入的duration来决定把元素马上添加到queue中，还是插入到queue的waitingForAddCh chan中，我们记住这个chan，等会在初始化的时候会分析

```go
func (q *delayingType) AddAfter(item interface{}, duration time.Duration) {
	// don't add if we're already shutting down
	if q.ShuttingDown() {
		return
	}

	q.metrics.retry()
	q.deprecatedMetrics.retry()
	
    // 如果duration小于等于0，那么就马上将元素添加到Queue中
	if duration <= 0 {
		q.Add(item)
		return
	}

	select {
	case <-q.stopCh:
		// unblock if ShutDown() is called
        // 按调用传入的参数，将该元素添加到waitingForAddCh的chan中，这个waitFor的数据结构挺有意思的，是保存元素，并且保存readyAt的时间戳
	case q.waitingForAddCh <- &waitFor{data: item, readyAt: q.clock.Now().Add(duration)}:
	}
}

```



## 初始化

```go
func newDelayingQueue(clock clock.Clock, name string) DelayingInterface {
	ret := &delayingType{
		Interface:         NewNamed(name),
		clock:             clock,
		heartbeat:         clock.NewTicker(maxWait),
		stopCh:            make(chan struct{}),
		waitingForAddCh:   make(chan *waitFor, 1000),
		metrics:           newRetryMetrics(name),
		deprecatedMetrics: newDeprecatedRetryMetrics(name),
	}
    // 这个是重点，上面只是构造对象，然后现在使用协程去进行真正的延时添加元素到队列中
	go ret.waitingLoop()

	return ret
}

func (q *delayingType) waitingLoop() {
	defer utilruntime.HandleCrash()

	// 新建一个没有buffer的Chan 
	never := make(<-chan time.Time)
    //waitingForQueue 是一个切片的对象
	waitingForQueue := &waitForPriorityQueue{}
    // 这里挺有意思，heap是构造一个树，使用heap来实现优先队列
	heap.Init(waitingForQueue)

	waitingEntryByData := map[t]*waitFor{}

	for {
		if q.Interface.ShuttingDown() {
			return
		}

		now := q.clock.Now()
		
        // 从AddAfter的代码块，我们确认了需要延时添加的元素，是被追加到waitingForAddCh 里面的，现在waitingForQueue对象就是这个chann的实例化，通过判断waitingForQueue的长度来决定是否有需要延时添加的元素
        // 死循环，直到这个队列长度为空，一直到延时时间戳是否就是现在，不是的话就继续循环， 是的话跳出循环添加到队列里面
		for waitingForQueue.Len() > 0 {
			entry := waitingForQueue.Peek().(*waitFor)
			if entry.readyAt.After(now) {
				break
			}

            // 延时的时间到了，然后从优先队列中取出，添加到队列中，按时间顺序添加
			entry = heap.Pop(waitingForQueue).(*waitFor)
			q.Add(entry.data)
			delete(waitingEntryByData, entry.data)
		}

		// Set up a wait for the first item's readyAt (if one exists)
		nextReadyAt := never
        // 计算等待第一个要添加元素的等待时间
		if waitingForQueue.Len() > 0 {
			entry := waitingForQueue.Peek().(*waitFor)
			nextReadyAt = q.clock.After(entry.readyAt.Sub(now))
		}

		select {
		case <-q.stopCh:
			return

		case <-q.heartbeat.C():
			// continue the loop, which will add ready items

		case <-nextReadyAt:
			// continue the loop, which will add ready items

            // 获取放入 waitingForAddCh chan中的元素
		case waitEntry := <-q.waitingForAddCh:
			if waitEntry.readyAt.After(q.clock.Now()) {
				insert(waitingForQueue, waitingEntryByData, waitEntry)
			} else {
				q.Add(waitEntry.data)
			}

			drained := false
			for !drained {
				select {
				case waitEntry := <-q.waitingForAddCh:
					if waitEntry.readyAt.After(q.clock.Now()) {
						insert(waitingForQueue, waitingEntryByData, waitEntry)
					} else {
						q.Add(waitEntry.data)
					}
				default:
					drained = true
				}
			}
		}
	}
}
```



# 限速队列

代码块`staging/src/k8s.io/client-go/util/workqueue/rate_limiting_queue.go`

## 数据结构

```go
type rateLimitingType struct {
	DelayingInterface
	rateLimiter RateLimiter
}

type RateLimiter interface {	
    // 一个元素应该等多久，才可以插入队列里面
	When(item interface{}) time.Duration
	// 清空该元素的排队数
	Forget(item interface{})
	// 获取指定元素的排队数
	NumRequeues(item interface{}) int
}

```



##  接口

从接口可以看出，限速队列是基于延时队列的基础上封装的方法，加了AddRateLimited， Forget和NumRequeues的接口方法

```go
type RateLimitingInterface interface {
	DelayingInterface	
    // 该方法是等时间到把元素插入workqueue，实际仍然是调用了延时队列的AddAfter方法
	AddRateLimited(item interface{})

	// Forget indicates that an item is finished being retried.  Doesn't matter whether it's for perm failing
	// or for success, we'll stop the rate limiter from tracking it.  This only clears the `rateLimiter`, you
	// still have to call `Done` on the queue.
	Forget(item interface{})

	// NumRequeues returns back how many times the item was requeued
	NumRequeues(item interface{}) int
}

// 调用延时队列的AddAfter方法把元素插入workqueue
func (q *rateLimitingType) AddRateLimited(item interface{}) {
	q.DelayingInterface.AddAfter(item, q.rateLimiter.When(item))
}

// 
func (q *rateLimitingType) Forget(item interface{}) {
	q.rateLimiter.Forget(item)
}
```



## 限速算法



### 令通牌算法

BucketRateLimiter

是通过第三方库"golang.org/x/time/rate" 实现的

默认的清空下就实例化令牌桶实现的，以固定速率往桶里面插入元素，被插入的元素都会拿到一个token，以此来达到限制速度的目的。

```go
func DefaultControllerRateLimiter() RateLimiter {
	return NewMaxOfRateLimiter(
		NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
		// 10 qps, 100 bucket size.  This is only for retry speed and its only the overall factor (not per item)
		&BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
	)
}

func (r *BucketRateLimiter) When(item interface{}) time.Duration {
	return r.Limiter.Reserve().Delay()
}
```

(rate.Limit(10), 100)

第一个参数10表示每秒往“桶”里填充的 token 数量

第二个参数100表示令牌桶的大小（即令牌桶最多存放的 token 数量）



### 排队指数算法

ItemExponentialFailureRateLimiter

排队指数算法将**相同元素**的排队数作为指数，排队数增大，速率限制呈指数级增长，但其最大值不会超过 `maxDelay`

限速队列利用延迟队列的特性，延迟多个相同元素的插入时间，达到限速目的

```go
type ItemExponentialFailureRateLimiter struct {
    // map元素的读写锁
	failuresLock sync.Mutex
    // 元素失败次数记录
	failures     map[interface{}]int

	baseDelay time.Duration
	maxDelay  time.Duration
}
// 初始化
func NewItemExponentialFailureRateLimiter(baseDelay time.Duration, maxDelay time.Duration) RateLimiter {
	return &ItemExponentialFailureRateLimiter{
		failures:  map[interface{}]int{},
		baseDelay: baseDelay,
		maxDelay:  maxDelay,
	}
}

// 代码挺简单的，就是通过计算失败次数来计算时间，不大于最大的maxdelay时间就返回当前计算需要延时的时间
func (r *ItemExponentialFailureRateLimiter) When(item interface{}) time.Duration {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	exp := r.failures[item]
	r.failures[item] = r.failures[item] + 1

	// The backoff is capped such that 'calculated' value never overflows.
	backoff := float64(r.baseDelay.Nanoseconds()) * math.Pow(2, float64(exp))
	if backoff > math.MaxInt64 {
		return r.maxDelay
	}

	calculated := time.Duration(backoff)
	if calculated > r.maxDelay {
		return r.maxDelay
	}

	return calculated
}
```

### 



# ParallelizeUntil

代码块`staging/src/k8s.io/client-go/util/workqueue/parallelizer.go`

这个是并发worker处理协程，总共有N个pieces的任务，然后交给doWorkPiece方法去处理这些pieces任务，也就是多消费者

```go
func ParallelizeUntil(ctx context.Context, workers, pieces int, doWorkPiece DoWorkPieceFunc) {
	var stop <-chan struct{}
	if ctx != nil {
		stop = ctx.Done()
	}

	toProcess := make(chan int, pieces)
	for i := 0; i < pieces; i++ {
		toProcess <- i
	}
	close(toProcess)

	if pieces < workers {
		workers = pieces
	}

	wg := sync.WaitGroup{}
	wg.Add(workers)
	for i := 0; i < workers; i++ {
		go func() {
			defer utilruntime.HandleCrash()
			defer wg.Done()
			for piece := range toProcess {
				select {
				case <-stop:
					return
				default:
					doWorkPiece(piece)
				}
			}
		}()
	}
	wg.Wait()
}
```



# 总结

在workqueue里面可以看出K8S大部分代码都把接口拆分得足够小，然后使用组合的方法去做类似Java的继承，例如延时队列就是基于普通队列的基础上加了一个延时处理的接口方法。







# Reference

《Kubernetes 源码剖析》第五章