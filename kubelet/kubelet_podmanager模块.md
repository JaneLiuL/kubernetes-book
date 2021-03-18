# Overview

Kubelet的PodManager模块是负责管理存储和访问Pod， 维护静态Pod和Mirror Pod的Mapping关系。

# 背景资料

## 静态Pod

从非API Server创建出来的Pod都叫静态Pod。静态Pod是只会绑定到一个特定的node上。

如果Pod的Anntation的`"kubernetes.io/config.source"`的值不等于`"api"` 就说明是静态Pod。

```go
// 代码位置 pkg/kubelet/types/pod_update.go
func IsStaticPod(pod *v1.Pod) bool {
	source, err := GetPodSource(pod)
	return err == nil && source != ApiserverSource
}
```

如何创建静态Pod呢？我们可以查询我们节点Kubelet的配置静态Pod的目录`staticPodPath` 对应的目录，我本地的目录是` /etc/kubernetes/manifests`， 因此我在该目录下创建以下文件， 即可被kubelet创建出来。 同样的，如果我们要删除静态Pod, 就删除我们创建的/etc/kubernetes/manifests/static-web.yaml文件即可。

```bash
cat <<EOF >/etc/kubernetes/manifests/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP
EOF
```



## Mirror Pod

静态Pod是不能被API Server来控制的， 为了让用户或者Kubelet通过API Server来管理静态Pod, Kubelet会创建一个叫Mirror Pod，Mirror Pod其实跟Static Pod是一一对应的。

如何查询一个Pod是否是Mirror Pod呢，Mirror Pod是带了`"kubernetes.io/config.mirror"`的Annotation

```go
// 代码位置 pkg/kubelet/types/pod_update.go
func IsMirrorPod(pod *v1.Pod) bool {
	_, ok := pod.Annotations[ConfigMirrorAnnotationKey]
	return ok
}
```

来看一个具体的Static Pod例子, Annotation里面会自动带上`kubernetes.io/config.mirror` 这个key：

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: xx
    kubernetes.io/config.hash: xx
    kubernetes.io/config.mirror: xx
    kubernetes.io/config.seen: "2021-01-18T04:27:29.959402482+01:00"
    kubernetes.io/config.source: file
  creationTimestamp: "2021-01-18T03:28:55Z"
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver-xx
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=...  
```



## 获取Pod 来源

查询Pod的Annotation， key是`"kubernetes.io/config.source"` 的值对应就是这个Pod的来源

```go
// 代码位置 pkg/kubelet/types/pod_update.go
func GetPodSource(pod *v1.Pod) (string, error) {
	if pod.Annotations != nil {
		if source, ok := pod.Annotations[ConfigSourceAnnotationKey]; ok {
			return source, nil
		}
	}
	return "", fmt.Errorf("cannot get source of pod %q", pod.UID)
}
```



## 静态Pod和Mirror  pod Mapping

通过一个Mirror Pod获取它对应的Static Pod，首先查询Mirror Pod的FullName， 然后查询`basicManager`对象的`podByFullName` map可得。

Pod的Full Name其实就是 `pod.Name + "_" + pod.Namespace`

```go
// 代码位置 pkg/kubelet/pod/pod_manager.go
func (pm *basicManager) GetPodByMirrorPod(mirrorPod *v1.Pod) (*v1.Pod, bool) {
	pm.lock.RLock()
	defer pm.lock.RUnlock()
	pod, ok := pm.podByFullName[kubecontainer.GetPodFullName(mirrorPod)]
	return pod, ok
}
```

# 实例化

`Kubelet` 在实例化的时候，会调用`NewBasicMirrorClient` 和 `NewBasicPodManager` 去实例化podManager。

```go
// 代码位置 pkg/kubelet/kubelet.go
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,...){
    ...
    // podManager is also responsible for keeping secretManager and configMapManager contents up-to-date.
	mirrorPodClient := kubepod.NewBasicMirrorClient(klet.kubeClient, string(nodeName), nodeLister)
	klet.podManager = kubepod.NewBasicPodManager(mirrorPodClient, secretManager, configMapManager, checkpointManager)
}

