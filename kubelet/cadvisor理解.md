# Overview

`cAdvisor`是谷歌开发的容器监控工具，可以对机器以及容器进行实时监控和数据采集，包括CPU、内存、文件系统等情况，cAdvisor是集成在`Kubelet` 每台worker node节点上都会自行启动`cAdvisor`, 主要代码位于`./pkg/kubelet/cadvisor`目录下。

# 接口

cadvisor调用的cadvisor方法有以下几种，像`ContainerInfo`，`SubcontainerInfo`和`MachineInfo`这些都是cadvisor本身已经提供的方法，`Kubelet`无需再实现或者简单封装，而`WatchEvents`这些

```go
type Interface interface {
	Start() error    
	DockerContainer(...)
	ContainerInfo(..)
	ContainerInfoV2(...)
	SubcontainerInfo(...)
	MachineInfo() (...)
	VersionInfo() (...)
	// 返回image信息
	ImagesFsInfo() (cadvisorapiv2.FsInfo, error)
	// 返回根的文件系统信息
	RootFsInfo() (cadvisorapiv2.FsInfo, error)
	// 从channel里面获取event信息
	WatchEvents(request *events.Request) (*events.EventChannel, error)
	//获取指定目录的文件系统信息 
	GetDirFsInfo(path string) (cadvisorapiv2.FsInfo, error)
}
```

那么`Kubelet`在什么时候如何使用这些接口呢？

`Kubelet` 在自身启动运行的时候获取机器信息，以及OOM模块中使用了`cAdvisor`去监听OOM事件





# OOM

OOM 是out of memory的意思，是Kubernetes使用google的cadvisor监控container的资源使用情况，一旦container out of memory， 则使用client-go的record把事件记录。

目前在Kubernetes只支持Linux操作系统的OOM record, 其他操作系统均不支持。

目前的一个设计缺陷是，一旦Kubernetes的某个pod对象因为OOM而被kill掉，那么event消息则不再保存，因为pod已经被kill掉再启动一个新对象了。我们需要把OOM Kill掉pod的event统一收集到central metrics之类的才可以看到此类情况。

```go
func (ow *realWatcher) Start(ref *v1.ObjectReference) error {
	oomLog, err := oomparser.New()
	...
	outStream := make(chan *oomparser.OomInstance, 10)
	go oomLog.StreamOoms(outStream)

	go func() {
		defer runtime.HandleCrash()

		for event := range outStream {
			if event.ContainerName == "/" {
				klog.V(1).Infof("Got sys oom event: %v", event)
				eventMsg := "System OOM encountered"
				if event.ProcessName != "" && event.Pid != 0 {
					eventMsg = fmt.Sprintf("%s, victim process: %s, pid: %d", eventMsg, event.ProcessName, event.Pid)
				}
                // 使用client-go的recorder记录事件消息
				ow.recorder.PastEventf(ref, metav1.Time{Time: event.TimeOfDeath}, v1.EventTypeWarning, systemOOMEvent, eventMsg)
			}
		}
		klog.Errorf("Unexpectedly stopped receiving OOM notifications")
	}()
	return nil
}

```







```
docker run \
--volume=/:/rootfs:ro \
--volume=/var/run:/var/run:rw \
--volume=/sys/fs/cgroup/cpu,cpuacct:/sys/fs/cgroup/cpuacct,cpu \
--volume=/var/lib/docker/:/var/lib/docker:ro \
--publish=8080:8080 \
--detach=true \
--name=cadvisor \
--privileged=true \
google/cadvisor:latest
```





