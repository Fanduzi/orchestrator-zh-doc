# Scripting samples
# [Scripting samples](https://github.com/openark/orchestrator/blob/master/docs/script-samples.md)
本文档介绍了`orchestrator`的脚本用法和思想.

### show all clusters with aliases 显示所有有别名的集群
```bash
$ orchestrator-client -c clusters-alias
mysql-9766.dc1.domain.net:3306,cl1
mysql-0909.dc1.domain.net:3306,olap
mysql-0246.dc1.domain.net:3306,mycluster
mysql-1111.dc1.domain.net:3306,oltp1
mysql-9002.dc1.domain.net:3306,oltp2
mysql-3972.dc1.domain.net:3306,oltp3
mysql-0019.dc1.domain.net:3306,oltp4
```
### Show only aliases 仅展示别名
```bash
$ orchestrator-client -c clusters-alias | cut -d"," -f2 | sort
cl1
mycluster
olap
oltp1
oltp2
oltp3
oltp4
```
#### master of cluster 集群主库
```bash
$ orchestrator-client -c which-cluster-master -alias mycluster
mysql-0246.dc1.domain.net:3306
```
#### All instances of cluster 集群所有实例
```bash
$ orchestrator-client -c which-cluster-instances -alias mycluster
mysql-0246.dc1.domain.net:3306
mysql-1357.dc2.domain.net:3306
mysql-bb00.dc1.domain.net:3306
mysql-00ff.dc1.domain.net:3306
mysql-8181.dc2.domain.net:3306
mysql-2222.dc1.domain.net:3306
mysql-ecec.dc2.domain.net:3306
```
上述内容表明`orchestrator`对复制拓扑的了解. 列表中包括可能offline/broken的服务器

#### Shell loop over instances
```bash
$ orchestrator-client -c which-cluster-instances -alias mycluster | cut -d":" -f 1 | while read h ; do echo "Host is $h" ; done
Host is mysql-0246.dc1.domain.net
Host is mysql-1357.dc2.domain.net
Host is mysql-bb00.dc1.domain.net
Host is mysql-00ff.dc1.domain.net
Host is mysql-8181.dc2.domain.net
Host is mysql-2222.dc1.domain.net
Host is mysql-ecec.dc2.domain.net
```
> 也许以后用这个比写SQL查CMDB方便了...

