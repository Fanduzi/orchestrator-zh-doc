# Configuration: Discovery, classifying servers
# [Configuration: discovery, classifying servers](https://github.com/openark/orchestrator/blob/master/docs/configuration-discovery-classifying.md)
配置`orchestrator` 如何获取集群名称, 数据中心等信息.

```yaml
{
  "ReplicationLagQuery": "select absolute_lag from meta.heartbeat_view",
  "DetectClusterAliasQuery": "select ifnull(max(cluster_name), '') as cluster_alias from meta.cluster where anchor=1",
  "DetectClusterDomainQuery": "select ifnull(max(cluster_domain), '') as cluster_domain from meta.cluster where anchor=1",
  "DataCenterPattern": "",
  "DetectDataCenterQuery": "select substring_index(substring_index(@@hostname, '-',3), '-', -1) as dc",
  "PhysicalEnvironmentPattern": "",
  "DetectSemiSyncEnforcedQuery": ""
}
```
### Replication lag
默认情况下, `orchestrator`使用`SHOW SLAVE STATUS` 来监控复制延迟(粒度为秒). 但这种延迟监控策略没有考虑链式复制情况. 许多人使用自定义的心跳机制, 如 `pt-heartbeat`. 这种方式可以获取绝对的延迟数值, 粒度可以达到亚秒级.

通过`ReplicationLagQuery` 参数, 你可以定义获取复制延迟的语句

### Cluster alias
在实际工作中, 集群名字往往是人为定义的, 例如:  "Main", "Analytics", "Shard031" 等. 然而, MySQL集群本身不知道自己的集群名称是什么.

`DetectClusterAliasQuery` 定一个查询, `orchestrator` 借此查询语句获取集群名称.

这个名字很重要. 你可能会用它来告诉`orchestrator`: "请自动恢复这个集群", 或者 "这个集群中的所有参与实例是什么".

为了能够获取集群名称, 一个技巧是在`meta`库中创建一个表:

