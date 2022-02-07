# 在生产环境中部署Orchestrator
# [Orchestrator deployment in production](https://github.com/openark/orchestrator/blob/master/docs/deployment.md)
`orchestrator`的部署是什么样子的？您需要在`puppet/chef`中设置什么？哪些服务需要运行, 在哪里运行？

## 部署服务和客户端
你首先要决定是在一个共享的后端数据库上运行`orchestrator` , 还是用raft协议.

> You will first decide whether you want to run `orchestrator` on a shared backend DB or with a `raft` setup

参见[Orchestrator高可用](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Deployment/Orchestrator高可用.md)，以及[orchestrator/raft vs. synchronous replication setup](Setup/部署/orchestrator%20raft%20vs.%20synchronous%20replication%20setup.md)的比较和讨论.

遵循这些部署指南:

* 在共享后端数据库上部署`orchestrator` : [shard backend模式部署](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Deployment/shard%20backend模式部署.md)
* 通过raft共识部署`orchestrator` : [raft模式部署](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Deployment/raft模式部署.md)

## Next steps
`orchestrator`在动态环境中工作得很好, 并适应inventory、配置和拓扑的变化. 它的动态特性表明, 环境也应该在动态特性中与它相互作用. 与硬编码的配置不同, `orchestrator`乐于接受动态提示和请求, 这些提示和请求可以更改其对拓扑结构的看法. 部署好`orchestrator`服务和客户端后, 请考虑执行以下操作以充分利用这一点.

### Discovering topologies 拓扑发现
`orchestrator`自动发现加入拓扑结构的实例. 如果一个新的副本加入了一个由`orchestrator`监控的现有集群, 那么当`orchestrator`下一次探测集群主库时就会发现它.

那么, `orchestrator` 是如何发现全新的拓扑结构的呢?

* 你可以要求`orchestrator`发现（探测）这样一个拓扑中的任何一个服务器, 然后从那里开始, 它将在整个拓扑中爬行.
> 译者注: 连接这个节点后, 通过processlist和show slave status, 顺藤摸瓜, 发现整个拓扑结构
* Or you may choose to just let `orchestrator` know about any single production server you have, routinely. Set up a cronjob on any production `MySQL` server to read:

```bash
0 0 * * * root "/usr/bin/perl -le 'sleep rand 600' && /usr/bin/orchestrator-client -c discover -i this.hostname.com"
```
在上述情况下, 每个主机每天让`orchestrator`了解自己一次; 新启动的主机在第二天午夜被发现. 引入睡眠是为了避免所有服务器在同一时间对`orchestrator`造成冲击.

以上使用的是[orchestrator-client](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Use/orchestrator-client.md), 但如果是使用[shard backend模式部署](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Deployment/shard%20backend模式部署.md), 你可以使用orchestrator cli.

### Adding promotion rules
在发生故障转移时, 有些server更适合提升为leader. 有些server则不是好选择. 例子:

* 硬件配置较差的server. 你不希望他被选为new leader.
* 在另一个数据中心的server. 你不希望他被选为new leader.
* 作为备份节点的server: 如会运行LVM快照备份, mysqldump, xtrabackup等等. 你不希望他被选为new leader.
* 一个配置较好的server, 是理想的候选人. 你希望他被选为new leader.
* 任何一个状态正常的server, 你没有特别的选择意见.

您将以下列方式向`orchestrator`协调者宣布您对某一特定服务器的偏爱:

```bash
orchestrator -c register-candidate -i ${::fqdn} --promotion-rule ${promotion_rule}
```
支持的promotion rules为:

* `prefer`
* `neutral`  中立
* `prefer_not`
* `must_not`

Promotion rules在一小时后失效.  That's the dynamic nature of `orchestrator`. 你需要设置一个cron作业来宣布服务器的promotion rule:

```bash
*/2 * * * * root "/usr/bin/perl -le 'sleep rand 10' && /usr/bin/orchestrator-client -c register-candidate -i this.hostname.com --promotion-rule prefer"
```
此设置来自生产环境. cron entries通过`puppet` 更新来设置新的`promotion_rule` . 一个server可能现在是`prefer`的, 但5分钟后就是`prefer_not`

