# overview
这个文档主要是讲大规模k8s集群的一些性能问题
内容包括对etcd, kube-apiserver, kube-controller等性能以及稳定性方面
在看这篇文档之前最好预计一下你所需要的规模，pod数量，node数量等

我们发现大规模集群的一些以下的问题：
* etcd: 读写延迟，经常拒绝服务
    * read/write latency spike 
    * too many request ddos
    * unable to write when hitting storage limit 
* apiserver： 查询资源延迟非常高
    * get pod/nodes latency very slow
* controller： 延时非常高
    * controller cannot catch up
* schduler： 延迟高
    * scheduler throughput low
* kubelet
    * fail restart slowly

基于上面的问题，我们做了以下的调整：
# kube-proxy
必须使用ipvs 模式

# kubelet
配置每个node的pod的上限
```yaml
--max-pods=500
--node-status-update-frequency=3s
```
# kube-controller-manager
配置调优
```yaml
 --node-monitor-period=2s #检查 kubelet 的状态时间间隔
  --node-monitor-grace-period=20s #检查 notready node 时间间隔
  --pod-eviction-timeout=30s # pod 绑定失败后重新调度时间间隔
  --concurrent-deployment-syncs=50 # deployment并发度
  --concurrent-endpoint-syncs=50 # endpoint并发度
  --concurrent-job-syncs-50 # job并发度
  --concurrent-namespace-syncs=100 # namespace并发度
  --concurrent-replicaset-syncs=50 # replicaset并发度
  --concurrent-service-syncs=100 # service并发度
  --kube-api-qps=500 # 与apiserver 限流 500qps
  --kube-api-burst=100 # 与apiserver 限并发 100连接并发
```

# etcd 优化
## 时间参数
etcd 地产分布式一致性协议是参数来保证节点之间能在部分节点掉线情况下还能处理选举，第一个参数就是所谓的心跳间隔，默认是250毫秒
这里我们是使用kubespray安装的集群, heartbeat-interval就是所谓的心跳间隔，即主节点通知从节点它还是领导者的频率。实践数据表明，该参数应该设置成节点之间 RTT 的时间, 评估RTT 的最简单的方法是使用ping 操作
```yaml
--heartbeat-interval=250
```
## 快照
存储创建快照的代价是很高的，所以只用当参数累积到一定的数量时，Etcd 才会创建快照文件。 默认值是 10000 在超大集群中，Etcd 的内存使用和磁盘使用过高，那么应该尝试调低快照触发的阈值
```yaml
   --snapshot-count=5000 #数量达到 5000 时才会建立快照, 我们现在这个是10000
```
## 磁盘
etcd 的存储目录分为 snapshot 和 wal，他们写入的方式是不同的，snapshot 是内存直接 dump file，而 wal 是顺序追加写。因此可以将 snap 与 wal 进行分盘，放在两块 SSD 盘上，提高整体的 IO 效率，这种方式可以提升 etcd 20%左右的性能。
Linux 中 etcd 的磁盘优先级可以使用 ionice 配置：
```bash
ionice -c2 -n0 -p `pgrep etcd`
```
etcd 集群对磁盘I/O的延时非常敏感，而磁盘IO可以通过ionice命令来调整，IO调度策略分为三种：
* idle: 其他进程没有磁盘io, 才进行磁盘io
* best effort:数值越小优先级越高
* real time： 立即访问磁盘，无视其他进程io
* none

## CPU 优先级调整
`renice -n -20 -P $(pgrep etcd)`
其中 nice 值可以由用户指定，默认值为 0，root 用户的取值范围是[-20, 19]，普通用户的值取值范围是[0, 19]，数字越小，CPU 执行优先级越高。

## 数据规模和自动整理
etcd 的硬盘存储上限（默认是 2GB）,当 etcd 数据量超过默认 quota 值后便不再接受写请求，可以通过设置 --quota-backend-bytes 参数来增加存储大小, quota-backend-bytes 默认值 2GB，上限值为 8 GB, 3.4版本支持100GB
```yaml
  --quota-backend-bytes=8589934592  # 后端存储 8G
  --auto-compaction-mode=revision
  --auto-compaction-retention=1000  # 开启每5分钟就自动压缩，并保留lastet 1000个revision
```
## K8s events 拆到单独的 etcd 集群
apiserver是通过event驱动的服务，因此，将apiserver中不同类型的数据存储到不同类型的etcd集群中。 从 etcd 内部看，也就对应了不同的数据目录，通过将不同目录的数据路由到不同的后端 etcd 中，从而降低了单个 etcd 集群中存储的数据总量，提高了扩展性。
简单计算下： 如果有5000个Pod， 每次kubelet 上报信息是15Kb大小； 10000个Pod 事件变更信息，每次变更4Kb
etcd 接受Node信息： 15KB * (60s/3s) * 5000 = 150000Kb = 1465Mb/min
etcd 接受Pod event信息：10000 * 4Kb * 30% = 12Mb/min
这些更新将产生近 1.2GB/min 的 transaction logs（etcd 会记录变更历史)
拆解原则：

pod etcd
lease etcd
event etcd
其他etcd （node、job、deployment等等）
apiserver events拆解：
```yaml
  --etcd-servers="http://etcd1:2379,http://etcd2:2379,http://etcd3:2379" \
  --etcd-servers-overrides="/events#http://etcd4:2379,http://etcd5:2379,http://etcd6:2379"
  --etcd-servers-overrides="coordination.k8s.io/leases#http://etcd7:2379,http://etcd8:2379,http://etcd9:2379"
  --etcd-servers-overrides="/pods#http://etcd10:2379,http://etcd11:2379,http://etcd12:2379"
```

# apiserver优化
生产环境中，kubelet每10s汇报一次心跳，心跳的内容15Kb, 会容易引起etcd 中Node对象更新的时候产生每分钟将近300多M的transaction logs
并且apiserver会有很高的cpu消耗
# 网络设计和优化
