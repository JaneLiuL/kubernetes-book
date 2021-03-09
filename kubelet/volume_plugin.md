# Overview

前几天把`pkg/kubelet/volumemanager`模块看完，主要是围绕着kubelet自身如何mount/umount, attach/detach 卷，并且维护了一套卷期望状态和实际状态的缓存。 在该模块中多次调用`pkg/volume`的插件去执行mount/umount等操作。

这篇文档主要是围绕`pkg/volume`来讲解。

Commid id: 370e2f4b298e7276f560c131e24d8f91b88ae89f

# 带着问题出发

1. volume 定义了什么接口
2. volume plugin是什么时候被调用
3. volume plugin是什么时候被注册的, 如何管理和扩展这些插件



# 接口

Kubernetes卷插件扩展了Kubernetes卷接口，以支持block和/或file storage system。

`Volume` 的 `VolumePlugin` 接口定义了作为一个卷最基础的必须要实现的方法，如下所示。 配合卷插件的单元测试比较能理解这些必须要实现的方法。

```go
// 代码位置 pkg/volume/plugins.go
type VolumePlugin interface {
	Init(host VolumeHost) error
	GetPluginName() string
	GetVolumeName(spec *Spec) (string, error)
	CanSupport(spec *Spec) bool
	...
}
```



## 单元测试

volume plugin的单元测试非常容易理解一个卷的插件需要实现的最基础的方法

1. 定义插件名字
2. 实现`Init()`  `GetPluginName()`  `NewMounter()`  `GetVolumeName()`等方法
3. 测试`VolumePluginMgr` 跟踪一个已注册的插件
   1. 调用`VolumePluginMgr.InitPlugins`去初始化插件，包括设置插件的名字，插件的host地址等，把这些信息写入到`VolumePluginMgr`对象中
   2. 调用`VolumePluginMgr.FindPluginByName` 测试能否获取该testPluginName名字一样的插件
   3. 调用`VolumePluginMgr.FindPluginBySpec(nil)` 来测试`FindPluginBySpec`方法本身，当Volume.spec是nil的时候应该返回Error才对
   4. 然后我们实例化`volumeSpec := &Spec{}`， 再调用`VolumePluginMgr.FindPluginBySpec(nil)` 来测试`FindPluginBySpec`方法本身这个时候，error就不应该是空。

```go
// 代码位置 pkg/volume/plugins_test.go
const testPluginName = "kubernetes.io/testPlugin"
type testPlugins struct {
}
// 卷插件的Init会实现插件的初始化，某些插件如NFS会设置host
func (plugin *testPlugins) Init(host VolumeHost) error {
	return nil
}
// 一般会用来返回插件的name, 卷插件的name pattern需要是"example.com/volume" 格式，中间必须包含/
func (plugin *testPlugins) GetPluginName() string {
	return testPluginName
}
// 返回卷的名字或者ID来确定唯一的设备/目录，如NFS 卷插件会返回nfsserver/nfspath
func (plugin *testPlugins) GetVolumeName(spec *Spec) (string, error) {
	return "", nil
}
...
// 实例化volume.Mounter
func (plugin *testPlugins) NewMounter(spec *Spec, podRef *v1.Pod, opts VolumeOptions) (Mounter, error) {
	return nil, nil
}

func newTestPlugin() []VolumePlugin {
	return []VolumePlugin{&testPlugins{}}
}

func TestVolumePluginMgrFunc(t *testing.T) {
	vpm := VolumePluginMgr{}
	var prober DynamicPluginProber = nil // TODO (#51147) inject mock
	vpm.InitPlugins(newTestPlugin(), prober, nil)

	plug, err := vpm.FindPluginByName(testPluginName)
	if err != nil {
		t.Errorf("Can't find the plugin by name")
	}
	if plug.GetPluginName() != testPluginName {
		t.Errorf("Wrong name: %s", plug.GetPluginName())
	}

	plug, err = vpm.FindPluginBySpec(nil)
	if err == nil {
		t.Errorf("Should return error if volume spec is nil")
	}

	volumeSpec := &Spec{}
	plug, err = vpm.FindPluginBySpec(volumeSpec)
	if err != nil {
		t.Errorf("Should return test plugin if volume spec is not nil")
	}
}
```

卷是包含临时卷和持久卷，临时卷也就是像`emptyDir`这种跟随Pod的生命周期被创建和销毁的卷。而持久卷是独立于Pod的生命周期，例如AWS EBS或者NFS这种。

持久卷接口是在`VolumePlugin`的基础上扩展了`GetAccessModes`

```go
type PersistentVolumePlugin interface {
	VolumePlugin
	GetAccessModes() []v1.PersistentVolumeAccessMode
}
```

具体其他接口可以查看`pkg/volume/plugins.go` 。



# VolumePluginMgr

```go
// 代码位置 pkg/volume/plugins.go
type VolumePluginMgr struct {
	mutex         sync.Mutex
	plugins       map[string]VolumePlugin
	prober        DynamicPluginProber
	probedPlugins map[string]VolumePlugin
	Host          VolumeHost
}
```



## 实例化

`VolumePluginMgr` 是在实例化kubelet的时候被实例化的。

```go
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,...) (*Kubelet, error) {	
    ...
	klet.volumePluginMgr, err =
		NewInitializedVolumePluginMgr(klet, secretManager, configMapManager, tokenManager, kubeDeps.VolumePlugins, kubeDeps.DynamicPluginProber)
    ...
}
```



调用链关系如下

```
NewInitializedVolumePluginMgr()
	kvh.volumePluginMgr.InitPlugins()
```



接下来我们查看`VolumePluginMgr`在实例化之后被执行了`InitPlugins`。 我们来看看该方法的工作流程：



```go
// 代码位置 pkg/volume/plugins.go
func (pm *VolumePluginMgr) InitPlugins(plugins []VolumePlugin, prober DynamicPluginProber, host VolumeHost) error {
	pm.mutex.Lock()
	defer pm.mutex.Unlock()

	pm.Host = host

	if prober == nil {
		// Use a dummy prober to prevent nil deference.
		pm.prober = &dummyPluginProber{}
	} else {
		pm.prober = prober
	}
	if err := pm.prober.Init(); err != nil {
		// Prober init failure should not affect the initialization of other plugins.
		klog.Errorf("Error initializing dynamic plugin prober: %s", err)
		pm.prober = &dummyPluginProber{}
	}

	if pm.plugins == nil {
		pm.plugins = map[string]VolumePlugin{}
	}
	if pm.probedPlugins == nil {
		pm.probedPlugins = map[string]VolumePlugin{}
	}

	allErrs := []error{}
	for _, plugin := range plugins {
		name := plugin.GetPluginName()
		if errs := validation.IsQualifiedName(name); len(errs) != 0 {
			allErrs = append(allErrs, fmt.Errorf("volume plugin has invalid name: %q: %s", name, strings.Join(errs, ";")))
			continue
		}

		if _, found := pm.plugins[name]; found {
			allErrs = append(allErrs, fmt.Errorf("volume plugin %q was registered more than once", name))
			continue
		}
		err := plugin.Init(host)
		if err != nil {
			klog.Errorf("Failed to load volume plugin %s, error: %s", name, err.Error())
			allErrs = append(allErrs, err)
			continue
		}
		pm.plugins[name] = plugin
		klog.V(1).Infof("Loaded volume plugin %q", name)
	}
	return utilerrors.NewAggregate(allErrs)
}
```

