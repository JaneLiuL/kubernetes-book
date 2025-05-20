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
# 架构设计和优化
架构设计需要从网络设计，存储架构设计等方面都要入手，这个具体看业务以及能拿到的硬件资源等，可以整体从以下几个方面考虑:
```
节点部署：	 多节点、地理分布、硬件高性能配置
存储：   	 NVMe SSD、优化文件系统、启用压缩
参数调优：	 快照频率、Wal存放、资源限制
网络：  	高速专用网络、TLS安全通信
监控：  	关键指标实时监控，及时调优
安全：  	TLS、权限控制、备份策略
```

# kube-proxy
必须使用ipvs 模式
kube-proxy 更改配置如下
```yaml
--ipvs-min-sync-period=1s #默认：最小刷新间隔1s
--ipvs-sync-period=5s  # 默认：5s周期性刷新
--cleanup=false
--ipvs-min-sync-period=0s # 发生事件实时刷新
--ipvs-sync-period=30s # 30s周期性刷新, 刷新一次节点 iptables 规则
--cleanup=true (清理 iptables 和 ipvs 规则并退出)
```
对于数据面高并发场景，服务服务配置
```yaml
--ipvs-scheduler=lc #最小连接，默认是rr round-robi
```

## conntracker设置
无论是iptable 或者 ipvs 底层都会走conntracker
```bash
$ sysctl -w net.netfilter.nf_conntrack_max=2310720
```
kube-proxy 启动的时候会重新设置, 因此配置参数可以按node CPU 个数进行调整
```bash
--conntrack-max-per-core=144420  # 2310720/cpu个数。 2310720/16核 = 144420
--conntrack-min=2310720 # 设置为nf_conntrack_max
```

# kubelet
配置每个node的pod的上限
```yaml
--max-pods=500
--node-status-update-frequency=3s
```
node lease信息上报调优 (update /etc/kubernetes/kubelet-config.yaml)
```yaml
  --node-status-update-frequency=3s #Node状态上报周期。默认10s --node-status-update-frequency - Specifies how often kubelet posts its node status to the API server. 可以参考 https://kubernetes.io/docs/concepts/architecture/nodes/
  --kube-api-qps=50 #node lease信息上报qps
  --kube-api-burst=100 #node lease信息上报并发
  --event-qps=100 #pod event信息上报 qps
  --event-burst=100 #pod event信息上报并发
```
集群中Node数据越少， kubelet的event/api qps与burst数越大，单机器上的产生的event数和apiserver交互的请求数就越大；
集群数据Node数据越大， apiserver和etcd的个数也需要相应按比例增加
**初略计算方式: qps值 = apiserver的qps数 / node个数**

## kubelet 测试
测试 kubelet部署Pod速度 和 kubelet 缩容后 event上报速度 两个关键指标
kubelet部署Pod速度
亲和到单机，pod 1个数 扩容到 200个， 用时 78,123s 下降到了 24.769s
```bash
$ time kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 of 200 updated replicas are available...
Waiting for deployment "nginx-deployment" rollout to finish: 2 of 200 updated replicas are available...
Waiting for deployment "nginx-deployment" rollout to finish: 3 of 200 updated replicas are available...
Waiting for deployment "nginx-deployment" rollout to finish: 4 of 200 updated replicas are available...
...
Waiting for deployment "nginx-deployment" rollout to finish: 197 of 200 updated replicas are available...
Waiting for deployment "nginx-deployment" rollout to finish: 198 of 200 updated replicas are available...
Waiting for deployment "nginx-deployment" rollout to finish: 199 of 200 updated replicas are available...
deployment "nginx-deployment" successfully rolled out

real    0m24.769s
user    0m0.075s
sys     0m0.012s
```
kubelet 缩容 event上报速度
亲和到单机，pod 200个数 缩容到 1个， 释放 199 个pod 时间， 从 32s 下降到 9.05s
```bash
$  watch time kubectl get rs  
Every 2.0s: time kubectl get rs                                                                                       Thu Sep 15 15:16:33 2022

NAME                         DESIRED   CURRENT   READY   AGE
nginx-deployment-b56784d9b   1         1         1       28h

real    0m9.058s
user    0m0.058s
sys     0m0.015s
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

## etcd性能测试
kubemark 来模拟k8s计算节点，我们mock 1000个Node, 部署5K个Pod
写入测试, 写入etcd 数据1.5个G
```bash
// leader
$ benchmark --endpoints="http://10.179.0.13:2379" --target-leader --conns=1 --clients=1 put --key-size=8 --sequential-keys --total=10000 --val-size=256

$ benchmark --endpoints="http://10.179.0.13:2379" --target-leader --conns=100 --clients=1000 put --key-size=8 --sequential-keys --total=100000 --val-size=256

// 所有 members
$ benchmark --endpoints="http://10.179.0.13:2379,http://10.179.0.2:2379,http://10.179.0.6:2379" --target-leader --conns=1 --clients=1 put --key-size=8 --sequential-keys --total=10000 --val-size=256

$ benchmark --endpoints=""http://10.179.0.13:2379,http://10.179.0.2:2379,http://10.179.0.6:2379"  --target-leader --conns=100 --clients=1000 put --key-size=8 --sequential-keys --total=100000 --val-size=256
```
读取测试
```bash
$ benchmark --endpoints="http://10.179.0.13:2379,http://10.179.0.2:2379,http://10.179.0.6:2379"  --conns=1 --clients=1  range foo --consistency=l --total=10000

$ benchmark --endpoints="http://10.179.0.13:2379,http://10.179.0.2:2379,http://10.179.0.6:2379"  --conns=1 --clients=1  range foo --consistency=s --total=10000

$ benchmark --endpoints="http://10.179.0.13:2379,http://10.179.0.2:2379,http://10.179.0.6:2379"  --conns=100 --clients=1000  range foo --consistency=l --total=100000

$ benchmark --endpoints="http://10.179.0.13:2379,http://10.179.0.2:2379,http://10.179.0.6:2379"  --conns=100 --clients=1000  range foo --consistency=s --total=100000
```


# apiserver优化




# 整体测试
使用45台机器 kubermark 在kubermark namespace下 mock 出来5000个 node
使用e2e工具 创建出来50个namespace， 1000个service， 40000个pod。
其中中1/4的namespace是大规模1000+ pod，1/4是中型规模400+ pod； 1/2的namespace是小型50+ pod； 按 namespace下的 configmap、serviceaccount、secrets等，按pod的个数等4：1比例随机创建