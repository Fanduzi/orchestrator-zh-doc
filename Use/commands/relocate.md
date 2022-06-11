# 说明

> 说人话:
> 将一个节点 A change master to指定节点B
> B不能是A的从库或A从库的从库



在另一个(目标)实例下重新定位一个副本。目的地的选择几乎是任意的;
它(-d指定的实例)必须不是实例(-i指定的实例)的子/孙节点, 否则它可以在任何地方，并且可以是普通副本或Binlog server. Orchestrator将选择最佳的行动方案来重新定位副本。
如果目标实例不能作为主实例（例如，没有二进制日志，版本不兼容，binlog格式不兼容等），则不采取任何行动。
>Relocate a replica beneath another (destination) instance. The choice of destination is almost arbitrary;  
it must not be a child/descendant of the instance, but otherwise it can be anywhere, and can be a normal replica or a binlog server.
Orchestrator will choose the best course of action to relocate the replica.  
No action taken when destination instance cannot act as master (e.g. has no binary logs, is of incompatible version, incompatible binlog format etc.)  


示例:
```bash
orchestrator -c relocate -i replica.to.relocate.com -d instance.that.becomes.its.master  
```

```bash
orchestrator -c relocate -d destination.instance.that.becomes.its.master  
```
> 如果不指定-i, 则隐式假定为 127.0.0.1:3306
 -i not given, implicitly assumed local hostname  

这个命令在之前叫做`relocate-below`
> (this command was previously named "relocate-below")