> 这取决于你们公司自己的选主逻辑, 如, prefer的服务器要是是例行维护了, 那么就要更改`promotion_rule`

你可以整合你自己的服务发现方法、你自己的脚本, 以提供你最新的`promotion_rule`



### Downtiming
当一个server出现问题, 它:

* 将在web页面中的`problems` 下拉菜单中展示.

* May be considered for recovery (example: server is dead and all of it's replicas are now broken).

你可以主动`下线` 一个server:

* 它不会出现在`problems` 下拉菜单中

* It will not be considered for recovery.


Downtiming takes place via:

```bash
orchestrator-client -c begin-downtime -duration 30m -reason "testing" -owner myself
```
有些server可能经常被破坏; 例如, auto-restore servers; dev boxes; testing boxes. 对于这样的服务器, 您可能希望持续停机(*continuous* downtime). 实现这一点的一种方法是设置一个很大的`-duration 240000h` . 但是，如果有什么变化，你需要记住结束停机时间(`end-downtime`). Continuing the dynamic approach, consider:

```bash
*/2 * * * * root "/usr/bin/perl -le 'sleep rand 10' && /data/orchestrator/current/bin/orchestrator -c begin-downtime -i ${::fqdn} --duration=5m --owner=cron --reason=continuous_downtime"
```
每`2`分钟，停机`5`分钟; 这意味着, 当我们取消cronjob时, 停机时间将在`5`分钟内到期.

上面展示的是`orchestrator-client` 和`orchestrator` cli的用法. 为了完整起见, 下面展示如何通过直接调用API来完成相同的操作:

```bash
$ curl -s "http://my.orchestrator.service:80/api/begin-downtime/my.hostname/3306/wallace/experimenting+failover/45m"
```
`orchestrator-client`脚本正是运行这个API调用, 将其包装起来并对URL路径进行编码. 它还可以自动检测leader, 以防你不想通过代理运行.

### Pseudo-GTID
如果你没有使用GTID, 你会很高兴知道`orchestrator`可以利用Pseudo-GTID来实现与GTID类似的好处, 例如将两个不相关的服务器关联起来, 使一个从另一个复制. This implies master and intermediate master failovers.

更多信息请参阅[Pseudo GTID](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Various/Pseudo%20GTID.md).

`orchestrator` 可以为你注入 Pseudo-GTID条目. 你的集群将神奇地拥有类似GTID的超能力. 遵循[Automated Pseudo-GTID injection](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Configuration%20%20Discovery%2C%20Pseudo-GTID.md#automated-pseudo-gtid-injection)

### Populating meta data 填充元数据
`orchestrator` 会从server中提取一些元数据:

* 这个实例所属的集群的别名是什么?
* 这个服务器所属的数据中心是什么?
* 这个server开启半同步了吗?

这些细节是通过下列查询获取的:

* `DetectClusterAliasQuery`
* `DetectClusterDomainQuery`
* `DetectDataCenterQuery`
* `DetectSemiSyncEnforcedQuery`
或通过正则表达式作用于主机名:
* `DataCenterPattern`
* `PhysicalEnvironmentPattern`

查询可以通过将数据注入你的主表的元数据表来满足. 例如, 你可以:

```sql
CREATE TABLE IF NOT EXISTS cluster (
  anchor TINYINT NOT NULL,
  cluster_name VARCHAR(128) CHARSET ascii NOT NULL DEFAULT '',
  PRIMARY KEY (anchor)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
然后用类似"1, my\_cluster\_name"的数据填充该表. 并使用伴随类似下面的查询去获取元数据.

```sql
{
  "DetectClusterAliasQuery": "select cluster_name from meta.cluster where anchor=1"
}
```
请注意`orchestrator`不会创建这样的表, 也不会填充它们. 你需要创建表, 填充它们, 并让orchestrator知道如何查询数据.

### Tagging 标签
`orchestrator`支持对实例打标签, 以及通过标签搜索实例. See[Tags](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Operation/Tags.md)