#### disable semi sync on cluster
```bash
$ orchestrator-client -c which-cluster-instances -alias mycluster | while read i ; do
  orchestrator-client -c disable-semi-sync-master -i $i
done
mysql-0246.dc1.domain.net:3306
mysql-1357.dc2.domain.net:3306
mysql-bb00.dc1.domain.net:3306
mysql-00ff.dc1.domain.net:3306
mysql-8181.dc2.domain.net:3306
mysql-2222.dc1.domain.net:3306
mysql-ecec.dc2.domain.net:3306
```
#### enable semi sync on cluster master
```bash
$ orchestrator-client -c which-cluster-master -alias mycluster | while read i ; do
  orchestrator-client -c enable-semi-sync-master -i $i
done
mysql-0246.dc1.domain.net:3306
```
#### Let's try again. This time disable semi sync on all instances *except master 在除主库外的所有的实例上禁用半同步*
```bash
$ master=$(orchestrator-client -c which-cluster-master -alias mycluster)
$ orchestrator-client -c which-cluster-instances -alias mycluster | grep -v $master | while read i ; do
  orchestrator-client -c disable-semi-sync-master -i $i
done
```
#### Likewise, set read-only on all replicas 在所有的副本上设置只读
```bash
$ orchestrator-client -c which-cluster-instances -alias mycluster | grep -v $master | while read i ; do
  orchestrator-client -c set-read-only -i $i
done
```
#### We don't really need to loop. We can use ccql
[ccql](https://github.com/github/ccql)是一个并发的、多服务器的MySQL客户端. 它与一般的脚本, 特别是与`orchestrator`发挥得很好

```bash
$ orchestrator-client -c which-cluster-instances -alias mycluster | grep -v $master | ccql -C ~/.my.cnf -q "set global read_only=1"
```
#### Extract master hostname (no ":3306")
```bash
$ master_host=$(orchestrator-client -c which-cluster-master -alias mycluster | cut -d":" -f1)
$ echo $master_host
mysql-0246.dc1.domain.net
```
We will use the `master_host` variable following.

#### Using the API to show all data of a specific host
```json
$ orchestrator-client -c api -path instance/$master_host/3306 | jq .
{
  "Key": {
    "Hostname": "mysql-0246.dc1.domain.net",
    "Port": 3306
  },
  "InstanceAlias": "",
  "Uptime": 12203,
  "ServerID": 65884260,
  "ServerUUID": "3e87bd92-2be0-13e8-ac1b-008cda544064",
  "Version": "5.7.18-log",
  "VersionComment": "MySQL Community Server (GPL)",
  "FlavorName": "MySQL",
  "ReadOnly": false,
  "Binlog_format": "ROW",
  "BinlogRowImage": "FULL",
  "LogBinEnabled": true,
  "LogReplicationUpdatesEnabled": true,
  "SelfBinlogCoordinates": {
    "LogFile": "mysql-bin.000002",
    "LogPos": 333006336,
    "Type": 0
  },
  "MasterKey": {
    "Hostname": "",
    "Port": 0
  },
  "IsDetachedMaster": false,
  "ReplicationSQLThreadRuning": false,
  "ReplicationIOThreadRuning": false,
  "HasReplicationFilters": false,
  "GTIDMode": "OFF",
  "SupportsOracleGTID": false,
  "UsingOracleGTID": false,
  "UsingMariaDBGTID": false,
  "UsingPseudoGTID": true,
  "ReadBinlogCoordinates": {
    "LogFile": "",
    "LogPos": 0,
    "Type": 0
  },
  "ExecBinlogCoordinates": {
    "LogFile": "",
    "LogPos": 0,
    "Type": 0
  },
  "IsDetached": false,
  "RelaylogCoordinates": {
    "LogFile": "",
    "LogPos": 0,
    "Type": 1
  },
  "LastSQLError": "",
  "LastIOError": "",
  "SecondsBehindMaster": {
    "Int64": 0,
    "Valid": false
  },
  "SQLDelay": 0,
  "ExecutedGtidSet": "",
  "GtidPurged": "",
  "ReplicationLagSeconds": {
    "Int64": 0,
    "Valid": true
  },
  "Replicas": [
    {
      "Hostname": "mysql-2222.dc1.domain.net",
      "Port": 3306
    },
    {
      "Hostname": "mysql-00ff.dc1.domain.net",
      "Port": 3306
    },
    {
      "Hostname": "mysql-1357.dc2.domain.net",
      "Port": 3306
    }
  ],
  "ClusterName": "mysql-0246.dc1.domain.net:3306",
  "SuggestedClusterAlias": "mycluster",
  "DataCenter": "dc1",
  "PhysicalEnvironment": "",
  "ReplicationDepth": 0,
  "IsCoMaster": false,
  "HasReplicationCredentials": false,
  "ReplicationCredentialsAvailable": false,
  "SemiSyncEnforced": false,
  "SemiSyncMasterEnabled": true,
  "SemiSyncReplicaEnabled": false,
  "LastSeenTimestamp": "2018-03-21 04:40:38",
  "IsLastCheckValid": true,
  "IsUpToDate": true,
  "IsRecentlyChecked": true,
  "SecondsSinceLastSeen": {
    "Int64": 2,
    "Valid": true
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
  "LastDiscoveryLatency": 7233416
}
```
#### Extract the hostname from the JSON:
```bash
$ orchestrator-client -c api -path instance/$master_host/3306 | jq .Key.Hostname -r
mysql-0246.dc1.domain.net
```
#### Extract master's hostname from the JSON:
```bash
$ orchestrator-client -c api -path instance/$master_host/3306 | jq .MasterKey.Hostname -r

(empty, this is the master)
```
#### Another way of listing all hostnames in a cluster: using API and jq
```bash
$ orchestrator-client -c api -path cluster/alias/mycluster | jq .[].Key.Hostname -r
mysql-0246.dc1.domain.net
mysql-1357.dc2.domain.net
mysql-bb00.dc1.domain.net
mysql-00ff.dc1.domain.net
mysql-8181.dc2.domain.net
mysql-2222.dc1.domain.net
mysql-ecec.dc2.domain.net
```
#### Show the master host for each member in the cluster:
```Plain Text
$ orchestrator-client -c api -path cluster/alias/mycluster | jq .[].MasterKey.Hostname -r

mysql-0246.dc1.domain.net
mysql-00ff.dc1.domain.net
mysql-0246.dc1.domain.net
mysql-bb00.dc1.domain.net
mysql-0246.dc1.domain.net
mysql-bb00.dc1.domain.net
```
#### What is the master hostname of a specific instance?
```bash
$ orchestrator-client -c api -path instance/mysql-bb00.dc1.domain.net/3306 | jq .MasterKey.Hostname -r
mysql-00ff.dc1.domain.net
```
#### How many replicas to a specific instance?
```bash
$ orchestrator-client -c api -path instance/$master_host/3306 | jq '.Replicas | length'
3
```
#### How many replicas to each of a cluster's members?
```bash
$ orchestrator-client -c api -path cluster/alias/mycluster | jq '.[].Replicas | length'
3
0
2
1
0
0
0
```
#### Another way of listing all replicas
We filter out those that don't have output for `show slave status`:

```bash
$ orchestrator-client -c which-cluster-instances -alias mycluster | ccql -C ~/.my.cnf -q "show slave status" | awk '{print $1}'
mysql-00ff.dc1.domain.net:3306
mysql-bb00.dc1.domain.net:3306
mysql-2222.dc1.domain.net:3306
mysql-ecec.dc2.domain.net:3306
mysql-1357.dc2.domain.net:3306
mysql-8181.dc2.domain.net:3306
```
#### Followup, restart replication on all cluster's instances
```bash
$ orchestrator-client -c which-cluster-instances -alias mycluster | ccql -C ~/.my.cnf -q "show slave status" | awk '{print $1}' | ccql -C ~/.my.cnf -q "stop slave; start slave;"
```
#### I'd like to apply changes to replication, without changing the replica's state (if it's running, I want it to keep running. If it's not running, I don't want to start replication)
我想在不改变复制状态的情况下对复制进行修改（如果它正在运行，我希望它能继续运行。如果它没有运行，我就不想启动复制)