```

实例化的时候，传入了`secretManager`和`configMapManager` ，这是因为当Pod需要使用`secret`或者`configmap`的时候，需要Podmanager创建对应的监听，监听资源的变化来写入缓存

```go
// 代码位置 pkg/kubelet/pod/pod_manager.go
func NewBasicPodManager(client MirrorClient, secretManager secret.Manager, configMapManager configmap.Manager, cpm checkpointmanager.CheckpointManager) Manager {
	pm := &basicManager{}
	pm.secretManager = secretManager
	pm.configMapManager = configMapManager
	pm.checkpointManager = cpm
	pm.MirrorClient = client
	pm.SetPods(nil)
	return pm
}
```



Kubelet有三种Pod来源，分别是File, http和API Server。

```go
// 代码位置 pkg/kubelet/kubelet.go
func makePodSourceConfig(kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *Dependencies, nodeName types.NodeName, bootstrapCheckpointPath string) (*config.PodConfig, error) {
	manifestURLHeader := make(http.Header)
	if len(kubeCfg.StaticPodURLHeader) > 0 {
		for k, v := range kubeCfg.StaticPodURLHeader {
			for i := range v {
                // 添加header
				manifestURLHeader.Add(k, v[i])
			}
		}
	}

	// source of all configuration
	cfg := config.NewPodConfig(config.PodConfigNotificationIncremental, kubeDeps.Recorder)
	
    // 定义 StaticPodPath 目录
	if kubeCfg.StaticPodPath != "" {		
		config.NewSourceFile(kubeCfg.StaticPodPath, nodeName, kubeCfg.FileCheckFrequency.Duration, cfg.Channel(kubetypes.FileSource))
	}

	// 定义 http 来源URL
	if kubeCfg.StaticPodURL != "" {	
		config.NewSourceURL(kubeCfg.StaticPodURL, manifestURLHeader, nodeName, kubeCfg.HTTPCheckFrequency.Duration, cfg.Channel(kubetypes.HTTPSource))
	}

	// Restore from the checkpoint path
	// NOTE: This MUST happen before creating the apiserver source
	// below, or the checkpoint would override the source of truth.

	var updatechannel chan<- interface{}
	if bootstrapCheckpointPath != "" {
		klog.Infof("Adding checkpoint path: %v", bootstrapCheckpointPath)
		updatechannel = cfg.Channel(kubetypes.ApiserverSource)
		err := cfg.Restore(bootstrapCheckpointPath, updatechannel)
		if err != nil {
			return nil, err
		}
	}

    // 添加API Server为第三种source
	if kubeDeps.KubeClient != nil {
		klog.Infof("Watching apiserver")
		if updatechannel == nil {
			updatechannel = cfg.Channel(kubetypes.ApiserverSource)
		}
		config.NewSourceApiserver(kubeDeps.KubeClient, nodeName, updatechannel)
	}
	return cfg, nil
}
```



## file

fiile模式主要是用于`kubelet` 对`staticPodPath` 这个目录的监控，只要在这个目录下有任何文件（点开头的文件除外）的增删改， 就将事件写入channel中。

```go
// 代码位置 pkg/kubelet/config/file.go
type sourceFile struct {
    // 静态Pod的地址
	path           string
    // 节点name
	nodeName       types.NodeName
    // 更新周期
	period         time.Duration
    // 缓存
	store          cache.Store
	fileKeyMapping map[string]string
    // 保存事件（Pod需要增加还是更新还是删除）的channel
	updates        chan<- interface{}
    // 事件的数据结构，包括fileName和podEventType
	watchEvents    chan *watchEvent
}
```

 我们可以看看file 的run 工作流程如下：

1. 新启动一个go协程

```go
// 代码位置 pkg/kubelet/config/file.go
func (s *sourceFile) run() {
	listTicker := time.NewTicker(s.period)

	go func() {
		// Read path immediately to speed up startup.
		if err := s.listConfig(); err != nil {
			klog.Errorf("Unable to read config path %q: %v", s.path, err)
		}
		for {
			select {
			case <-listTicker.C:
				if err := s.listConfig(); err != nil {
					klog.Errorf("Unable to read config path %q: %v", s.path, err)
				}
			case e := <-s.watchEvents:
				if err := s.consumeWatchEvent(e); err != nil {
					klog.Errorf("Unable to process watch event: %v", err)
				}
			}
		}
	}()

	s.startWatch()
}

