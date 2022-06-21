# [Orchestrator Failover过程源码分析-I](http://fuxkdb.com/2022/04/28/2022-05-11-Orchestrator-Failover%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-I/)

## 模拟故障

使用测试环境, 模拟`3307`集群故障

| 角色 | IP            | 端口 | 主机名   |
| ---- | ------------- | ---- | -------- |
| 主库 | 172.16.120.10 | 3307 | centos-1 |
| 从库 | 172.16.120.11 | 3307 | centos-2 |
| 从库 | 172.16.120.12 | 3307 | centos-3 |


关闭3307主库`172.16.120.10:3307`
```bash
[2022-04-25 13:10:56][root@centos-1 13:10:56 ~]
[2022-04-25 13:11:22]#systemctl stop mysql3307


mysql日志
2022-04-25T13:11:35.959667+08:00 0 [Note] /usr/local/mysql5732/bin/mysqld: Shutdown complete
```


## 源码分析

我的思路是通过日志找入口.
在下文中:
- 对主库简称为centos-1
- 对两个从库分别简称为:
    - centos-2
    - centos-3
```sql
[mysql] 2022/04/25 13:11:27 packets.go:37: unexpected EOF
2022-04-25 13:11:27 ERROR invalid connection
2022-04-25 13:11:27 ERROR ReadTopologyInstance(172.16.120.10:3307) show global status like 'Uptime': Error 1053: Server shutdown in progress
[mysql] 2022/04/25 13:11:27 packets.go:37: unexpected EOF
2022-04-25 13:11:27 ERROR invalid connection
[mysql] 2022/04/25 13:11:27 packets.go:37: unexpected EOF
2022-04-25 13:11:27 ERROR invalid connection
2022-04-25 13:11:27 ERROR dial tcp 172.16.120.10:3307: connect: connection refused
2022-04-25 13:11:28 ERROR dial tcp 172.16.120.10:3307: connect: connection refused
2022-04-25 13:11:28 DEBUG writeInstance: will not update database_instance due to error: invalid connection
2022-04-25 13:11:32 WARNING DiscoverInstance(172.16.120.10:3307) instance is nil in 0.104s (Backend: 0.001s, Instance: 0.103s), error=dial tcp 172.16.120.10:3307: connect: connection refused
2022-04-25 13:11:33 DEBUG analysis: ClusterName: 172.16.120.10:3307, IsMaster: true, LastCheckValid: false, LastCheckPartialSuccess: false, CountReplicas: 2, CountValidReplicas: 2, CountValidReplicatingReplicas: 2, CountLaggingReplicas: 0, CountDelayedReplicas: 0, CountReplicasFailingToConnectToMaster: 0
2022-04-25 13:11:33 INFO executeCheckAndRecoverFunction: proceeding with UnreachableMaster detection on 172.16.120.10:3307; isActionable?: false; skipProcesses: false
2022-04-25 13:11:33 INFO topology_recovery: detected UnreachableMaster failure on 172.16.120.10:3307
2022-04-25 13:11:33 INFO topology_recovery: Running 1 OnFailureDetectionProcesses hooks
```

关闭centos-1后, 从日志可以看出:
- orchestrator对centos-1的`一些探测`操作失败了
- executeCheckAndRecoverFunction: proceeding with UnreachableMaster

### 通过第二条信息找"入口"
全局搜索`executeCheckAndRecoverFunction: proceeding with`

搜索到函数executeCheckAndRecoverFunction

这个函数比较长, 先不看. 先看一下是谁调用 executeCheckAndRecoverFunction . 

搜索`executeCheckAndRecoverFunction(`, 搜到CheckAndRecover

继续搜索CheckAndRecover, 查到是ContinuousDiscovery在调用它


ContinuousDiscovery 在logic包中, 被http.standardHttp调用, 而http.standardHttp又被http.Http调用, http.Http是在启动orchestrator时被调用的
```go
go/cmd/orchestrator/main.go

// 截取部分代码

    switch {  
   case helpTopic != "":  
      app.HelpCommand(helpTopic)  
   case len(flag.Args()) == 0 || flag.Arg(0) == "cli":  
      app.CliWrapper(*command, *strict, *instance, *destination, *owner, *reason, *duration, *pattern, *clusterAlias, *pool, *hostnameFlag)  
   case flag.Arg(0) == "http":  
      app.Http(*discovery)  
   default:  
      fmt.Fprintln(os.Stderr, `Usage:  
  orchestrator --options... [cli|http]See complete list of commands:  
  orchestrator -c helpFull blown documentation:  
  orchestrator`)  
      os.Exit(1)  
   }}
```

