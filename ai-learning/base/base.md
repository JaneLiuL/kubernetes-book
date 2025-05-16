
# 背景
目前k8s是无法进行针对gpu/ mlu或者npu设备让多个任务共享的功能，并且我们也需要优化异构ai计算资源的利用率

目标要求如下，总结是必须要做到设备共享以及设备资源隔离
设备共享：
* 多个任务可以共享同一个设备，每个任务仅占用部分资源
* 设备内存控制，可以按MB之类来分配
* 设备规格制定
* 无太大侵入控制，不需要更改程序就能控制资源分配
设备资源隔离：
对一个pod或者job 来说,按以下设定我们应该要在容器里面看到3G设备内存才对
```yaml
      resources:
        limits:
          nvidia.com/gpu: 1 # 请求1个vGPU
          nvidia.com/gpumem: 3000 # 每个vGPU包含3000m设备内存
```
kubectl exec -it xxx -- /bin/sh
nvidia-smi 即可查看memory-used

# 在每个 GPU 节点中配置 nvidia 容器运行时
需要先安装 nidia 驱动和 nvidia-container-toolkit
配置运行时，这里只记录配置containerd
在使用 containerd 运行 Kubernetes 时，修改配置文件，通常位于 /etc/containerd/config.toml，以设置 nvidia-container-runtime 为默认的低级运行时：
```yaml
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "nvidia"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
            BinaryName = "/usr/bin/nvidia-container-runtime"
```
然后重启 containerd：
```bash
sudo systemctl daemon-reload && systemctl restart containerd
```
给节点打标签
通过添加标签标记您的 GPU 节点。没有此标签，节点无法被我们的调度器管理。
注意使用不同比如说hami 或者volcano都有特定的标签，具体查询你所需要的管理k8s的异构ai 计算设备的工具
kubectl label nodes {nodeid} gpu=on
link: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html

#