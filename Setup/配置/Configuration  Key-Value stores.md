# Configuration: Key-Value stores
# [Configuration: Key-Value stores](https://github.com/openark/orchestrator/blob/master/docs/configuration-kv.md)
`orchestrator` 支持以下key-value存储:

* 一个基于关系表的内部存储

* [Consul](https://github.com/hashicorp/consul)
* [ZooKeeper](https://zookeeper.apache.org/)
`orchestrator` 通过将集群的 master 存储在 KV 中来支持 master 发现.

```yaml
  "KVClusterMasterPrefix": "mysql/master",
  "ConsulAddress": "127.0.0.1:8500",
  "ZkAddress": "srv-a,srv-b:12181,srv-c",
  "ConsulCrossDataCenterDistribution": true,
```
`KVClusterMasterPrefix` is the prefix to use for master discovery entries. 例如, 您的集群别名是 `mycluster` 并且主库主机名是 `some.host-17.com` 那么您会看到一个条目, 其中:

* Key为`mysql/master/mycluster`
* Value为`some.host-17.com:3306`

注意: 在 ZooKeeper 上, 键将自动以 `/` 为前缀.

#### Breakdown entries
除了上述之外, `orchestrator` 还分解了主条目并添加了以下内容（通过上面的示例进行说明）:

* `mysql/master/mycluster/hostname`, value is `some.host-17.com`
* `mysql/master/mycluster/port`, value is `3306`
* `mysql/master/mycluster/ipv4`, value is `192.168.0.1`
* `mysql/master/mycluster/ipv6`, value is `<whatever>`

`/hostname`、`/port`、`/ipv4` 和 `/ipv6` 扩展会自动添加到任何master entry.

### Stores
如果指定, `ConsulAddress`表示一个可以使用Consul HTTP服务的地址. 如果没有指定, 则不会尝试访问Consul.

如果指定, `ZkAddress`表示要连接的一个或多个ZooKeeper服务器. 每个服务器的默认端口是2181. 以下都是等效的:

* srv-a,srv-b:12181,srv-c

* srv-a,srv-b:12181,srv-c:2181
* srv-a:2181,srv-b:12181,srv-c:2181

### Consul specific
关于Consul的具体设置, 见[Key-Value stores](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Failure%20detection%20%26%20recovery/Key-Value%20stores.md) .
