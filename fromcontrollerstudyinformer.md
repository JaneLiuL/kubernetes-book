# Overview

此篇文章记录于2020-8-4，是因为看了《Kubernetes 源码剖析》第五章之后，做了一个理解，写完Informer-study.md之后，仍然觉得理解不够深刻，翻开controller的代码去理解Informer。

# Controller 数据结构

代码目录`staging/src/k8s.io/client-go/tools/cache/controller.go`

我是按着代码位置来，Config这个数据结构其实是存储一个`Controller`的所有配置

构造controller的配置文件，构造process，即HandleDeltas，该函数为后面使用到的process函数。

```go
type Config struct {
    // 对象的队列，也就是我们的DeltaFIFO。 Process()函数会接收DeltaFIFO消费出来的对象。
	Queue
	// 熟悉的配方
	ListerWatcher
	// 当对象被Pop出来的时候同时触发Process函数
	Process ProcessFunc	
	ObjectType runtime.Object
    // 全量同步周期，目前是使用random随机
	FullResyncPeriod time.Duration
...
}
```





```go
type controller struct {
	config         Config
	reflector      *Reflector
	reflectorMutex sync.RWMutex
	clock          clock.Clock
}

type Controller interface {
    // 核心业务
	Run(stopCh <-chan struct{})
	HasSynced() bool
	LastSyncResourceVersion() string
}
// 使用New通过提供一个Config去初始化一个Controller对象
func New(c *Config) Controller {
	ctlr := &controller{
		config: *c,
		clock:  &clock.RealClock{},
	}
	return ctlr
}
```

# Controller -- Run

解下来我们看看`controller`的`Run`的实现



```go
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
    // 除非收到信号，否则死循环运行这个函数
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
    // 创建Reflector
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.clock = c.clock

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group
	defer wg.Wait()

    // 这里是重点，创建Reflector就启动了r.Run，也就是Reflector的Run
    // Reflector.Run我们在<Informer-study.md>里面有详细说明
	wg.StartWithChannel(stopCh, r.Run)

    // 然后触发processLoop
	wait.Until(c.processLoop, time.Second, stopCh)
}
```

从代码可以看出`controller`的`run`的主要目的是`reflector`一直在往`DeltaFIFO`中存数据, 另外一边是一直从`DeltaFIFO`中出队并且给自定义用户逻辑`c.config.Process`处理.

![1597446104889](C:\Users\EZLIUJA\AppData\Roaming\Typora\typora-user-images\1597446104889.png)





# Controller -- processLoop

```go
func (c *controller) processLoop() {
	for {
        // 当对象被Pop出来消费的时候，就传递给PopProcessFunc执行
        // c.config.Queue也就是Deltafifo.Pop
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
...
		}
	}
}

type PopProcessFunc func(interface{}) error
```

`Pop`其实就是Deltafifo的消费者方法，具体放在<informer-study.md>





# 总结

在controller的初始化的时候就初始化了我们的Reflector， controller.Run里面Reflector的主要作用是watch指定的资源，并且将变化同步到本地的`store`中。

Reflector接着执行ListAndWatch函数，ListAndWatch第一次会列出所有的对象，并获取资源对象的版本号，然后watch资源对象的版本号来查看是否有被变更。首先会将资源版本号设置为0，`list()`可能会导致本地的缓存相对于etcd里面的内容存在延迟，`Reflector`会通过`watch`的方法将延迟的部分补充上，使得本地的缓存数据与etcd的数据保持一致。（这部分其实需要对着这个文章以及<informer-study.md>一起看）

controller.Run函数还会调用processLoop函数，processLoop通过调用HandleDeltas，再调用distribute，processorListener.add最终将不同更新类型的对象加入`processorListener`的channel中，供processorListener.Run使用。

processor的主要功能就是记录了所有的回调函数实例(即 ResourceEventHandler 实例)，并负责触发这些函数。processor记录了不同类型的事件函数，其中事件函数在NewXxxController构造函数部分注册，具体事件函数的处理，一般是将需要处理的对象加入对应的controller的任务队列中，然后由类似syncDeployment的同步函数来维持期望状态的同步逻辑。









# Reference

《Kubernetes 源码剖析》第五章