```



### 生产者

工作流程如下：

开启一个go协程永久执行如下事项：

1. 如果正在执行watch 则直接返回
2. 执行doWatch()，如果返回错误不为空，则判断错误是否可以retry解决，如果是则执行retry
   1. 使用`fsnotify` 监控配置的静态Pod的目录的任何改动(点开头的文件忽略)。
      1. 如果是创建文件，设置`eventType` 为`podAdd` 
      2. 如果是改动文件或者改动权限，设置`eventType` 为`podModify` 
      3. 如果是重命名文件或者删除文件，设置`eventType` 为`podDelete`
   2. 将`event` 和`eventType`写入`watchEvents` channel 中

```go
// 代码位置 pkg/kubelet/config/file_linux.go
func (s *sourceFile) startWatch() {
	backOff := flowcontrol.NewBackOff(retryPeriod, maxRetryPeriod)
	backOffID := "watch"

	go wait.Forever(func() {
		if backOff.IsInBackOffSinceUpdate(backOffID, time.Now()) {
			return
		}

		if err := s.doWatch(); err != nil {
			klog.Errorf("Unable to read config path %q: %v", s.path, err)
			if _, retryable := err.(*retryableError); !retryable {
				backOff.Next(backOffID, time.Now())
			}
		}
	}, retryPeriod)
}

func (s *sourceFile) doWatch() error {
	_, err := os.Stat(s.path)
	if err != nil {
		if !os.IsNotExist(err) {
			return err
		}
		// Emit an update with an empty PodList to allow FileSource to be marked as seen
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{}, Op: kubetypes.SET, Source: kubetypes.FileSource}
		return &retryableError{"path does not exist, ignoring"}
	}

	w, err := fsnotify.NewWatcher()
	...
	err = w.Add(s.path)
	...

	for {
		select {
		case event := <-w.Events:
			if err = s.produceWatchEvent(&event); err != nil {
				return fmt.Errorf("error while processing inotify event (%+v): %v", event, err)
			}
		case err = <-w.Errors:
			return fmt.Errorf("error while watching %q: %v", s.path, err)
		}
	}
}

func (s *sourceFile) produceWatchEvent(e *fsnotify.Event) error {
	// Ignore file start with dots
	if strings.HasPrefix(filepath.Base(e.Name), ".") {
		klog.V(4).Infof("Ignored pod manifest: %s, because it starts with dots", e.Name)
		return nil
	}
	var eventType podEventType
	switch {
	case (e.Op & fsnotify.Create) > 0:
		eventType = podAdd
	case (e.Op & fsnotify.Write) > 0:
		eventType = podModify
	case (e.Op & fsnotify.Chmod) > 0:
		eventType = podModify
	case (e.Op & fsnotify.Remove) > 0:
		eventType = podDelete
	case (e.Op & fsnotify.Rename) > 0:
		eventType = podDelete
	default:
		// Ignore rest events
		return nil
	}

	s.watchEvents <- &watchEvent{e.Name, eventType}
	return nil
}
```

### 消费者

这里的消费者只是放入缓存，真正的消费者，仍然是背后的`syncLoop`。

消费者的工作流程如下：

`eventType`属于增加/更改的情况下：

​	1. 提取文件内容给Pod对象，将此对象增加到sourceFile(client-go)的Store缓存

`eventType`属于删除的情况下：

1.  获取文件名objKey，从缓存中获取这个Key的Pod对象，如果Pod不存在就直接返回，否则从缓存里面删除该对象

```go
func (s *sourceFile) consumeWatchEvent(e *watchEvent) error {
	switch e.eventType {
	case podAdd, podModify:
		pod, err := s.extractFromFile(e.fileName)
		if err != nil {
			return fmt.Errorf("can't process config file %q: %v", e.fileName, err)
		}
		return s.store.Add(pod)
	case podDelete:
		if objKey, keyExist := s.fileKeyMapping[e.fileName]; keyExist {
			pod, podExist, err := s.store.GetByKey(objKey)
			if err != nil {
				return err
			} else if !podExist {
				return fmt.Errorf("the pod with key %s doesn't exist in cache", objKey)
			} else {
				if err = s.store.Delete(pod); err != nil {
					return fmt.Errorf("failed to remove deleted pod from cache: %v", err)
				}
				delete(s.fileKeyMapping, e.fileName)
			}
		}
	}
	return nil
}
```



## http

http模式跟file模式类似，也是kubelet定期去扫描，不一样的是，http模式是定期到`manifest-url` 参数配置定期下载指定文件，并且转行成yaml格式的Pod文件，再去创建静态Pod。

（通过kubelet的启动参数`–manifest-url` `–manifest-url-header`来定义manifest url。 ）

举个例子，我们创建以下yaml 文件，并且保存在一个我们自己的web服务上，生成一个URL

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: http-web
  labels:
    role: myrole
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP
```

