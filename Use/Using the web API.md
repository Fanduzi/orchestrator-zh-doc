# Using the web API
## [Using the web API](https://github.com/openark/orchestrator/blob/master/docs/using-the-web-api.md)
`orchestrator` 提供了一个精心设计的 Web API.

敏锐的网络开发者会注意到（通过Firebug或开发者工具）web interface是如何完全依赖JSON API请求的.

开发人员可以使用该API来实现自动化的目的.

### A very very brief look at a few API commands
举例来说:

* `/api/instance/:host/:port` : 读取并返回一个实例的详细信息(例如`/api/instance/mysql10/3306`).
* `/api/discover/:host/:port` : discover给定的实例(`orchestrator`服务将从那里获取它并递归扫描整个拓扑).
* `/api/relocate/:host/:port/:belowHost/:belowPort` : (试图)将一个实例移到另一个实例的下面, `orchestrator`选择最佳行动方案.
* `/api/relocate-replicas/:host/:port/:belowHost/:belowPort` : (试图)将一个实例的副本移到另一个实例的下面, 协调者选择最佳行动方案.
* `/api/recover/:host/:post` : 在给定的实例上启动恢复, 假设有东西需要恢复.
* `/api/force-master-failover/:mycluster` : 强制在给定的集群上立即进行failover.

