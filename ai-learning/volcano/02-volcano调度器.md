

# 插件
插件主要实现了3个函数：Name， OnSessionOpen， OnSessionClose
OnSessionOpen在会话开始时执行一些操作，并注册一些关于调度细节的函数。
OnSessionClose在会话结束时清理一些资源。
下面写一些volcano现在有的插件
首先是怎么配置volcano 调度器使用的插件
```yaml
# default configuration for scheduler
actions: "enqueue, allocate, backfill"
tiers:
- plugins:
  - name: priority
  - name: gang
  - name: conformance
- plugins:
  - name: overcommit
  - name: drf
  - name: predicates
  - name: proportion
  - name: nodeorder
  - name: binpack
```

等上述配置apply 之后，会按以下顺序来启动加载

(OpenSession) --> (enqueue) --> (allocate) --> (backfill) --> (CloseSession)


scheduler首先加载配置文件loadSchedulerConf，从配置文件读取actions以及plugins，这个第一次读取配置文件之后会不断的watch这个配置文件是否有modify/create来update,接下来启动scheduler初始化所需要的informer
定时默认每秒钟去执行每个schduler cycle: `runOnce`
`runOnce`主要是
所有的插件注册都是通过执行`OpenSession` 来被call，

`predicate`插件
`gang`插件