# Key-Value stores
# [Key-Value stores](https://github.com/openark/orchestrator/blob/master/docs/kv.md)
`orchestrator` 支持以下key-value存储:

* 一个基于关系表的内部存储

* [Consul](https://github.com/hashicorp/consul)
* [ZooKeeper](https://zookeeper.apache.org/)

更多信息请参阅[Configuration: Key-Value stores](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Configuration%20%20Key-Value%20stores.md)

### Key-Value usage
At this time Key-Value (aka KV) stores are used for:

* Master discoveries

### Master discoveries, key-values and failovers
其目的是, 诸如基于`Consul`或`ZookKeeper`的服务发现将能够提供master discovery和/或根据集群的master身份和变化采取行动.

最常见的场景是更新代理以将集群的写入流量定向到特定的主节点. 例如, 可以通过 `consul-template` 设置 `HAProxy`, 这样 `consul-template` 根据由 orchestrator 编写的键值存储填充a single-host master pool

orchestrator 在 master failover时更新所有 KV 存储.

#### Populating master entries
Clusters' master entries在以下情况下被填入:

* 遇到一个新的集群, 或遇到一个没有KV entry的master. 这个检查是自动和定期进行的
   * 定期检查首先咨询`orchestrator`的内部KV存储. 只有在内部存储没有master entries的情况下, 它才会尝试填充外部存储(`Consul`, `Zookeeper`). 由此可见, 定期检查将只注入一次外部KV.
* An actual failover: `orchestrator`用new master的身份覆盖现有条目
* 手工的entry填充请求:
   * `orchestrator-client -c submit-masters-to-kv-stores` 提交所有集群的master到KV, 或
   * `orchestrator-client -c submit-masters-to-kv-stores -alias mycluster` 提交`mycluster` 集群的master到KV
另见[orchestrator-client](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Use/orchestrator-client.md). 也可以使用orchestrator命令行.
或者你可以直接访问API:
   * `/api/submit-masters-to-kv-stores`
   * /api/submit-masters-to-kv-stores/:alias

实际的故障转移和手动请求都将覆盖任何现有的内部和外部KV entries.

### KV and orchestrator/raft
在[Orchestrator/raft, consensus cluster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%83%A8%E7%BD%B2/Orchestrator%20raft%2C%20consensus%20cluster.md) 部署模式中, 所有KV写入都要通过`raft` 协议. 因此, 一旦领导者确定需要对KV存储进行写入, 它就会向所有`raft`节点发布请求. 每个节点将根据自己的配置, 独立地应用写入.

#### Implications
举例来说, 假设你在一个3个数据中心的设置中运行`orchestrator/raft`, 每个DC一个节点. 另外, 我们假设你在这些DC上都部署了Consul. Consul的设置通常是在DC之间, 可能还有跨DC的异步复制.

在主站故障切换时, 每个`orchestrator`节点都会用new master的身份更新Consul.

如果你的Consul运行跨DC复制, 那么同一个KV更新有可能运行两次: 一次通过Consul复制, 一次通过本地`orchestrator`节点. 这两次更新是相同的、一致的, 因此可以安全运行

如果你的Consul没有相互复制, 那么`orchestrator`是使你的master discovery在你的Consul集群中一致的唯一手段. 你可以得到raft带来的所有好的特性: 如果一个DC被网络分区, 该DC中的`orchestrator`节点将不会收到KV更新事件, 并且在一段时间内，Consul 集群也不会收到. 然而, 一旦网络访问被恢复 `orchestrator`将赶上事件日志, 并将KV更新应用到本地Consul集群. The setup is eventual-consistent(最终一致).

在主站failover后不久, `orchestrator`就会生成一个raft快照. 这不是严格的要求的, 但却是一个有用的操作: 在`orchestrator`节点重新启动的情况下, 快照可以防止协调器重放KV写入. 这在 failover-and-failback的情况下特别有趣, 像consul这样的远程KV可能会得到同一个集群的两个更新. 快照可以缓解此类事件.

### Consul specific
Optionally, you may configure:

```yaml
  "ConsulCrossDataCenterDistribution": true,
```
...which can (and will) take place in addition to the flow illustrated above.

通过`ConsulCrossDataCenterDistribution`, `orchestrator`对Consul集群的扩展列表运行一个额外的、定期更新.

每分钟一次, `orchestrator`领导节点查询其配置的Consul服务器, 以获取[known datacenters](https://www.consul.io/api/catalog.html#list-datacenters)的列表. 然后, 它遍历这些数据中心集群, 并以主站的当前身份更新每一个集群.

如果一个人有更多的Consul数据中心, 而不是只有一个本地Consul-per-orchestrator-node, 那么就需要这个功能. 我们在上面说明了在`orchestrator/raft`部署模式中, 每个节点如何更新其本地Consul集群. 然而, 不属于任何`orchestrator`节点本地的Consul集群不受这种方法的影响. `ConsulCrossDataCenterDistribution`是包括所有这些其他DC的方式

#### Consul Transaction support
Atomic [Consul Transaction](https://www.consul.io/api-docs/txn) support is enabled by configuring:

```yaml
  "ConsulKVStoreProvider": "consul-txn",
```
*Note: this feature requires Consul version 0.7 or greater.*

This causes Orchestrator to use a [Consul Transaction](https://www.consul.io/api-docs/txn) when distributing one or more Consul KVs. KVs are read from the server in one transaction and any necessary updates are performed in a second transaction.

Orchestrator groups KV updates by key-prefix into groups of of 5 to 64 operations *(default 5)*. This grouping ensures updates to a single cluster *(5 x KVs)* happen atomically. Increasing the `ConsulMaxKVsPerTransaction` configuration setting from `5` *(default)* to a max of `64` *(Consul Transaction API limit)* allows more operations to be grouped into fewer transactions but more can fail at once.