```sql
CREATE TABLE IF NOT EXISTS cluster (
  anchor TINYINT NOT NULL,
  cluster_name VARCHAR(128) CHARSET ascii NOT NULL DEFAULT '',
  cluster_domain VARCHAR(128) CHARSET ascii NOT NULL DEFAULT '',
  PRIMARY KEY (anchor)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
...并按如下方式填充数据（例如，通过puppet/cron）:

```sql
mysql meta -e "INSERT INTO cluster (anchor, cluster_name, cluster_domain) VALUES (1, '${cluster_name}', '${cluster_domain}') ON DUPLICATE KEY UPDATE cluster_name=VALUES(cluster_name), cluster_domain=VALUES(cluster_domain)"
```
也许你们公司MySQL服务器的主机名包含集群名(比如都是单机单实例部署, 主机名是 cluster1-xxx). 那么你也可以通过`@@hostname` 取获取集群名称.

> 就是说, 集群名字, 怎么搞都行, 具体看你们公司的情况. 大不了就自己定义一个表, 让`orchestrator` 通过查询这个表获取集群名称

### Data center
`orchestrator`是数据中心感知的. 它不仅会在Web界面上很好地给它们着色; 而且在运行故障转移时, 它还会考虑到DC.

您将通过以下两种方法之一配置数据中心感知:

* `DataCenterPattern`: 用于fqdn的正则表达式. 例如. `"db-.*?-.*?[.](.*?)[.].myservice[.]com"`
* `DetectDataCenterQuery`: 一个返回数据中心名称的查询语句

### Cluster domain
较为次要的是, 主要是为了可见性, `DetectClusterDomainQuery`应该返回VIP或CNAME或其他信息来指代集群主库的地址.

### Semi-sync topology
在某些环境中, 不仅要控制半同步复制的数量, 而且要控制一个复制是半同步还是异步复制. `orchestrator`可以检测到不想要的半同步配置, 并切换半同步标志`rpl_semi_sync_slave_enabled`和`rpl_semi_sync_master_enabled`来纠正这种情况.

#### Semi-sync master (`rpl_semi_sync_master_enabled`)
如果新主库的`DetectSemiSyncEnforcedQuery`值大于0, 则`orchestrator`在主库故障切换期间(e.g. `DeadMaster)`会打开新主库的`rpl_semi_sync_master_enabled`参数.  如果主库标志以其他方式改变或错误设置, `orchestrator`不会触发任何恢复.

>  `orchestrator` enables the semi-sync master flag during a master failover (e.g. `DeadMaster`) if `DetectSemiSyncEnforcedQuery` returns a value > 0 for the new master. `orchestrator` does not trigger any recoveries if the master flag is otherwise changed or incorrectly set.

A semi-sync master可以进入两种故障情况. [LockedSemiSyncMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#lockedsemisyncmaster)和[MasterWithTooManySemiSyncReplicas](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#masterwithtoomanysemisyncreplicas), 在这两种情况的恢复过程中, `orchestrator`会关闭半同步的从库(打开了rpl\_semi\_sync\_slave\_enabled的从库)的`rpl_semi_sync_master_enabled` 参数.

#### Semi-sync replicas (`rpl_semi_sync_slave_enabled`)
`orchestrator`可以检测拓扑结构中是否存在不正确的半同步复制数量([LockedSemiSyncMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#lockedsemisyncmaster)和[MasterWithTooManySemiSyncReplicas](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#masterwithtoomanysemisyncreplicas)) 然后可以通过启用/禁用相应的半同步复制参数来纠正这种情况.

这种行为可以由以下选项控制:

* `DetectSemiSyncEnforcedQuery` : 返回半同步优先级的查询(零表示异步复制; 数字越大表示优先级越高).
* `EnforceExactSemiSyncReplicas` : 决定是否执行严格的半同步复制拓扑结构的标志. 如果启用, `LockedSemiSyncMaster`和`MasterWithTooManyReplicas`的恢复将在副本上启用和禁用半同步, 以根据优先级顺序完全匹配所需的拓扑结构.
* `RecoverLockedSemiSyncMaster`: 决定是否从`LockedSemiSyncMaster`情况下恢复的标志. 如果启用, `LockedSemiSyncMaster`的恢复将按照优先级顺序在副本上启用（但绝不会禁用）半同步, 以匹配主库的等待计数(rpl\_semi\_sync\_master\_wait\_for\_slave\_count). 如果`EnforceExactSemiSyncReplicas`被设置了, 这个选项就没有效果. 如果你想只处理半同步复制太少的情况, 而不是太多的话, 这个选项很有用.
* `ReasonableLockedSemiSyncMasterSeconds` : 触发`LockedSemiSyncMaster`条件的秒数; 如果没有设置, 则退回到`ReasonableReplicationLagSeconds` .

**Example 1**: 执行严格的半同步复制拓扑结构:

```yaml
"DetectSemiSyncEnforcedQuery": "select priority from meta.semi_sync where cluster_member = @@hostname",
  "EnforceExactSemiSyncReplicas": true
```
假设有以下拓扑结构,

```Plain Text
         ,- replica1 (priority = 10, rpl_semi_sync_slave_enabled = 1)
  master
         `- replica2 (priority = 20, rpl_semi_sync_slave_enabled = 1)
```
`orchestrator`会检测到[MasterWithTooManySemiSyncReplicas](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#masterwithtoomanysemisyncreplicas)的情况, 并在replica1上禁用半同步(低优先级).

**Example 2**: Enforcing a weak semi-sync replica toplogy, with

`rpl_semi_sync_master_wait_for_slave_count=1`:

```yaml
  "DetectSemiSyncEnforcedQuery": "select 2586",
  "DetectPromotionRuleQuery": "select promotion_rule from meta.promotion_rules where cluster_member = @@hostname",
  "RecoverLockedSemiSyncMaster": true
```
假设有以下拓扑结构,

```Plain Text
         ,- replica1 (priority = 2586, promotion rule = prefer, rpl_semi_sync_slave_enabled = 0)
  master
         `- replica2 (priority = 2586, promotion rule = neutral, rpl_semi_sync_slave_enabled = 0)
```
`orchestrator`会检测到一个[LockedSemiSyncMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#lockedsemisyncmaster)场景, 并在replica1上启用半同步(因为replica1的promotion rule是prefer)
