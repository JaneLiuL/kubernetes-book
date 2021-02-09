# Overview

PluginManager是Kubelet的插件注册服务，代码位置位于`pkg/kubelet/pluginmanager`目录。该目录包含一个程序pluginwatcher，用于Kubelet注册不同类型的节点级插件，比如设备插件或CSI插件。它通过监视kubelet.getPluginsDir()返回的目录下的inotify事件来发现插件。我们将把这个目录称为PluginsDir。

插件需要实现在`pkg/kubelet/apis/pluginregistration/v*/api.proto`中指定的gRPC注册服务。

# 插件发现

代码位置：`pkg/kubelet/pluginmanager/pluginwatcher/plugin_wtcher.go`

当他们把一个socket放在那个目录中时，或者在Kubelet启动时，如果套接字已经在那里，pluginwatcher服务将发现PluginDir中的插件。同样，如果socket文件从目录里面移除，那么pluginwatcher会从从kubelet移除该插件。

我们以CSI驱动来举例子，Kubelet需要通过Unix Socket跟外部的CSI驱动沟通，CSI volume启动会在每一个Node节点的`/var/lib/kubelet/plugins/[CSIDriverName]/csi.sock`创建一个socket（CSIDriverName就是该插件的名字）。我们的PluginManager会监控这个目录，一旦有新文件创建，那么就把该插件注册。

```go
func (w *Watcher) Start(stopCh <-chan struct{}) error {
	w.stopped = make(chan struct{})
    // init方法会创建目录，并且授权0755
	if err := w.init(); err != nil {
		return err
	}
    // 使用fsnotify不断监控目录
	fsWatcher, err := fsnotify.NewWatcher()
	...
	w.fsWatcher = fsWatcher

	// 遍历插件目录并添加fsWatcher文件系统观察者
	if err := w.traversePluginDir(w.path); err != nil {
		klog.Errorf("failed to traverse plugin socket path %q, err: %v", w.path, err)
	}

	go func(fsWatcher *fsnotify.Watcher) {
		defer close(w.stopped)
		for {
			select {
                // 监控到任何对该目录下有文件的添加都会触发handleCreateEvent
			case event := <-fsWatcher.Events:				
				if event.Op&fsnotify.Create == fsnotify.Create {
					err := w.handleCreateEvent(event)
					...
				} else if event.Op&fsnotify.Remove == fsnotify.Remove {
					w.handleDeleteEvent(event)
				}
				continue
			case err := <-fsWatcher.Errors:
				if err != nil {
					..
			case <-stopCh:
					...
			}
		}
	}(fsWatcher)
	return nil
}
```

使用`handleCreateEvent`来处理目录里面新添加的插件文件

这个socket文件名不应该以.开头。因为它会被忽略。

```go
func (w *Watcher) handleCreateEvent(event fsnotify.Event) error {
	fi, err := os.Stat(event.Name)
	..
    // 忽略.开头的任何文件
	if strings.HasPrefix(fi.Name(), ".") {
		klog.V(5).Infof("Ignoring file (starts with '.'): %s", fi.Name())
		return nil
	}

	if !fi.IsDir() {
		isSocket, err := util.IsUnixDomainSocket(util.NormalizePath(event.Name))
		...
		if !isSocket {
			klog.V(5).Infof("Ignoring non socket file %s", fi.Name())
			return nil
		}
		// 调用插件注册程序
		return w.handlePluginRegistration(event.Name)
	}

	return w.traversePluginDir(event.Name)
}

// 注册插件
func (w *Watcher) handlePluginRegistration(socketPath string) error {
	if runtime.GOOS == "windows" {
		socketPath = util.NormalizePath(socketPath)
	}
	err := w.desiredStateOfWorld.AddOrUpdatePlugin(socketPath)
	return nil
}
```



# 插件注册

对于任何发现的插件，kubelet将发布一个注册。GetInfo gRPC调用获取插件类型，名称，端点和支持的服务API版本。

如果以下任何步骤注册失败，在重试注册将从头开始:

1. 登记。针对套接字调用GetInfo。

2. 对内部插件类型处理程序调用Validate。

3. 针对内部插件类型处理程序调用Register。

4. 对套接字调用NotifyRegistrationStatus来指示注册结果。

5. 在插件初始化阶段，Kubelet将发出特定于插件的调用(例如:DevicePlugin::GetDevicePluginOptions)。

6. 一旦Kubelet确定它已经准备好使用你的插件，它将发出一个注册。NotifyRegistrationStatus gRPC调用。

如果插件从PluginDir中删除了它的套接字，这将被解释为插件注销。如果以下任何步骤在注销失败，在重试注销将从头开始:

1. 登记。针对套接字调用GetInfo。

2. DeRegisterPlugin是针对内部插件类型处理程序调用的。

我们

# 接口