当我们的Kubelet启动的时候带上`KUBELET_ARGS="--manifest-url=<manifest-url>"` 那么该静态Pod就会被创建。



工作流程如下：

1. 使用Get尝试发送http 请求下载文件， 尝试检查response.StatusCode是否是200，以及返回的数据大小
2. 比对上次读取的数据有没有发生改变，如果没有改变，则直接返回
3. 走到这里，说明数据是发生了改变，尝试decode是否是一个Pod，检验数据是否是Pod格式，如果没有错误，则将Pod数据和Pod source: `kubetypes.HTTPSource` 写入到channel中。
4. 尝试decode 是否是多个Pod， 如果是，则轮询将数据写入pods中，然后写入到channel中。

```go
// 代码位置 pkg/kubelet/config/http.go

func (s *sourceURL) extractFromURL() error {
	req, err := http.NewRequest("GET", s.url, nil)
	req.Header = s.header
	resp, err := s.client.Do(req)

	defer resp.Body.Close()
	data, err := utilio.ReadAtMost(resp.Body, maxConfigLength)

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("%v: %v", s.url, resp.Status)
	}
	if len(data) == 0 {
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{}, Op: kubetypes.SET, Source: kubetypes.HTTPSource}
		return fmt.Errorf("zero-length data received from %v", s.url)
	}
	// 比对上次读取的数据有没有发生改变，如果没有改变，则直接返回
	if bytes.Equal(data, s.data) {
		return nil
	}
	s.data = data

	// 尝试decode是否是一个Pod，检验数据是否是Pod格式
	parsed, pod, singlePodErr := tryDecodeSinglePod(data, s.applyDefaults)
	if parsed {
		if singlePodErr != nil {
			// It parsed but could not be used.
			return singlePodErr
		}
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{pod}, Op: kubetypes.SET, Source: kubetypes.HTTPSource}
		return nil
	}

	// 尝试decode 是否是多个Pod， 如果是，则轮询将数据写入pods中，然后写入到channel中
	parsed, podList, multiPodErr := tryDecodePodList(data, s.applyDefaults)
	if parsed {
		if multiPodErr != nil {
			// It parsed but could not be used.
			return multiPodErr
		}
		pods := make([]*v1.Pod, 0)
		for i := range podList.Items {
			pods = append(pods, &podList.Items[i])
		}
		s.updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.HTTPSource}
		return nil
	}

	return fmt.Errorf("%v: received '%v', but couldn't parse as "+
		"single (%v) or multiple pods (%v)",
		s.url, string(data), singlePodErr, multiPodErr)
}
```



## API Server

这是我们最常用的Pod创建方法，也就是用户通过`kubectl apply -f `的方法将数据传给API Server， 经过调度器选出节点后，节点监听到是本机则在本机启动该Pod。这种方法创建的Pod不是静态Pod。



# 和其他模块的关系

然后podMgmt调用secretManger和configmapManager

然后以下三个都调用 Pod Manager

status Manager																				

volume Manager             

runtime Manager

# why need podManager

Kubelet 是从三个来源发现Pod的更新:

1. file 
2. http
3. API Server

从非API Server创建出来的Pod都叫静态Pod, 例如我们常见使用`kubeadm` 启动集群的API Server, Scheduler都是使用静态Pod 被Kubelet启动的。 同样的， API Server是不感知静态Pod的存在的。

为了监控静态Pod, Kubelet会给每一个静态Pod通过API Server去创建一个Mirror Pod。

一个Mirror Pod会跟它对应的static pod有相同pod name跟namespace，Mirror  pod的状态总是反映静态pod的实际状态。当static pod被删除时，相关的Mirror  pod也将被删除。