### Full listing
完整的接口清单请见[api.go](https://github.com/openark/orchestrator/blob/master/go/http/api.go)(向下滚动到`RegisterRequests`)

你可能还想看看[orchestrator-client](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Use/orchestrator-client.md)([source code](https://github.com/openark/orchestrator/blob/master/resources/bin/orchestrator-client)), 看看command line interface是如何转化为API调用的. 或者, 直接使用[orchestrator-client](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Use/orchestrator-client.md)作为你的API客户端, 这就是它的作用.

### Instance JSON breakdown
许多 API 调用返回 *instance objects*实例对象, 描述单个 MySQL server. 此示例之后是字段细分

```yaml
{

    "Key": {
        "Hostname": "mysql.02.instance.com",
        "Port": 3306
    },
    "Uptime": 45,
    "ServerID": 101,
    "Version": "5.6.22-log",
    "ReadOnly": false,
    "Binlog_format": "ROW",
    "LogBinEnabled": true,
    "LogReplicationUpdatesEnabled": true,
    "SelfBinlogCoordinates": {
        "LogFile": "mysql-bin.015656",
        "LogPos": 15082,
        "Type": 0
    },
    "MasterKey": {
        "Hostname": "mysql.01.instance.com",
        "Port": 3306
    },
    "ReplicationSQLThreadRuning": true,
    "ReplicationIOThreadRuning": true,
    "HasReplicationFilters": false,
    "SupportsOracleGTID": true,
    "UsingOracleGTID": true,
    "UsingMariaDBGTID": false,
    "UsingPseudoGTID": false,
    "ReadBinlogCoordinates": {
        "LogFile": "mysql-bin.015993",
        "LogPos": 20146,
        "Type": 0
    },
    "ExecBinlogCoordinates": {
        "LogFile": "mysql-bin.015993",
        "LogPos": 20146,
        "Type": 0
    },
    "RelaylogCoordinates": {
        "LogFile": "mysql_sandbox21088-relay-bin.000051",
        "LogPos": 16769,
        "Type": 1
    },
    "LastSQLError": "",
    "LastIOError": "",
    "SecondsBehindMaster": {
        "Int64": 0,
        "Valid": true
    },
    "SQLDelay": 0,
    "ExecutedGtidSet": "230ea8ea-81e3-11e4-972a-e25ec4bd140a:1-49",
    "ReplicationLagSeconds": {
        "Int64": 0,
        "Valid": true
    },
    "Replicas": [ ],
    "ClusterName": "mysql.01.instance.com:3306",
    "DataCenter": "",
    "PhysicalEnvironment": "",
    "ReplicationDepth": 1,
    "IsCoMaster": false,
    "IsLastCheckValid": true,
    "IsUpToDate": true,
    "IsRecentlyChecked": true,
    "SecondsSinceLastSeen": {
        "Int64": 9,
        "Valid": true
    },
    "CountMySQLSnapshots": 0,
    "IsCandidate": false,
    "UnresolvedHostname": ""
}
```
实例的结构在不断发展, 文档可能更新不及时. 一些关键属性是:

* `Key`: 实例的唯一标识: 一个host和port的组合.
* `ServerID`:  MySQL `server_id` 参数值.
* `Version`: MySQL 版本.
* `ReadOnly`:  global `read_only` 值,  布尔值.
* `Binlog_format`:  global `binlog_format` 参数值.
* `LogBinEnabled`: 是否启用了binlog.
* `LogReplicationUpdatesEnabled`:  是否启用了 `log_slave_updates` .
* `SelfBinlogCoordinates`: 当前写到哪个binary log file 哪个 position  (和 `SHOW MASTER STATUS`显示的一样).
* `MasterKey`: 主库的hostname & port, 如果这个实例有主库的话.
* `ReplicationSQLThreadRuning`: 从`SHOW SLAVE STATUS`的`Slave_SQL_Running`直接映射.
* `ReplicationIOThreadRuning`: 从`SHOW SLAVE STATUS`的`Slave_IO_Running`直接映射.
* `HasReplicationFilters`: 如果设置了复制过滤规则, 此值为true.
* `SupportsOracleGTID`: true if cnfigured with `gtid_mode` (Oracle MySQL >= 5.6)
> 有点歧义, 是说`gtid_mode` 为ON吗? 需要看代码
* `UsingOracleGTID`: true if replica replicates via Oracle GTID
* `UsingMariaDBGTID`:  true if replica replicates via MariaDB GTID
* `UsingPseudoGTID`: true if replica known to have Pseudo-GTID coordinates (see related `DetectPseudoGTIDQuery` config)
* `ReadBinlogCoordinates`: (复制时) 从主库读取到的binlog坐标 (即 `IO_THREAD` 拉取的内容) .
> 应该就是Master\_Log\_File 和 Read\_Master\_Log\_Pos. 需要看代码
* `ExecBinlogCoordinates`: (复制时) 现在正在应用的主库binlog坐标 (即 `SQL_THREAD` 执行到的位置).
> 应该就是Relay\_Master\_Log\_File 和 Exec\_Master\_Log\_Pos. 需要看代码
* `RelaylogCoordinates`: (复制时) 现在正在执行的中继日志的坐标.
> 应该就是Relay\_Log\_File 和 Relay\_Log\_Pos. 需要看代码
* `LastSQLError`: `SHOW SLAVE STATUS` 中的`Last_SQL_Error` .
* `LastIOError`:  `SHOW SLAVE STATUS` 中的`Last_IO_Error` .
* `SecondsBehindMaster`:  从`SHOW SLAVE STATUS`的`Seconds_Behind_Master`的直接映射 `"Valid": false`表示`NULL`
> 如sql\_thread停止后, Seconds\_Behind\_Master: NULL
* `SQLDelay`: change master语句中的 `MASTER_DELAY` .
* `ExecutedGtidSet`: if using Oracle GTID, the executed GTID set
> 怀疑就是SHOW SLAVE STATUS中的`Executed_Gtid_Set` , 需要看代码.
* `ReplicationLagSeconds`: 当提供`ReplicationLagQuery` 时, 计算出的复制延迟. 否则与`SecondsBehindMaster` 相同.
* `Replicas`: 该实例的从库列表(hostname & port).
* `ClusterName`: 这个实例所关联的集群的名称; 唯一标识一个集群.
* `DataCenter`: (元数据) 数据中心的名称, 由DataCenterPattern配置变量推断.
* `PhysicalEnvironment`: (元数据) 环境名称, 由`PhysicalEnvironmentPattern`配置变量推断出.
* `ReplicationDepth`: 与master的距离 (master is `0`, direct replica is `1` and so on)
* `IsCoMaster`: 当实例是双主的一部分时, 此值为true.
* `IsLastCheckValid`: 上次尝试读取此实例是否成功.
* `IsUpToDate`: 这些数据是否是最新的.
* `IsRecentlyChecked`: 最近是否对该实例进行了读取尝试.
* `SecondsSinceLastSeen`: 自上次成功访问此实例以来经过的时间.
* `CountMySQLSnapshots`: 已知快照的数量（由 `orchestrator-agent` 提供的数据）
* `IsCandidate`: (元数据) 当这个实例通过`register-candidate` CLI命令被标记为候选实例时为`true`. 可以在崩溃恢复中使用, 以确定故障转移选项的优先级
* `UnresolvedHostname`: name this host *unresolves* to, as indicated by the `register-hostname-unresolve` CLI command

### Cheatsheet
下面是几个有用的API使用例子:

* 获取集群的一般信息:

```json
curl -s "http://my.orchestrator.service.com/api/cluster-info/my_cluster" | jq .

{
  "ClusterName": "my-cluster-fqdn:3306",
  "ClusterAlias": "my_cluster",
  "ClusterDomain": "my-cluster.com",
  "CountInstances": 10,
  "HeuristicLag": 0,
  "HasAutomatedMasterRecovery": true,
  "HasAutomatedIntermediateMasterRecovery": true
}
```
* 找到`my_cluster`中没有启动binary log的主机:

```javascript
curl -s "http://my.orchestrator.service.com/api/cluster/alias/my_cluster" | jq '.[] | select(.LogBinEnabled==false) .Key.Hostname' -r
```
* 找到`my_cluster`的master的直接副本

```json
curl -s "http://my.orchestrator.service.com/api/cluster/alias/my_cluster" | jq '.[] | select(.ReplicationDepth==1) .Key.Hostname' -r
```
或:

```json
master=$(curl -s "http://my.orchestrator.service.com/api/cluster-info/my_cluster" | jq '.ClusterName' | tr ':' '/')
curl -s "http://my.orchestrator.service.com/api/instance-replicas/${master}" | jq '.[] | .Key.Hostname' -r
```
* Find all intermediate masters in `my_cluster`:

```json
curl -s "http://my.orchestrator.service.com/api/cluster/alias/my_cluster" | jq '.[] | select(.MasterKey.Hostname!="") | select(.Replicas!=[]) .Key.Hostname'
```
