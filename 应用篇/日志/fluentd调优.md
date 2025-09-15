目前我们使用的是banzai logging-operator
# fluent-bit资源优化
fluent-bit是节点级采集，我们以daemonset方式部署

```yaml
resources:
  requests:
    cpu: 100m
```
当日志量每秒大于5000条，需要翻倍资源


fluent-bit 需要`enableUpstream: true`
这个很有意思，默认是不启动enableUpstream, fluent-bit连接用户配置fluentd的service的地址，他是通过解析获取fluentd pod的ip随机选择一个进行通信
当启动enableUpstream之后，fluent-bit会启动内置的上游管理功能，会自动发现所有的fluentd副本维护一个动态的fluentd节点列表
fluent-bit是怎么做到这个的呢？他是通过svc dns解析或者api 监听获取endpoints列表，然后定期健康检查可用节点通过负载均衡策略分配日志流量


# fluentd资源优化
使用sts部署
关键性能优化：
```yaml
bufferStorageVolume
```

# kafka 的优化
required_acks设置0
compression_codec: gzip
buffer:
  flush_thread_count
  flush_mode
