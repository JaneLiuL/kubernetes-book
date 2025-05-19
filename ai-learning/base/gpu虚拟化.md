这是一个关于gpu硬件的知识篇
# 背景以及为什么虚拟化
gpu的划分（虚拟化）的需求其实是因为很多时候我们不能用满一整张显卡的所有资源，从资源利用率来说我们更希望gpu的虚拟化可以利用率更高，毕竟那么贵

# gpu并发机制
* cuda 流
* cuda多进程服务 MPS
* time-slicing时间分片
* 多实例 gpu MIG 
* vGPU 虚拟化

# mig之前的虚拟化 vGPU
NVIDIA 的vGPU：
vGPU是NVIDIA推出的一个支持在VM应用上的虚拟GPU软件，它支持在一块物理GPU创建许多小型的虚拟GPU，并让这些虚拟GPU提供给不同的用户使用。
对于物理硬件的虚拟化，主要分为两个：存储的虚拟化、运算的虚拟化。
基于时间片（time-sliced） 的vGPU本质上面还是让任务公用物理设备(引擎)，**通过改变虚拟GPU的时间片使用多少，来控制任务对整个物理设备上的资源的使用量**。 这么方式虽然能够满足一些应用场景，但是由于物理卡上的资源是公用的，**所有任务要轮着用，使得整卡切分后在算力/带宽/任务切换上面难做到好的QoS**。举个例子，比如两个任务要共用Video Decode设备时会涉及任务的来回切换，相比按照比例来使用的方式（假设有4个Video Decode，每个任务使用两个）的成本更高。 同时对于每个任务而言，并不是所有任务都能够使用一张完整GPU的全部资源，单个任务运算时会产生一定的资源浪费。
国内虚拟化比如华为云的ModelArts,阿里云的GPU服务基本也是参考vGPU的方式。

GPU的虚拟化基本都是围绕：数据能否安全，隔离是否充分，QoS能否保证来设计

# MIG
MIG其实是分块+组合，对物理卡上的物理资源进行切分，将分块后的资源重新整合，这样每个切分后的子GPU能够做到数据保护，故障隔离等

MIG的创建基本就是两个步骤：从物理GPU上建立GI（GPU Instance），然后从创建的GI上创建CI（Compute Instance）

# 有vGPU了，为什么还要MIG
MIG和vGPU目标都是对物理GPU进行虚拟化来满足应用需求。vGPU能够结合MIG使用，什么意思？ 就是在MIG的基础上可以进一步使用vGPU。MIG出现之前，vGPU底层的实现用的是时间分片模式（time-sliced），我们比较MIG和vGPU，一般是对比“时间分片模式vGPU”和“MIG版的vGPU”
是测试数据发现 MIG-vGPU 的表现优于 Time-sliced vGPU，对比了训练时间，训练的吞吐，推理的时间，推理的吞吐等


# cuda 流
Compute Unified Device Architecture (CUDA) 是由 NVIDIA 开发的并行计算平台和编程模型，用于 GPU 上的常规计算。
流是 GPU 上问题顺序执行的操作序列。CUDA 命令通常在默认流中按顺序执行，任务在前面的任务完成后才会启动。
跨不同流的异步处理操作允许并行执行任务。在一个流中发布的任务之前、期间或另一个任务签发到另一个流后运行。这允许 GPU 以任何规定的顺序同时运行多个任务，从而提高性能。


## 一些基本的命令
打开MIG功能： nvidia-smi -i 7 (查看7号Gpu状态可以看到MIG是不是disable)
启动MIG功能： nvidia-smi -i 7 -mig 1

## 常见问题
MIG的使能可能会遇到一些问题，会有pending的错误提示：
```
$ sudo nvidia-smi -i 0 -mig 1
Warning: MIG mode is in pending enable state for GPU 00000000:00:03.0:Not Supported
Reboot the system or try nvidia-smi --gpu-reset to make MIG mode effective on GPU 00000000:00:03.0
All done.
```
可能的原因：

GPU正在被某些程序使用；
docker容器挂载了这个GPU；
某些server（比如nvsm dcgm）占用了GPU；
解决方案：
```bash
# nvidia-smi --gpu-reset
# systemctl stop nvsm
# systemctl stop dcgm
# docker stop [container] # 停止运行的容器
```