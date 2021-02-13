

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