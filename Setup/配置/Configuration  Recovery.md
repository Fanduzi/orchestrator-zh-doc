# Configuration: Recovery
# [Configuration: recovery](https://github.com/openark/orchestrator/blob/master/docs/configuration-recovery.md)
`orchestrator`将对您的拓扑结构进行故障恢复. 您将指示`orchestrator`对哪些集群进行自动恢复, 哪些集群需要人工来恢复. 您将为`orchestrator`配置钩子以移动VIP, 更新服务发现等.

恢复依赖于检测，在[Configuration: Failure detection](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/配置/Configuration%20%20Failure%20detection.md)中讨论过.

关于恢复的所有信息, 请参考 [Topology recovery](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Topology%20recovery.md)

还要考虑到, 你的MySQL拓扑结构本身需要遵循一些规则, 参考[MySQL Configuration](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Configuration%20%20Recovery.md#mysql-configuration)

```yaml
{
  "RecoveryPeriodBlockSeconds": 3600,
  "RecoveryIgnoreHostnameFilters": [],
  "RecoverMasterClusterFilters": [
    "thiscluster",
    "thatcluster"
  ],
  "RecoverIntermediateMasterClusterFilters": [
    "*"
  ],
}
```
上述配置中:

* `orchestrator`将自动恢复所有集群intermediate master故障
* `orchestrator`将自动恢复两个指定集群的主站故障(即`RecoverMasterClusterFilters`中指定的`thiscluster`和`thatcluster`); 其他集群的主库将不会自动恢复. 可以人工触发恢复操作.
* 一旦集群经历了恢复, 协调器将在3600秒(1小时)之后阻止自动恢复. This is an anti-flapping mechanism.
>  什么是anti-flapping?. 可以看[这篇文章](https://medium.com/@jonfinerty/flapping-and-anti-flapping-dcba5ba92a05)间接了解一下

请再次注意, 自动恢复是可选的.

>  Note, again, that automated recovery is *opt in*.

### Promotion actions
不同的环境需要对recovery/promotion采取不同的行动

```yaml
{
  "ApplyMySQLPromotionAfterMasterFailover": true,
  "PreventCrossDataCenterMasterFailover": false,
  "PreventCrossRegionMasterFailover": false,
  "FailMasterPromotionOnLagMinutes": 0,
  "FailMasterPromotionIfSQLThreadNotUpToDate": true,
  "DelayMasterPromotionIfSQLThreadNotUpToDate": false,
  "MasterFailoverLostInstancesDowntimeMinutes": 10,
  "DetachLostReplicasAfterMasterFailover": true,
  "MasterFailoverDetachReplicaMasterHost": false,
  "MasterFailoverLostInstancesDowntimeMinutes": 0, 官方文档写错了
  "PostponeReplicaRecoveryOnLagMinutes": 0,
}
```
* `ApplyMySQLPromotionAfterMasterFailover` : 当为`true` 时, `orchestrator` 将在选举出的新主库上执行`reset slave all` 和`set read_only=0` . 默认: `true` . 当该参数为`true` 时, 将覆盖`MasterFailoverDetachSlaveMasterHost` .
* `PreventCrossDataCenterMasterFailover` : 默认`false` . 当为`true` 时, `orchestrator`将只用与故障集群主库位于同一DC的从库替换故障的主库. 它将尽最大努力从同一DC中找到一个替代者, 如果找不到, 将中止（失败）故障转移. 另请参阅`DetectDataCenterQuery`和`DataCenterPattern`配置变量.
* `PreventCrossRegionMasterFailover` : 默认`false` . 当为`true` 时, `orchestrator`将只用与故障集群主库位于同一region的从库替换故障的主库. 它将尽最大努力找到同一region的替代者, 如果找不到, 将中止（失败）故障转移. 另请参阅`DetectRegionQuery`和`RegionPattern`配置变量.
* `FailMasterPromotionOnLagMinutes` : 默认`0` (not failing promotion). 如果候选replica落后太多, 该参数可以用于将选举置为失败状态. 例如: 由于种种原因从库(候选主库)故障了5个小时(数据存在5小时延迟), 随后, 主库出现故障. 这种情况下, 我们可能希望阻止failover, 以便恢复那5个小时的复制延迟. 要使用这个参数, 你必须设置`ReplicationLagQuery` 并使用类似于`pt-heatbeat` 的心跳机制. 因为当复制故障时, `SHOW SLAVE STATUS` 中的`Seconds_behind_master` 不会显示延迟(会显示为Null).
* `FailMasterPromotionIfSQLThreadNotUpToDate` : 如果在故障发生时, 所有的从库都是滞后的, 即使是拥有最新数据的、被选举为新主库的候选节点也可能有未应用的中继日志. 在这样的节点上执行`reset slave all`会丢失中继日志数据.
* `DelayMasterPromotionIfSQLThreadNotUpToDate` : 如果在故障发生时, 所有的从库都是滞后的, 即使是拥有最新数据的、被选举为新主库的候选节点也可能有未应用的中继日志. 当该参数为`true` 时, `orchestrator` 将等待SQL thread应用完所有relay log, 然后再将候选从库提升为新主库. `FailMasterPromotionIfSQLThreadNotUpToDate`和`DelayMasterPromotionIfSQLThreadNotUpToDate`是相互排斥的.
* `DetachLostReplicasAfterMasterFailover` : 一些从库在恢复过程中可能会丢失. 如果该参数为`true`, `orchestrator`将通过`detach-replica`命令强行中断它们的复制, 以确保没有人认为它们是正常的.
>  some replicas may get lost during recovery. When `true`, `orchestrator` will forcibly break their replication via `detach-replica` command to make sure no one assumes they're at all functional.
* `MasterFailoverDetachReplicaMasterHost` : 当该参数为`true` 时, `orchestrator`将对被选举为新主库的节点执行`detach-replica-master-host`（这确保了即使旧主库"复活了", 新主库也不会试图从旧主库复制数据）. 默认值: `false`. 如果`ApplyMySQLPromotionAfterMasterFailover`为真，这个参数将失去意义. `MasterFailoverDetachSlaveMasterHost`是它的一个别名.
* `MasterFailoverLostInstancesDowntimeMinutes` :  number of minutes to downtime any server that was lost after a master failover (including failed master & lost replicas). Set to 0 to disable. Default: 0.
>  个人理解为一个"置为下线"的时间. 一个节点发生或failover后, 不应该立即再次进行恢复. MHA也有类似机制, 默认8小时内不能再次failover
* `PostponeReplicaRecoveryOnLagMinutes` : 在崩溃恢复时, 延迟超过给定分钟的副本只会在恢复过程的后期恢复, 在 master/intermediate master 被选择并执行进程之后.  值 0 禁用此功能.  默认值: 0.  `PostponeSlaveRecoveryOnLagMinutes` 是它的别名.

### Hooks
在恢复过程中可以使用以下钩子:

* `PreGracefulTakeoverProcesses` : 在计划中的、优雅的主库接管时, 在主库进入只读状态之前立即执行.
* `PreFailoverProcesses` : 在`orchestrator`采取恢复行动之前立即执行. 任何这些进程的失败（非零退出代码）都会中止恢复. 提示: 这让你有机会根据系统的一些内部状态中止恢复.
* `PostMasterFailoverProcesses` : 在主库恢复成功后执行.
* `PostIntermediateMasterFailoverProcesses` : executed at the end of a successful intermediate master or replication group member with replicas recovery. (属实没看懂这个, 看沃趣文章翻译是: 在intermediate master恢复成功后执行).
* `PostFailoverProcesses` : 在任何成功恢复结束时执行(包括并添加到上述两个 including and adding to the above two).
* `PostUnsuccessfulFailoverProcesses` : 在任何不成功的恢复结束时执行.
* `PostGracefulTakeoverProcesses` : 在计划中的、优雅的主库接管时, 在就主库作为新主库从库后执行.

任何以`"&"`结尾的进程命令都将被异步执行，并且此类进程的失败将被忽略

以上都是`orchestrator`按定义顺序依次执行的命令列表

一个简单的实现可能看起来像:

```yaml
{
  "PreGracefulTakeoverProcesses": [
    "echo 'Planned takeover about to take place on {failureCluster}. Master will switch to read_only' >> /tmp/recovery.log"
  ],
  "PreFailoverProcesses": [
    "echo 'Will recover from {failureType} on {failureCluster}' >> /tmp/recovery.log"
  ],
  "PostFailoverProcesses": [
    "echo '(for all types) Recovered from {failureType} on {failureCluster}. Failed: {failedHost}:{failedPort}; Successor: {successorHost}:{successorPort}' >> /tmp/recovery.log"
  ],
  "PostUnsuccessfulFailoverProcesses": [],
  "PostMasterFailoverProcesses": [
    "echo 'Recovered from {failureType} on {failureCluster}. Failed: {failedHost}:     {failedPort}; Promoted: {successorHost}:{successorPort}' >> /tmp/recovery.log"
  ],
  "PostIntermediateMasterFailoverProcesses": [],
  "PostGracefulTakeoverProcesses": [
    "echo 'Planned takeover complete' >> /tmp/recovery.log"
  ],
}
```
#### Hooks arguments and environment
`orchestrator` 为所有钩子提供了与 failure/recovery相关的信息, 例如故障实例的身份、提升实例的身份、受影响的副本、故障类型、集群名称等.

这些信息以两种方式独立传递, 您可以选择使用一种或两种方式:

1. 环境变量：`orchestrator` 将设置环境以下变量, 您的钩子可以从环境变量中获取这些信息(`orchestrator` will set the following, which can be retrieved by your hooks):

* `ORC_FAILURE_TYPE`
* `ORC_INSTANCE_TYPE` ("master", "co-master", "intermediate-master")
* `ORC_IS_MASTER` (true/false)
* `ORC_IS_CO_MASTER` (true/false)
* `ORC_FAILURE_DESCRIPTION`
* `ORC_FAILED_HOST`
* `ORC_FAILED_PORT`
* `ORC_FAILURE_CLUSTER`
* `ORC_FAILURE_CLUSTER_ALIAS`
* `ORC_FAILURE_CLUSTER_DOMAIN`
* `ORC_COUNT_REPLICAS`
* `ORC_IS_DOWNTIMED`
* `ORC_AUTO_MASTER_RECOVERY`
* `ORC_AUTO_INTERMEDIATE_MASTER_RECOVERY`
* `ORC_ORCHESTRATOR_HOST`
* `ORC_IS_SUCCESSFUL`
* `ORC_LOST_REPLICAS`
* `ORC_REPLICA_HOSTS`
* `ORC_COMMAND` (`"force-master-failover"`, `"force-master-takeover"`, `"graceful-master-takeover"` if applicable)
并且, 如果恢复成功, `orchestrator`还将设置以下环境变量:

* `ORC_SUCCESSOR_HOST`
* `ORC_SUCCESSOR_PORT`
* `ORC_SUCCESSOR_BINLOG_COORDINATES`
* `ORC_SUCCESSOR_ALIAS`



2. 命令行文本替换. `orchestrator`在你的`*Proccesses`命令中替换了以下神奇的标记.

* `{failureType}`
* `{instanceType}` ("master", "co-master", "intermediate-master")
* `{isMaster}` (true/false)
* `{isCoMaster}` (true/false)
* `{failureDescription}`
* `{failedHost}`
* `{failedPort}`
* `{failureCluster}`
* `{failureClusterAlias}`
* `{failureClusterDomain}`
* `{countReplicas}` (replaces `{countSlaves}`)
* `{isDowntimed}`
* `{autoMasterRecovery}`
* `{autoIntermediateMasterRecovery}`
* `{orchestratorHost}`
* `{lostReplicas}` (replaces `{lostSlaves}`)
* `{countLostReplicas}`
* `{replicaHosts}` (replaces `{slaveHosts}`)
* `{isSuccessful}`
* `{command}` (`"force-master-failover"`, `"force-master-takeover"`, `"graceful-master-takeover"` if applicable)
并且, 如果恢复成功, `orchestrator`还将提供以下变量:

* `{successorHost}`
* `{successorPort}`
* `{successorBinlogCoordinates}`
* `{successorAlias}`


### MySQL Configuration
您的 MySQL 拓扑必须满足一些要求才能支持故障转移.  这些要求在很大程度上取决于您使用的拓扑/配置类型.

* Oracle/Percona with GTID: promotable server必须启用`log_bin`和`log_slave_updates`. 复制体必须使用 `AUTO_POSITION=1`(通过 CHANGE MASTER TO MASTER\_AUTO\_POSITION=1).
* MariaDB GTID: promotable server必须启用`log_bin`和`log_slave_updates`.
* [Pseudo GTID](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Various/Pseudo%20GTID.md): promotable server必须启用`log_bin`和`log_slave_updates`. 如果使用`5.7/8.0` 并行复制(parallel replication), 请设置`slave_preserve_commit_order=1`
* BinlogServers: promotable servers must have `log_bin` enabled.

还可以考虑通过阅读[MySQL configuration](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Configuration%20%20Failure%20detection.md#mysql-configuration)优化故障检测的一些配置.
