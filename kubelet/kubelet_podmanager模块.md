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

静态Pod是不能被API Server来控制的， 为了让用户/Kubelet通过API Server来管理静态Pod, Kubelet会创建一个叫Mirror Pod，Mirror Pod其实跟Static Pod是一一对应的。

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



# 实例化

```go
// 代码位置 pkg/kubelet/kubelet.go
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,...){
    ...
    // podManager is also responsible for keeping secretManager and configMapManager contents up-to-date.
	mirrorPodClient := kubepod.NewBasicMirrorClient(klet.kubeClient, string(nodeName), nodeLister)
	klet.podManager = kubepod.NewBasicPodManager(mirrorPodClient, secretManager, configMapManager, checkpointManager)
}

```



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



# 调用关系



# Kubelet更新Pod

## file

## http

## API Server

到时候记得加个图

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



# why need podManager

Kubelet 是从三个来源发现Pod的更新:

1. file 
2. http
3. API Server

从非API Server创建出来的Pod都叫静态Pod, 例如我们常见使用`kubeadm` 启动集群的API Server, Scheduler都是使用静态Pod 被Kubelet启动的。 同样的， API Server是不感知静态Pod的存在的。

为了监控静态Pod, Kubelet会给每一个静态Pod通过API Server去创建一个Mirror Pod。

一个Mirror Pod会跟它对应的static pod有相同pod name跟namespace，Mirror  pod的状态总是反映静态pod的实际状态。当static pod被删除时，相关的Mirror  pod也将被删除。