# overview
在 Kubernetes 中，CNI Bandwidth 插件是一个用于限制 Pod 网络带宽的官方 CNI 插件，它通过 Linux Traffic Control（tc）工具实现对 Pod 进出流量的速率限制，帮助避免单个 Pod 占用过多网络资源，保障集群网络的稳定性。
核心功能与解决的问题
CNI Bandwidth 插件主要解决以下问题：
***Pod 网络资源争抢：防止单个 Pod 无限制占用带宽，导致其他 Pod 网络拥塞。***
***QoS 保障：为不同优先级的 Pod（如生产环境关键服务）分配不同带宽配额。***
***流量控制精细化：支持对 Pod 的入站（ingress）和出站（egress）流量分别设置速率限制。***

# 工作原理
依赖 Linux tc 工具：通过 tc 命令在 Pod 对应的网络接口（如 eth0）上创建队列规则（qdisc），限制流量速率。
与 CNI 流程集成：在 Pod 创建时，CNI 插件根据配置为 Pod 网络接口应用带宽限制规则；Pod 销毁时自动清理规则。
支持的限制维度：
出站带宽（egress）：Pod 发送数据的最大速率。
入站带宽（ingress）：Pod 接收数据的最大速率（部分 CNI 实现可能不支持，需依赖底层网络插件）。

# 使用方式
CNI Bandwidth 插件通常不单独使用，而是作为主 CNI 插件（如 Calico、Flannel、Antrea）的补充，通过 Pod 注解（annotation）或 CNI 配置文件定义带宽限制。

1. 通过 Pod 注解配置（推荐）
在 Pod 的 metadata.annotations 中添加带宽限制参数：
yaml
apiVersion: v1
kind: Pod
metadata:
  name: bandwidth-demo
  annotations:
    kubernetes.io/ingress-bandwidth: "10M"  # 入站带宽限制（10Mbps）
    kubernetes.io/egress-bandwidth: "20M"   # 出站带宽限制（20Mbps）
spec:
  containers:
  - name: demo-container
    image: nginx
单位支持：K（Kbps）、M（Mbps）、G（Gbps）等。
2. 通过 CNI 配置文件全局默认限制
在 CNI 配置文件（如 /etc/cni/net.d/10-calico.conflist）中添加 bandwidth 插件配置，设置全局默认带宽：
```
{
  "cniVersion": "0.3.1",
  "name": "calico-network",
  "plugins": [
    {
      "type": "calico",  # 主 CNI 插件
      ...
    },
    {
      "type": "bandwidth",  # 带宽插件
      "capabilities": {"bandwidth": true},
      "defaults": {
        "ingressRate": 10000000,   # 默认入站速率（10Mbps，单位 bps）
        "ingressBurst": 1000000,   # 入站突发流量（1Mbps）
        "egressRate": 20000000,    # 默认出站速率（20Mbps）
        "egressBurst": 2000000     # 出站突发流量（2Mbps）
      }
    }
  ]
}```
antrea开启配置
```
{
  "cniVersion": "0.3.0",
  "name": "antrea",
  "plugins": [
    {
      "type": "antrea",  # 主 CNI 插件
      "ipam": {
        "type": "host-local"
      }
      ...
    },
    {
      "type": "bandwidth",  # 带宽插件
      "capabilities": {"bandwidth": true},
    }
  ]
}```

全局配置会被 Pod 注解的自定义配置覆盖（注解优先级更高）。
支持的 CNI 插件
CNI Bandwidth 插件需与主 CNI 插件配合使用，主流支持的 CNI 包括：
* Calico：通过 bandwidth 插件扩展支持带宽限制。
* Flannel：需额外部署 bandwidth 插件并配置。
* Weave Net：原生支持带宽限制，无需单独配置。
* Antrea：通过 trafficControl 功能实现类似带宽限制（非 CNI Bandwidth 插件，但效果一致）。

# 验证与监控
检查 Pod 带宽限制是否生效：
进入节点，找到 Pod 对应的网络接口（如 calixxxx 或 vethxxxx）：
bash
查看 Pod 对应的容器 ID
```
POD_UID=$(kubectl get pod bandwidth-demo -o jsonpath='{.metadata.uid}')
```
# 查找对应的网络接口
```
ip link | grep $POD_UID
```
使用 tc 命令查看规则：
```bash
tc qdisc show dev <interface-name>
```
若配置生效，会显示类似 tbf rate 20Mbit burst 2097152b latency 50ms 的规则（对应 20Mbps 出站限制）。
测试带宽：
在 Pod 内使用 iperf 或 speedtest-cli 测试实际带宽是否符合限制。

# 注意事项
入站限制的局限性：部分 CNI 插件（如 Flannel）对入站带宽限制支持不完善，可能仅能限制出站流量。
精度问题：基于 Linux tc 的限制存在一定误差，尤其在高并发场景下。
与网络策略的兼容性：需确保带宽限制与网络策略（如 Calico NetworkPolicy）不冲突。
节点依赖：要求节点安装 tc 工具（通常默认安装在主流 Linux 发行版中）。
# 总结
CNI Bandwidth 插件是 Kubernetes 中实现 Pod 网络带宽控制的轻量方案，通过与主 CNI 插件配合，可简单高效地限制 Pod 流量速率，避免网络资源滥用。实际使用中，推荐通过 Pod 注解 进行精细化配置，并结合具体 CNI 插件的特性验证限制效果。