```bash
$ orchestrator-client -c restart-replica-statements -i mysql-bb00.dc1.domain.net -query "change master to auto_position=1" | jq .[] -r
stop slave io_thread;
stop slave sql_thread;
change master to auto_position=1;
start slave sql_thread;
start slave io_thread;
```
Compare with:

```bash
$ orchestrator-client -c stop-replica -i mysql-bb00.dc1.domain.net
mysql-bb00.dc1.domain.net:3306

$ orchestrator-client -c restart-replica-statements -i mysql-bb00.dc1.domain.net -query "change master to auto_position=1" | jq .[] -r
change master to auto_position=1;
```
上面只是输出语句，我们需要把它们推回给服务器

```bash
orchestrator-client -c restart-replica-statements -i mysql-bb00.dc1.domain.net -query "change master to auto_position=1" | jq .[] -r | mysql -h mysql-bb00.dc1.domain.net
```
#### In which DC (data center) is a specific instance?
这个问题和下一个问题假设`DetectDataCenterQuery`或`DataCenterPattern`已经配置好.

```bash
$ orchestrator-client -c api -path instance/mysql-bb00.dc1.domain.net/3306 | jq '.DataCenter'
dc1
```
#### In which DCs is a cluster deployed, and how many hosts in each DC?
```bash
$ orchestrator-client -c api -path cluster/mycluster | jq '.[].DataCenter' -r | sort | uniq -c
  4 dc1
  3 dc2
```
#### Which replicas are replicating cross DC?
```bash
$ orchestrator-client -c api -path cluster/mycluster |
    jq '.[] | select(.MasterKey.Hostname != "") |
        (.Key.Hostname + ":" + (.Key.Port | tostring) + " " + .DataCenter + " " + .MasterKey.Hostname + "/" + (.MasterKey.Port | tostring))' -r |
    while read h dc m ; do
      orchestrator-client -c api -path "instance/$m" | jq '.DataCenter' -r |
        { read master_dc ; [ "$master_dc" != "$dc" ] && echo $h ; } ;
    done

mysql-bb00.dc1.domain.net:3306
mysql-8181.dc2.domain.net:3306
```