# 实例
![](https://raw.githubusercontent.com/Fanduzi/Figure_bed/master/img/picgo/2022/06/11/20-23-56-20220611202355-503154.png)

## 将172.16.120.12:3306 change 到 172.16.120.11:3306下
```bash
orchestrator-client -b 'dba:superpass' -c relocate -i 172.16.120.12 -d 172.16.120.11
172.16.120.12:3306<172.16.120.11:3306
```
![](https://raw.githubusercontent.com/Fanduzi/Figure_bed/master/img/picgo/2022/06/11/20-27-02-20220611202701-404931.png)
日志
```go
Jun 11 20:25:56 centos-4 yzh-orchestrator: [martini] Started GET /api/relocate/172.16.120.12/3306/172.16.120.11/3306 for [::1]:45567
Jun 11 20:25:56 centos-4 yzh-orchestrator: 2022-06-11 20:25:56 INFO Will move 172.16.120.12:3306 below 172.16.120.11:3306 via GTID
Jun 11 20:25:57 centos-4 yzh-orchestrator: 2022-06-11 20:25:57 INFO Stopped replication on 172.16.120.12:3306, Self:mysql-bin.000002:61870700, Exec:mysql-bin.000034:13789895
Jun 11 20:25:57 centos-4 yzh-orchestrator: 2022-06-11 20:25:57 DEBUG ChangeMasterTo: will attempt changing master on 172.16.120.12:3306 to 172.16.120.11:3306, mysql-bin.000006:23265316
Jun 11 20:25:57 centos-4 yzh-orchestrator: 2022-06-11 20:25:57 INFO ChangeMasterTo: Changed master on 172.16.120.12:3306 to: 172.16.120.11:3306, mysql-bin.000006:23265316. GTID: true
Jun 11 20:25:57 centos-4 yzh-orchestrator: 2022-06-11 20:25:57 DEBUG semi-sync: analysis results for recovery of cluster 172.16.120.10:3306:
Jun 11 20:25:57 centos-4 yzh-orchestrator: 2022-06-11 20:25:57 DEBUG semi-sync: master = 172.16.120.11:3306, master semi-sync wait count = 1, master semi-sync replica count = 0
Jun 11 20:25:57 centos-4 yzh-orchestrator: 2022-06-11 20:25:57 DEBUG semi-sync: possible semi-sync replicas (in priority order): (none)
Jun 11 20:25:57 centos-4 yzh-orchestrator: 2022-06-11 20:25:57 DEBUG semi-sync: always-async replicas: (none)
Jun 11 20:25:57 centos-4 yzh-orchestrator: 2022-06-11 20:25:57 DEBUG semi-sync: excluded replicas (defunct): (none)
Jun 11 20:25:57 centos-4 yzh-orchestrator: 2022-06-11 20:25:57 DEBUG semi-sync: suggested actions: (none)
Jun 11 20:25:57 centos-4 yzh-orchestrator: 2022-06-11 20:25:57 INFO semi-sync: 172.16.120.12:3306: no action taken; this may lead to future recoveries
Jun 11 20:25:57 centos-4 yzh-orchestrator: 2022-06-11 20:25:57 INFO Started replication on 172.16.120.12:3306
Jun 11 20:25:57 centos-4 yzh-orchestrator: [martini] Completed 200 OK in 552.812542ms
```

## 无法relocate到自己的子孙节点下
![](https://raw.githubusercontent.com/Fanduzi/Figure_bed/master/img/picgo/2022/06/11/20-27-02-20220611202701-404931.png)
```bash
orchestrator-client -b 'dba:123' -c relocate -i 172.16.120.11 -d 172.16.120.12
2022-06-11 20:27:25 ERROR relocate: 172.16.120.12:3306 is a descendant of 172.16.120.11:3306
```


> orchestrator-client 不支持同义词 relocate-below
> ```go
> #orchestrator-client -b 'dba:123' -c relocate-below -i 172.16.120.12 -d 172.16.120.11
> orchestrator-client[8220]: Unsupported command relocate-below
> ```


# orchestrator命令
无需认证, 同时会输出日志到终端
```go
#/usr/local/yzh-orchestrator/orchestrator -c relocate -i 172.16.120.12 -d 172.16.120.11
2022-06-11 20:29:56 DEBUG Hostname unresolved yet: 172.16.120.12
2022-06-11 20:29:56 DEBUG Cache hostname resolve 172.16.120.12 as 172.16.120.12
2022-06-11 20:29:56 DEBUG Hostname unresolved yet: 172.16.120.11
2022-06-11 20:29:56 DEBUG Cache hostname resolve 172.16.120.11 as 172.16.120.11
2022-06-11 20:29:56 DEBUG Connected to orchestrator backend: orchestrator:?@tcp(172.16.120.13:3306)/orchestrator?timeout=5s&readTimeout=5s&rejectReadOnly=true&interpolateParams=true
2022-06-11 20:29:56 DEBUG Orchestrator pool SetMaxOpenConns: 128
2022-06-11 20:29:56 DEBUG Initializing orchestrator
2022-06-11 20:29:56 INFO Connecting to backend 172.16.120.13:3306: maxConnections: 128, maxIdleConns: 32
2022-06-11 20:29:56 INFO Will move 172.16.120.12:3306 below 172.16.120.11:3306 via GTID
2022-06-11 20:29:56 INFO auditType:begin-maintenance instance:172.16.120.12:3306 cluster:172.16.120.10:3306 message:maintenanceToken: 5, owner: root, reason: move below 172.16.120.11:3306
2022-06-11 20:29:56 DEBUG Hostname unresolved yet: 172.16.120.10
2022-06-11 20:29:56 DEBUG Cache hostname resolve 172.16.120.10 as 172.16.120.10
2022-06-11 20:29:56 DEBUG Hostname unresolved yet: 172.16.120.10
2022-06-11 20:29:56 DEBUG Cache hostname resolve 172.16.120.10 as 172.16.120.10
2022-06-11 20:29:56 INFO Stopped replication on 172.16.120.12:3306, Self:mysql-bin.000002:62043964, Exec:mysql-bin.000034:13975059
2022-06-11 20:29:56 DEBUG ChangeMasterTo: will attempt changing master on 172.16.120.12:3306 to 172.16.120.11:3306, mysql-bin.000006:23438580
2022-06-11 20:29:56 INFO ChangeMasterTo: Changed master on 172.16.120.12:3306 to: 172.16.120.11:3306, mysql-bin.000006:23438580. GTID: true
2022-06-11 20:29:56 DEBUG semi-sync: analysis results for recovery of cluster 172.16.120.10:3306:
2022-06-11 20:29:56 DEBUG semi-sync: master = 172.16.120.11:3306, master semi-sync wait count = 1, master semi-sync replica count = 0
2022-06-11 20:29:56 DEBUG semi-sync: possible semi-sync replicas (in priority order): (none)
2022-06-11 20:29:56 DEBUG semi-sync: always-async replicas: (none)
2022-06-11 20:29:56 DEBUG semi-sync: excluded replicas (defunct): (none)
2022-06-11 20:29:56 DEBUG semi-sync: suggested actions: (none)
2022-06-11 20:29:56 INFO semi-sync: 172.16.120.12:3306: no action taken; this may lead to future recoveries
2022-06-11 20:29:56 INFO Started replication on 172.16.120.12:3306
2022-06-11 20:29:56 INFO auditType:move-below-gtid instance:172.16.120.12:3306 cluster:172.16.120.10:3306 message:moved 172.16.120.12:3306 below 172.16.120.11:3306
2022-06-11 20:29:56 INFO auditType:end-maintenance instance:172.16.120.12:3306 cluster:172.16.120.10:3306 message:maintenanceToken: 5
2022-06-11 20:29:56 INFO auditType:relocate-below instance:172.16.120.12:3306 cluster:172.16.120.10:3306 message:relocated 172.16.120.12:3306 below 172.16.120.11:3306
172.16.120.12:3306<172.16.120.11:3306
```

> 支持同义词relocate-below
> ```go
> #/usr/local/yzh-orchestrator/orchestrator -c relocate-below -i 172.16.120.12 -d 172.16.120.10
> 2022-06-11 20:57:11 DEBUG Hostname unresolved yet: 172.16.120.12
> 2022-06-11 20:57:11 DEBUG Cache hostname resolve 172.16.120.12 as 172.16.120.12
> 2022-06-11 20:57:11 DEBUG Hostname unresolved yet: 172.16.120.10
> 2022-06-11 20:57:11 DEBUG Cache hostname resolve 172.16.120.10 as 172.16.120.10
> 2022-06-11 20:57:11 DEBUG Connected to orchestrator backend: orchestrator:?@tcp(172.16.120.13:3306)/orchestrator?timeout=5s&readTimeout=5s&rejectReadOnly=true&interpolateParams=true
> 2022-06-11 20:57:11 DEBUG Orchestrator pool SetMaxOpenConns: 128
> 2022-06-11 20:57:11 DEBUG Initializing orchestrator
> 2022-06-11 20:57:11 INFO Connecting to backend 172.16.120.13:3306: maxConnections: 128, maxIdleConns: 32
> 2022-06-11 20:57:11 INFO Will move 172.16.120.12:3306 below 172.16.120.10:3306 via GTID
> 2022-06-11 20:57:11 INFO auditType:begin-maintenance instance:172.16.120.12:3306 cluster:172.16.120.10:3306 message:maintenanceToken: 6, owner: root, reason: move below 172.16.120.10:3306
> 2022-06-11 20:57:11 DEBUG Hostname unresolved yet: 172.16.120.11
> 2022-06-11 20:57:11 DEBUG Cache hostname resolve 172.16.120.11 as 172.16.120.11
> 2022-06-11 20:57:11 DEBUG Hostname unresolved yet: 172.16.120.11
> 2022-06-11 20:57:11 DEBUG Cache hostname resolve 172.16.120.11 as 172.16.120.11
> 2022-06-11 20:57:11 INFO Stopped replication on 172.16.120.12:3306, Self:mysql-bin.000002:63223688, Exec:mysql-bin.000006:24618668
> 2022-06-11 20:57:11 DEBUG ChangeMasterTo: will attempt changing master on 172.16.120.12:3306 to 172.16.120.10:3306, mysql-bin.000034:15233474
> 2022-06-11 20:57:11 INFO ChangeMasterTo: Changed master on 172.16.120.12:3306 to: 172.16.120.10:3306, mysql-bin.000034:15233474. GTID: true
> 2022-06-11 20:57:11 DEBUG semi-sync: analysis results for recovery of cluster 172.16.120.10:3306:
> 2022-06-11 20:57:11 DEBUG semi-sync: master = 172.16.120.10:3306, master semi-sync wait count = 1, master semi-sync replica count = 1
> 2022-06-11 20:57:11 DEBUG semi-sync: possible semi-sync replicas (in priority order): (none)
> 2022-06-11 20:57:11 DEBUG semi-sync: always-async replicas:
> 2022-06-11 20:57:11 DEBUG semi-sync: - 172.16.120.11:3306: semi-sync enabled = true, priority = 0, promotion rule = neutral, last check = true, replicating = true
> 2022-06-11 20:57:11 DEBUG semi-sync: excluded replicas (defunct): (none)
> 2022-06-11 20:57:11 DEBUG semi-sync: suggested actions: (none)
> 2022-06-11 20:57:11 INFO semi-sync: 172.16.120.12:3306: no action taken; this may lead to future recoveries
> 2022-06-11 20:57:11 INFO Started replication on 172.16.120.12:3306
> 2022-06-11 20:57:11 INFO auditType:move-below-gtid instance:172.16.120.12:3306 cluster:172.16.120.10:3306 message:moved 172.16.120.12:3306 below 172.16.120.10:3306
> 2022-06-11 20:57:11 INFO auditType:end-maintenance instance:172.16.120.12:3306 cluster:172.16.120.10:3306 message:maintenanceToken: 6
> 2022-06-11 20:57:11 INFO auditType:relocate-below instance:172.16.120.12:3306 cluster:172.16.120.10:3306 message:relocated 172.16.120.12:3306 below 172.16.120.10:3306
> 172.16.120.12:3306<172.16.120.10:3306
> ```


## curl
```go
#curl --basic --user dba:123 -s "http://localhost:3000/api/relocate/172.16.120.12/3306/172.16.120.10/3306" | jq '.'
{
  "Code": "OK",
  "Message": "Instance 172.16.120.12:3306 relocated below 172.16.120.10:3306",
  "Details": {
    "Key": {
      "Hostname": "172.16.120.12",
      "Port": 3306
    },
    "InstanceAlias": "centos-3_3306",
    "Uptime": 2430317,
    "ServerID": 5557,
    "ServerUUID": "90d43a13-6581-11ec-8260-0050563108d2",
    "Version": "5.7.32-35-log",
    "VersionComment": "Percona Server (GPL), Release 35, Revision 5688520",
    "FlavorName": "",
    "ReadOnly": true,
    "Binlog_format": "ROW",
    "BinlogRowImage": "FULL",
    "LogBinEnabled": true,
    "LogSlaveUpdatesEnabled": true,
    "LogReplicationUpdatesEnabled": true,
    "SelfBinlogCoordinates": {
      "LogFile": "mysql-bin.000002",
      "LogPos": 64174456,
      "Type": 0
    },
    "MasterKey": {
      "Hostname": "172.16.120.10",
      "Port": 3306
    },
    "MasterUUID": "",
    "AncestryUUID": "8a1094ff-657b-11ec-85f8-0050563b7b42,90d43a13-6581-11ec-8260-0050563108d2",
    "IsDetachedMaster": false,
    "Slave_SQL_Running": true,
    "ReplicationSQLThreadRuning": true,
    "Slave_IO_Running": true,
    "ReplicationIOThreadRuning": true,
    "ReplicationSQLThreadState": 1,
    "ReplicationIOThreadState": 1,
    "HasReplicationFilters": false,
    "GTIDMode": "ON",
    "SupportsOracleGTID": true,
    "UsingOracleGTID": true,
    "UsingMariaDBGTID": false,
    "UsingPseudoGTID": false,
    "ReadBinlogCoordinates": {
      "LogFile": "",
      "LogPos": 4,
      "Type": 0
    },
    "ExecBinlogCoordinates": {
      "LogFile": "",
      "LogPos": 0,
      "Type": 0
    },
    "IsDetached": false,
    "RelaylogCoordinates": {
      "LogFile": "mysql-relay-bin.000002",
      "LogPos": 4,
      "Type": 1
    },
    "LastSQLError": "",
    "LastIOError": "",
    "SecondsBehindMaster": {
      "Int64": 0,
      "Valid": true
    },
    "SQLDelay": 0,
    "ExecutedGtidSet": "8a1094ff-657b-11ec-85f8-0050563b7b42:1-188491,\n90d43a13-6581-11ec-8260-0050563108d2:1",
    "GtidPurged": "8a1094ff-657b-11ec-85f8-0050563b7b42:1-5894",
    "GtidErrant": "",
    "SlaveLagSeconds": {
      "Int64": 0,
      "Valid": true
    },
    "ReplicationLagSeconds": {
      "Int64": 0,
      "Valid": true
    },
    "SlaveHosts": [],
    "Replicas": [],
    "ClusterName": "172.16.120.10:3306",
    "SuggestedClusterAlias": "orc_prod_infra_broker",
    "DataCenter": "cn-north-1c",
    "Region": "HuaweiCloud",
    "PhysicalEnvironment": "prod",
    "ReplicationDepth": 1,
    "IsCoMaster": false,
    "HasReplicationCredentials": true,
    "ReplicationCredentialsAvailable": true,
    "SemiSyncAvailable": true,
    "SemiSyncPriority": 100,
    "SemiSyncMasterEnabled": false,
    "SemiSyncReplicaEnabled": true,
    "SemiSyncMasterTimeout": 10000,
    "SemiSyncMasterWaitForReplicaCount": 1,
    "SemiSyncMasterStatus": false,
    "SemiSyncMasterClients": 0,
    "SemiSyncReplicaStatus": true,
    "LastSeenTimestamp": "",
    "IsLastCheckValid": true,
    "IsUpToDate": true,
    "IsRecentlyChecked": true,
    "SecondsSinceLastSeen": {
      "Int64": 0,
      "Valid": false
    },
    "CountMySQLSnapshots": 0,
    "IsCandidate": false,
    "PromotionRule": "neutral",
    "IsDowntimed": false,
    "DowntimeReason": "",
    "DowntimeOwner": "",
    "DowntimeEndTimestamp": "",
    "ElapsedDowntime": 0,
    "UnresolvedHostname": "",
    "AllowTLS": false,
    "Problems": [],
    "LastDiscoveryLatency": 9092526,
    "ReplicationGroupName": "",
    "ReplicationGroupIsSinglePrimary": false,
    "ReplicationGroupMemberState": "",
    "ReplicationGroupMemberRole": "",
    "ReplicationGroupMembers": [],
    "ReplicationGroupPrimaryInstanceKey": {
      "Hostname": "",
      "Port": 0
    }
  }
}
```


> 可以使用同义词relocate-below
> ```go
> curl --basic --user dba:123 -s "http://localhost:3000/api/relocate-below/172.16.120.12/3306/172.16.120.11/3306" | jq '.'
> ```
