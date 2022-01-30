# Orchestrator configuration in larger environments
# [Orchestrator configuration in larger environments](https://github.com/openark/orchestrator/blob/master/docs/configuration-large.md)
如果监控大量的服务器, 后端数据库会成为一个瓶颈. 下面的内容是指使用MySQL作为orchestrator的后端.

一些配置选项允许你控制吞吐量. 这些设置是:

* `BufferInstanceWrites`
* `InstanceWriteBufferSize`
* `InstanceFlushIntervalMilliseconds`

* `DiscoveryMaxConcurrency`
使用`DiscoveryMaxConcurrency`限制orchestrator的并发发现数量, 并确保后端服务器的`max_connections`设置足够高, 以允许 orchestrator 根据需要建立尽可能多的连接.
通过设置 `BufferInstanceWrites: True`, 当轮询完成后, 结果将被缓冲，直到`InstanceFlushIntervalMilliseconds`过后或`InstanceWriteBufferSize`缓冲写入完成.
缓冲写入是按照写入的时间排序的, 使用单个 `insert ... on duplicate key update ...` 调用. 如果同一主机出现两次, 则只有最后一次写入会写入数据库.
`InstanceFlushIntervalMilliseconds`应该远远低于`InstancePollSeconds`, 因为这个值太高意味着数据没有被写入orchestrator 后端数据库. 这可能会导致`not recently checked`问题. 另外, 不同的健康检查是针对后端数据库状态运行的, 所以如果更新不够频繁, 可能会导致orchestrator无法正确检测到不同的故障情况.

对于较大的Orchestrator环境, 建议的初始值可能是:

```yaml
  ...
  "BufferInstanceWrites": true,
  "InstanceWriteBufferSize": 1000, 
  "InstanceFlushIntervalMilliseconds": 50,
  "DiscoveryMaxConcurrency": 1000,
  ...
```