也就是说, 当我们用以下命令启动orchestrator后
```
orchestrator -config orchestrator.conf.json -debug http
```

就会 http.Http -> http.standardHttp -> `go logic.ContinuousDiscovery()`


## ContinuousDiscovery都干了啥

### 持续发现
```go
// ContinuousDiscovery starts an asynchronuous infinite discovery process where instances are
// periodically investigated and their status captured, and long since unseen instances are  
// purged and forgotten.
ContinuousDiscovery启动一个永不停止的异步"发现"过程, 在这个过程中, 实例被周期性地调查并捕获它们的状态, 长期以来不可见的实例被清除和遗忘.
```
> 这段注释中 asynchronuous 还拼写错了, 应该是 asynchronous

ContinuousDiscovery 先启动一个协程
```go
func ContinuousDiscovery() {  
   ...
   go handleDiscoveryRequests()
```

handleDiscoveryRequests 在[Orchestrator Discover源码分析](https://blog.csdn.net/ashic/article/details/124773909)中介绍过
handleDiscoveryRequests迭代`discoveryQueue` channel 并在每个条目上调用[DiscoverInstance](http://fuxkdb.com/2022/04/26/2022-04-26-Orchestrator-Discover%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/#%E6%A0%91%E8%97%A4%E6%91%B8%E7%93%9CDiscoverInstance), 而[DiscoverInstance](http://fuxkdb.com/2022/04/26/2022-04-26-Orchestrator-Discover%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/#%E6%A0%91%E8%97%A4%E6%91%B8%E7%93%9CDiscoverInstance)又会调用[ReadTopologyInstanceBufferable](http://fuxkdb.com/2022/04/26/2022-04-26-Orchestrator-Discover%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/#gt-ReadTopologyInstanceBufferable), 后者会实际连接MySQL实例, 获取各种指标/参数信息, 最终将结果写入`database_instance database_instance` 表

那么`discoveryQueue`里的"数据"又是谁放进来的呢?, 有两个地方
- 通过命令行或前端页面手动触发"发现"时(本质是调用[orchestrator discover接口](http://fuxkdb.com/2022/04/26/2022-04-26-Orchestrator-Discover%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/#orchestrator-discover%E6%8E%A5%E5%8F%A3)), 会将指定instance的`ReplicaKey`和`MasterKey`放入discoveryQueue
- ContinuousDiscovery 还会创建一个`healthTick`定时器, 周期性(每秒)调用onHealthTick, onHealthTick会取出所有"过期"的instance, 放到discoverQueue中
```go
const HealthPollSeconds = 1

func ContinuousDiscovery() {
    ...省略部分代码
    healthTick := time.Tick(config.HealthPollSeconds * time.Second)
    ...省略部分代码
    
    for {
        select {
        case <-healthTick:
            go func() {
                onHealthTick()
            }()
    ...省略部分代码
}
```
```go
// onHealthTick handles the actions to take to discover/poll instances
func onHealthTick() {

...省略部分代码
    instanceKeys, err := inst.ReadOutdatedInstanceKeys() // 读出过期的实例. 过期的定义是: InstancePollSeconds秒未探测的connectable实例 或 2*InstancePollSeconds秒未探测的连接出现异常(hang)的实例

    // avoid any logging unless there's something to be done  
    if len(instanceKeys) > 0 {  
       for _, instanceKey := range instanceKeys {  
          if instanceKey.IsValid() {  
             discoveryQueue.Push(instanceKey)  
          }   
    }
}
```
那么就是说, 每秒钟(HealthPollSeconds=1), onHealthTick会把所有"过期"的实例放到discoveryQueue

#### 流程图
![](https://raw.githubusercontent.com/Fanduzi/Figure_bed/master/img/Downloads/2022/05/14/21-10-51-流程图-d254c0.jpeg)
### 需要与实例轮询(或大致)相同频率的常规操作
```go
func ContinuousDiscovery() {
    ...省略部分代码
    instancePollTick := time.Tick(instancePollSecondsDuration()) // InstancePollSeconds 默认5秒
    ...省略部分代码
    
    for {
        select {
        ...省略部分代码
        case <-instancePollTick: // 5秒一次
            go func() {
                // This tick does NOT do instance poll (these are handled by the oversampling discoveryTick)
                // But rather should invoke such routinely operations that need to be as (or roughly as) frequent
                // as instance poll
                if IsLeaderOrActive() {
                    go inst.UpdateClusterAliases()
                    go inst.ExpireDowntime()
                    go injectSeeds(&seedOnce)
                }
            }()
    ...省略部分代码
}
```
看起来都是些不太重要的操作

### 定时注入伪GTID
```go
const PseudoGTIDIntervalSeconds = 5
func ContinuousDiscovery() {
    ...省略部分代码
    autoPseudoGTIDTick := time.Tick(time.Duration(config.PseudoGTIDIntervalSeconds) * time.Second)
    ...省略部分代码
    
    for {
        select {
        ...省略部分代码
        case <-autoPseudoGTIDTick:
            go func() {
                if config.Config.AutoPseudoGTID && IsLeader() {
                    go InjectPseudoGTIDOnWriters()
                }
            }()
    ...省略部分代码
}
```
[Pseudo GTID](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Various/Pseudo%20GTID.md) , 不太重要, 现在还会有人不开GTID吗?

### 护理工作
```go
func ContinuousDiscovery() {
    ...省略部分代码
    caretakingTick := time.Tick(time.Minute)
    ...省略部分代码
    for {
        select {
        ...省略部分代码
        case <-caretakingTick:
            // Various periodic internal maintenance tasks
            go func() {
                if IsLeaderOrActive() {
                    go inst.RecordInstanceCoordinatesHistory()
                    go inst.ReviewUnseenInstances()
                    go inst.InjectUnseenMasters()

                    go inst.ForgetLongUnseenInstances()
                    go inst.ForgetLongUnseenClusterAliases()
                    go inst.ForgetUnseenInstancesDifferentlyResolved()
                    go inst.ForgetExpiredHostnameResolves()
                    go inst.DeleteInvalidHostnameResolves()
                    go inst.ResolveUnknownMasterHostnameResolves()
                    go inst.ExpireMaintenance()
                    go inst.ExpireCandidateInstances()
                    go inst.ExpireHostnameUnresolve()
                    go inst.ExpireClusterDomainName()
                    go inst.ExpireAudit()
                    go inst.ExpireMasterPositionEquivalence()
                    go inst.ExpirePoolInstances()
                    go inst.FlushNontrivialResolveCacheToDatabase()
                    go inst.ExpireInjectedPseudoGTID()
                    go inst.ExpireStaleInstanceBinlogCoordinates()
                    go process.ExpireNodesHistory()
                    go process.ExpireAccessTokens()
                    go process.ExpireAvailableNodes()
                    go ExpireFailureDetectionHistory()
                    go ExpireTopologyRecoveryHistory()
                    go ExpireTopologyRecoveryStepsHistory()

                    if runCheckAndRecoverOperationsTimeRipe() && IsLeader() {
                        go SubmitMastersToKvStores("", false)
                    }
                } else {
                    // Take this opportunity to refresh yourself
                    go inst.LoadHostnameResolveCache()
                }
            }()
```
从方法名字可以看出来, 就是做一些"护理"工作, 如:
- 清理unseened instance, 详见参数: [UnseenInstanceForgetHours](https://github.com/Fanduzi/orchestrator-zh-doc/blob/91f843197ef5c70d6a0f001b8fa9e0c7d7d30acf/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-%E2%85%A0.md#unseeninstanceforgethours)
- 清理过期审计日志

### raft护理工作
```go
func ContinuousDiscovery() {
    raftCaretakingTick := time.Tick(10 * time.Minute)
    ...省略部分代码
    for {
        select {
        ...省略部分代码
        case <-raftCaretakingTick:
            if orcraft.IsRaftEnabled() && orcraft.IsLeader() {
            // publishDiscoverMasters will publish to raft a discovery request for all known masters.
            // This makes for a best-effort keep-in-sync between raft nodes, where some may have  
            // inconsistent data due to hosts being forgotten, for example.
                go publishDiscoverMasters()
            }
```
如果orchestrator是[[raft模式部署]]的, 并且本节点是leader, 那么leader会发布一个discovery request给每个raft节点, 这些节点会对所有MySQL主库进行discover. 这么做的目的是保持所有raft nodes的数据"同步"
> [A visual example](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Deployment/raft%E6%A8%A1%E5%BC%8F%E9%83%A8%E7%BD%B2.md#a-visual-example)
>
> ![image](https://raw.githubusercontent.com/openark/orchestrator/master/docs/images/orchestrator-deployment-raft.png)
>
> 如上图所示, 三个`orchestrator` 组成一个raft cluster, 每个`orchestrator` 节点使用自己的专用数据库(`MySQL`或`SQLite`)
> - `orchestrator` 节点之间会进行通信.
> - 只有一个`orchestrator` 节点会成为leader.
> - 所有`orchestrator`节点探测整个`MySQL`舰队. 每个`MySQL` server都被每个raft成员探测.

### 保存拓扑快照
如果[SnapshotTopologiesIntervalHours](https://github.com/Fanduzi/orchestrator-zh-doc/blob/91f843197ef5c70d6a0f001b8fa9e0c7d7d30acf/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-%E2%85%A0.md#snapshottopologiesintervalhours)值大于0, 那么会每[SnapshotTopologiesIntervalHours](https://github.com/Fanduzi/orchestrator-zh-doc/blob/91f843197ef5c70d6a0f001b8fa9e0c7d7d30acf/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-%E2%85%A0.md#snapshottopologiesintervalhours)小时保存database_instance到database_instance_topology_history
```go
func ContinuousDiscovery() {
    ...省略部分代码
    var snapshotTopologiesTick <-chan time.Time
    if config.Config.SnapshotTopologiesIntervalHours > 0 {  
       snapshotTopologiesTick = time.Tick(time.Duration(config.Config.SnapshotTopologiesIntervalHours) * time.Hour)  
    }
    ...省略部分代码
    for {
        select {
        ..省略部分代码
        case <-snapshotTopologiesTick:
            go func() {
                if IsLeaderOrActive() {
                    go inst.SnapshotTopologies()
                }
            }()
        }
```
就是执行insert ignore into database_instance_topology_history select x from database_instance
```go
// SnapshotTopologies records topology graph for all existing topologies
func SnapshotTopologies() error {
    writeFunc := func() error {
        _, err := db.ExecOrchestrator(`
            insert ignore into
                database_instance_topology_history (snapshot_unix_timestamp,
                    hostname, port, master_host, master_port, cluster_name, version)
            select
                UNIX_TIMESTAMP(NOW()),
                hostname, port, master_host, master_port, cluster_name, version
            from
                database_instance
                `,
        )
        if err != nil {
            return log.Errore(err)
        }

        return nil
    }
    return ExecDBWriteFunc(writeFunc)
}
```


### recover工作
```go
const RecoveryPollSeconds = 1

func ContinuousDiscovery() {
    ...省略部分代码
    continuousDiscoveryStartTime := time.Now()  
    checkAndRecoverWaitPeriod := 3 * instancePollSecondsDuration()
    ...省略部分代码
    runCheckAndRecoverOperationsTimeRipe := func() bool {  
       return time.Since(continuousDiscoveryStartTime) >= checkAndRecoverWaitPeriod  
    }
    ...省略部分代码
    recoveryTick := time.Tick(time.Duration(config.RecoveryPollSeconds) * time.Second)
    
    for {
        select {
        case <-recoveryTick:
            go func() {
                if IsLeaderOrActive() {
                    go ClearActiveFailureDetections()
                    go ClearActiveRecoveries()
                    go ExpireBlockedRecoveries()
                    go AcknowledgeCrashedRecoveries()
                    go inst.ExpireInstanceAnalysisChangelog()

                    go func() {
                        // This function is non re-entrant (it can only be running once at any point in time)
                        if atomic.CompareAndSwapInt64(&recoveryEntrance, 0, 1) { // 如果返回true, 说明当时没有运行中的恢复任务
                            defer atomic.StoreInt64(&recoveryEntrance, 0) 
                        } else { // 否则直接return
                            return
                        }
                        if runCheckAndRecoverOperationsTimeRipe() { // 从开始运行ContinuousDiscovery至今的时间 > (3 * InstancePollSeconds = 15秒) 才可以运行recover
                            CheckAndRecover(nil, nil, false)
                        } else {
                            log.Debugf("Waiting for %+v seconds to pass before running failure detection/recovery", checkAndRecoverWaitPeriod.Seconds())
                        }
                    }()
                }
            }()
```

从注释`// This function is non re-entrant (it can only be running once at any point in time)` 可以看出, 同一时间只能有一个恢复任务运行
>atomic.CompareAndSwapInt64
>在Go语言中，原子包提供lower-level原子内存，这对实现同步算法很有帮助。 Go语言中的CompareAndSwapInt64()函数用于对int64值执行比较和交换操作。此函数在原子包下定义。在这里，您需要导入“sync/atomic”软件包才能使用这些函数。
>**用法:**
>```
>func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)
>```
>在这里，addr表示地址，old表示int64值，它是从交换操作返回的旧交换值，而new则是int64新值，它将与旧交换值进行交换。
>**返回值：**如果交换完成，则返回true，否则返回false。

恢复机制的入口在CheckAndRecover
#### CheckAndRecover
`CheckAndRecover(nil, nil, false)` 

`CheckAndRecover` 首先调用GetReplicationAnalysis 获取`分析结果replicationAnalysis`(是一个切片). 后者实际是通过查询database_instance表, 查出所有有`问题`的实例信息, 封装成ReplicationAnalysis结构体, 这个结构体中包含实例基础信息(如是否是主库, 是否开启GTID等), `Analysis`(即orc定义故障名称, 详见[Failure detection scenarios 故障检测场景](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#failure-detection-scenarios-%E6%95%85%E9%9A%9C%E6%A3%80%E6%B5%8B%E5%9C%BA%E6%99%AF))和`StructureAnalysis`(即拓扑结构的故障列表, 如`NotEnoughValidSemiSyncReplicasStructureWarning`等)
> 当然, GetReplicationAnalysis有可能返回一个空切片, 即代表当前无任何故障
> 如果GetReplicationAnalysis返回err!=nil, 那么整个CheckAndRecover也会就此退出return error

接着, `CheckAndRecover` 按随机顺序迭代replicationAnalysis, 对每一个analysisEntry开启协程调用executeCheckAndRecoverFunction
```go
    go func() {  
       _, _, err := executeCheckAndRecoverFunction(analysisEntry, candidateInstanceKey, false, skipProcesses)  // 实际参数是 analysisEntry, nil, false, false
       log.Errore(err)  
    }()
```
> executeCheckAndRecoverFunction 函数注释
> // executeCheckAndRecoverFunction will choose the correct check & recovery function based on analysis.
> // It executes the function synchronuously 
> > synchronuously拼写错误, 应为synchronously
>
> 直译: executeCheckAndRecoverFunction将根据分析选择正确的检查和恢复函数。它同步地执行该功能

##### executeCheckAndRecoverFunction
executeCheckAndRecoverFunction 首先调用getCheckAndRecoverFunction, 后者根据analysisEntry.Analysis(即orc定义故障名称, 详见[Failure detection scenarios 故障检测场景](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#failure-detection-scenarios-%E6%95%85%E9%9A%9C%E6%A3%80%E6%B5%8B%E5%9C%BA%E6%99%AF))返回对应的`checkAndRecoverFunction`, 以及一个布尔值`isActionableRecovery`, 这个值会赋值给analysisEntry.IsActionableRecovery 
```go
    checkAndRecoverFunction, isActionableRecovery := getCheckAndRecoverFunction(analysisEntry.Analysis, &analysisEntry.AnalyzedInstanceKey)
```
以本次实验模拟的主库宕机为例, 我们关闭主库后, 主库会先被认为处于[UnreachableMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#unreachablemaster)状态
```go
func getCheckAndRecoverFunction(analysisCode inst.AnalysisCode, analyzedInstanceKey *inst.InstanceKey) (
    checkAndRecoverFunction func(analysisEntry inst.ReplicationAnalysis, candidateInstanceKey *inst.InstanceKey, forceInstanceRecovery bool, skipProcesses bool) (recoveryAttempted bool, topologyRecovery *TopologyRecovery, err error),
    isActionableRecovery bool,
) {
    switch analysisCode {
    ...省略部分代码
    case inst.UnreachableMaster:
        return checkAndRecoverGenericProblem, false
    ...省略部分代码
    }
    // Right now this is mostly causing noise with no clear action.
    // Will revisit this in the future.
    // case inst.AllMasterReplicasStale:
    //   return checkAndRecoverGenericProblem, false

    return nil, false
}
```
于是第一次, 拿到的checkAndRecoverFunction是checkAndRecoverGenericProblem, 这个函数啥也没干, 就是返回fale, nil, nil
```go
// checkAndRecoverGenericProblem is a general-purpose recovery function
func checkAndRecoverGenericProblem(analysisEntry inst.ReplicationAnalysis, candidateInstanceKey *inst.InstanceKey, forceInstanceRecovery bool, skipProcesses bool) (bool, *TopologyRecovery, error) {  
   return false, nil, nil  
}
```

随后会运行runEmergentOperations
```go
    runEmergentOperations(&analysisEntry)
```
以本次实验模拟的主库宕机为例, 我们关闭主库后, 主库会先被认为处于[UnreachableMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#unreachablemaster)状态
```go
func runEmergentOperations(analysisEntry *inst.ReplicationAnalysis) {
    switch analysisEntry.Analysis {
    ...省略部分代码
    case inst.UnreachableMaster:
        go emergentlyReadTopologyInstance(&analysisEntry.AnalyzedInstanceKey, analysisEntry.Analysis)
        go emergentlyReadTopologyInstanceReplicas(&analysisEntry.AnalyzedInstanceKey, analysisEntry.Analysis)
```
 那么按照代码逻辑, 会运行:
 - emergentlyReadTopologyInstance
 - emergentlyReadTopologyInstanceReplicas
这两步实际是去连接数据库实例(主库, 和其所有的从库), 获取实例的各项信息(如从库的复制延迟, IO_THREAD状态等)

> 下面分别展示了上述两个函数的注释:
> ```
> // Force a re-read of a topology instance; this is done because we need to substantiate a suspicion
> // that we may have a failover scenario. we want to speed up reading the complete picture.
> 强制重新读取一个拓扑实例；这样做是因为我们需要证实一个怀疑，即我们可能有一个故障转移的情况。我们希望加快读取完整的图片。
> // Force reading of replicas of given instance. This is because we suspect the instance is dead, and want to speed up
> // detection of replication failure from its replicas.
> 强制读取给定实例的副本。这是因为我们怀疑该实例已经死亡，并希望加快从其副本中检测复制失败。
> 
> 从这是可以看出, UnreachableMaster时, orc会立即触发对主从的探测, 目的是加速整个Failover速度, 而不依赖与周期性持续探测
> ```



接着,  executeCheckAndRecoverFunction运行checkAndRecoverFunction(即checkAndRecoverGenericProblem)
```go
    recoveryAttempted, topologyRecovery, err = checkAndRecoverFunction(analysisEntry, candidateInstanceKey, forceInstanceRecovery, skipProcesses)
    // 实参为 analysisEntry, nil, false, false
```
所以这里是recoveryAttempted, topologyRecovery, err就是false, nil . 
然后代码判断recoveryAttempted为false时, 直接return了
```go
if !recoveryAttempted {  
   return recoveryAttempted, topologyRecovery, err  
}
```
于是至此流程就是

![](https://raw.githubusercontent.com/Fanduzi/Figure_bed/master/img/Downloads/2022/05/14/21-16-05-Jietu20220428-153051-4f053d.jpeg)

那么问题来了, 只要GetReplicationAnalysis分析后一直认为故障处于[UnreachableMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#unreachablemaster)状态, 就会一直处于这个循环. 所以要看一下[DeadMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#deadmaster)的判断逻辑是什么.
在官方文档中是这样定义的:
1.  主库访问失败
2.  所有主库的副本复制失败

这里列出分别列出 [UnreachableMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#unreachablemaster)  和 [DeadMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#deadmaster) 代码判断逻辑
```go
} else if a.IsMaster && !a.LastCheckValid && a.CountValidReplicas == a.CountReplicas && a.CountValidReplicatingReplicas == 0 {  
   a.Analysis = DeadMaster  
   a.Description = "Master cannot be reached by orchestrator and none of its replicas is replicating"  
   //

} else if a.IsMaster && !a.LastCheckValid && !a.LastCheckPartialSuccess && a.CountValidReplicas > 0 && a.CountValidReplicatingReplicas > 0 {  
   // partial success is here to reduce noise  
   a.Analysis = UnreachableMaster  
   a.Description = "Master cannot be reached by orchestrator but it has replicating replicas; possibly a network/host issue"  
   //  
} else if a.IsMaster && !a.LastCheckValid && a.LastCheckPartialSuccess && a.CountReplicasFailingToConnectToMaster > 0 && a.CountValidReplicas > 0 && a.CountValidReplicatingReplicas > 0 {  
   // there's partial success, but also at least one replica is failing to connect to master  
   a.Analysis = UnreachableMaster  
   a.Description = "Master cannot be reached by orchestrator but it has replicating replicas; possibly a network/host issue"  
   //
```

##### database_instance重要列含义解读

首先, 上面的a.IsMaster, a.LastCheckValid等属性是GetReplicationAnalysis通过SQL查询backend db的database_instance等表获取的. 所以要说清楚这些判断条件, 就要理解SQL含义, 要理解SQL含义, 就又要先了解database_instance表中几个列的含义:
- last_checked 无论被探测的实例是否可以连接, 都会更新此列值为Now()
- last_seen  只有实例探测正常, 才会更新此列值为Now()
- last_check_partial_success 如果被探测实例至少可以连接并执行select @@global.hostname.. 则此列值为1
- last_attempted_check 这个列含义比较复杂. 简单来说, 如果 last_attempted_check <= last_checked 那么这目标实例是正常的, 没有遇到连接hang住的问题

##### ReplicationAnalysis 属性含义解读 
**IsMaster** 主库(本身没有主库, 也不是MGR成员)
```go
        /* To be considered a master, traditional async replication must not be present/valid AND the host should either */
        /* not be a replication group member OR be the primary of the replication group */
        MIN(master_instance.last_check_partial_success) as last_check_partial_success,
        MIN(
            (
                master_instance.master_host IN ('', '_')
                OR master_instance.master_port = 0
                OR substr(master_instance.master_host, 1, 2) = '//'
            )
            AND (
                master_instance.replication_group_name = ''
                OR master_instance.replication_group_member_role = 'PRIMARY'
            )
        ) AS is_master,
```
> substr(master_instance.master_host, 1, 2) = '//'的含义详见[DetachLostReplicasAfterMasterFailover](https://github.com/Fanduzi/orchestrator-zh-doc/blob/91f843197ef5c70d6a0f001b8fa9e0c7d7d30acf/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-%E2%85%A1.md#detachlostreplicasaftermasterfailover)

**LastCheckValid** 本实例最近一次探测正常
```go
        MIN(
            master_instance.last_checked <= master_instance.last_seen  
            and master_instance.last_attempted_check <= master_instance.last_seen + interval ? second
        ) = 1 AS is_last_check_valid

interval ? 的实际值是

// ValidSecondsFromSeenToLastAttemptedCheck returns the maximum allowed elapsed time// between last_attempted_check to last_checked before we consider the instance as invalid.  
func ValidSecondsFromSeenToLastAttemptedCheck() uint {  
   return config.Config.InstancePollSeconds + config.Config.ReasonableInstanceCheckSeconds  
}

5 + 1 = 6s

master_instance.last_checked <= master_instance.last_seen 表示主库探测一切正常, 否则表是探测是无法联机数据库或查询出现错误等
master_instance.last_attempted_check <= master_instance.last_seen + 6s  
假设
    last_attempted_check = 10:10
    last_seen = 10:00
那么这种情况表示主库探测有问题, 可能连接hang住了

两个都为true, true and true就是 1. 然后再和1比较. 

```

**LastCheckPartialSuccess** 本实例至少可以连接并执行select @@global.hostname..
```go
MIN(master_instance.last_check_partial_success) as last_check_partial_success,
```

**CountReplicas**  本实例的从库数量(无论死活)
```go
        COUNT(replica_instance.server_id) AS count_replicas,
```


**CountValidReplicas** 本实例正常的从库数量(只表示实例正常, 能连接能查询, 但不一定复制正常)
```go
        IFNULL(
            SUM(
                replica_instance.last_checked <= replica_instance.last_seen
            ),
            0
        ) AS count_valid_replicas,
```

**CountValidReplicatingReplicas** 本实例正常且复制状态正常的从库数量
```go
        IFNULL(
            SUM(
                replica_instance.last_checked <= replica_instance.last_seen
                AND replica_instance.slave_io_running != 0
                AND replica_instance.slave_sql_running != 0
            ),
            0
        ) AS count_valid_replicating_replicas,
```

**CountReplicasFailingToConnectToMaster** 本实例的, 自身正常(可连接可查询), IO线程处于连接异常, SQL线程正常的从库数量
```go
        IFNULL(
            SUM(
                replica_instance.last_checked <= replica_instance.last_seen
                AND replica_instance.slave_io_running = 0
                AND replica_instance.last_io_error like '%%error %%connecting to master%%'
                AND replica_instance.slave_sql_running = 1
            ),
            0
        ) AS count_replicas_failing_to_connect_to_master,

```

##### 再看 [UnreachableMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#unreachablemaster)  和 [DeadMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#deadmaster)
```go
} else if a.IsMaster && !a.LastCheckValid && a.CountValidReplicas == a.CountReplicas && a.CountValidReplicatingReplicas == 0 {  
   a.Analysis = DeadMaster  
   a.Description = "Master cannot be reached by orchestrator and none of its replicas is replicating"  
   //
本实例是主库(本身没有主库, 也不是MGR成员) && 本实例最近一次探测异常 && 本实例正常的从库数量(只表示实例正常, 能连接能查询, 但不一定复制正常) == 本实例的从库数量(无论死活) && 本实例正常且复制状态正常的从库数量为0

} else if a.IsMaster && !a.LastCheckValid && !a.LastCheckPartialSuccess && a.CountValidReplicas > 0 && a.CountValidReplicatingReplicas > 0 {  
   // partial success is here to reduce noise  
   a.Analysis = UnreachableMaster  
   a.Description = "Master cannot be reached by orchestrator but it has replicating replicas; possibly a network/host issue"  
   //  
本实例是主库(本身没有主库, 也不是MGR成员) && 本实例最近一次探测异常 && 本实例无法连接 && 本实例正常的从库数量(只表示实例正常, 能连接能查询, 但不一定复制正常) > 0 && 本实例正常且复制状态正常的从库数量 > 0


} else if a.IsMaster && !a.LastCheckValid && a.LastCheckPartialSuccess && a.CountReplicasFailingToConnectToMaster > 0 && a.CountValidReplicas > 0 && a.CountValidReplicatingReplicas > 0 {  
   // there's partial success, but also at least one replica is failing to connect to master  
   a.Analysis = UnreachableMaster  
   a.Description = "Master cannot be reached by orchestrator but it has replicating replicas; possibly a network/host issue"  
   //
本实例是主库(本身没有主库, 也不是MGR成员) && 本实例最近一次探测异常 && 本实例无法连接 && 本实例的, 自身正常(可连接可查询), IO线程处于连接异常, SQL线程正常的从库数量 > 0 && 本实例正常且复制状态正常的从库数量 > 0

```
由此可以看出, Master宕机后的最初一段时间内, orchestrator已经无法连接Master, 但是并非所有从库都意识到主库已经宕机, IO线程可能还是处于RUNNING状态(这与[`slave_net_timeout`](https://dev.mysql.com/doc/refman/5.7/en/replication-options-replica.html#sysvar_slave_net_timeout)有关, [Orchestrator官方文档也有描述](https://github.com/Fanduzi/orchestrator-zh-doc/blob/91f843197ef5c70d6a0f001b8fa9e0c7d7d30acf/Setup/%E9%85%8D%E7%BD%AE/Configuration%20%20Failure%20detection.md#mysql-configuration)) . 所以在这段时间内, orchestrator认为Master处于[UnreachableMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#unreachablemaster) 状态, 通过getCheckAndRecoverFunction获取的就永远是checkAndRecoverGenericProblem(也就是啥都不干, 直接return)

直到所有从库复制状态都出现异常, orchestrator才会认为Master处于[DeadMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#deadmaster)状态. 那么此后getCheckAndRecoverFunction会返回checkAndRecoverDeadMaster, 而这才是Failover的真正开始, 欲知后事如何, 请看[Orchestrator Failover过程源码分析-II](http://fuxkdb.com/2022/04/30/2022-05-06-Orchestrator-Failover%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-II/)



## 初步流程总结

![](https://raw.githubusercontent.com/Fanduzi/Figure_bed/master/img/Downloads/2022/05/14/21-16-50-Orchestrator%20Failover%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-I-Excalidraw-6235a2.png)