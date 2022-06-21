# [Orchestrator Failover过程源码分析-II](http://fuxkdb.com/2022/04/30/2022-05-06-Orchestrator-Failover%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-II/)

书接上文[Orchestrator Failover过程源码分析-I](http://fuxkdb.com/2022/04/28/2022-05-11-Orchestrator-Failover%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-I/)

## DeadMaster恢复流程
首先通过`getCheckAndRecoverFunction`获取"checkAndRecoverFunction"
```go
func getCheckAndRecoverFunction(analysisCode inst.AnalysisCode, analyzedInstanceKey *inst.InstanceKey) (
    checkAndRecoverFunction func(analysisEntry inst.ReplicationAnalysis, candidateInstanceKey *inst.InstanceKey, forceInstanceRecovery bool, skipProcesses bool) (recoveryAttempted bool, topologyRecovery *TopologyRecovery, err error),
    isActionableRecovery bool,
) {
    switch analysisCode {
    // master
    case inst.DeadMaster, inst.DeadMasterAndSomeReplicas: // 如果analysisCode是DeadMaster 或 DeadMasterAndSomeReplicas
        if isInEmergencyOperationGracefulPeriod(analyzedInstanceKey) { // 首先判断是否处于 EmergencyOperationGracefulPeriod
            return checkAndRecoverGenericProblem, false // 如果处于EmergencyOperationGracefulPeriod, 则又相当于啥也没干, 等下一轮recoverTick
        } else {
            return checkAndRecoverDeadMaster, true
        }
```
这里先判断isInEmergencyOperationGracefulPeriod
```go
func isInEmergencyOperationGracefulPeriod(instanceKey *inst.InstanceKey) bool {
    _, found := emergencyOperationGracefulPeriodMap.Get(instanceKey.StringCode()) // emergencyOperationGracefulPeriodMap 是一个cache, 有过期时间的"缓存"
    return found
}
```
实际是尝试去emergencyOperationGracefulPeriodMap这个cache找有没有这个instance. 那么问题来了, 是谁在什么时候向这个cache放这个instance呢?
其实是在主库处于[UnreachableMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#unreachablemaster)状态下, `executeCheckAndRecoverFunction`中调用runEmergentOperations时


```go
// executeCheckAndRecoverFunction will choose the correct check & recovery function based on analysis.
// It executes the function synchronuously
func executeCheckAndRecoverFunction(analysisEntry inst.ReplicationAnalysis, candidateInstanceKey *inst.InstanceKey, forceInstanceRecovery bool, skipProcesses bool) (recoveryAttempted bool, topologyRecovery *TopologyRecovery, err error) {
    atomic.AddInt64(&countPendingRecoveries, 1)
    defer atomic.AddInt64(&countPendingRecoveries, -1)

    checkAndRecoverFunction, isActionableRecovery := getCheckAndRecoverFunction(analysisEntry.Analysis, &analysisEntry.AnalyzedInstanceKey) // 最初处于UnreachableMaster时, 这里拿到的是checkAndRecoverGenericProblem
    analysisEntry.IsActionableRecovery = isActionableRecovery
    runEmergentOperations(&analysisEntry) 
```
可以看到, [UnreachableMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#unreachablemaster)时, 会运行emergentlyReadTopologyInstance
```go
func runEmergentOperations(analysisEntry *inst.ReplicationAnalysis) {
    switch analysisEntry.Analysis {
    ...省略部分代码
    case inst.UnreachableMaster: // 实际目的是强制重新读一下该实例和其所有从库的"信息"
        go emergentlyReadTopologyInstance(&analysisEntry.AnalyzedInstanceKey, analysisEntry.Analysis)
        go emergentlyReadTopologyInstanceReplicas(&analysisEntry.AnalyzedInstanceKey, analysisEntry.Analysis)
```
而在emergentlyReadTopologyInstance中会先尝试向emergencyReadTopologyInstanceMap中Add
```go
// Force a re-read of a topology instance; this is done because we need to substantiate a suspicion
// that we may have a failover scenario. we want to speed up reading the complete picture.
func emergentlyReadTopologyInstance(instanceKey *inst.InstanceKey, analysisCode inst.AnalysisCode) (instance *inst.Instance, err error) {
    if existsInCacheError := emergencyReadTopologyInstanceMap.Add(instanceKey.StringCode(), true, cache.DefaultExpiration); existsInCacheError != nil {
        // Just recently attempted
        return nil, nil
    }
    instance, err = inst.ReadTopologyInstance(instanceKey) // 会调用 ReadTopologyInstanceBufferable
    inst.AuditOperation("emergently-read-topology-instance", instanceKey, string(analysisCode))
    return instance, err
}
```
> ReadTopologyInstance 实际调用 [ReadTopologyInstanceBufferable](http://fuxkdb.com/2022/04/26/2022-04-26-Orchestrator-Discover%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/#gt-ReadTopologyInstanceBufferable)

这个cache是在logic包init时创建的
```go
emergencyOperationGracefulPeriodMap = cache.New(time.Second*5, time.Millisecond*500) // 过期时间5s
```

isInEmergencyOperationGracefulPeriod 如果返回true, 表示尚处于EmergentOperations运行窗口期内, 所以需要等
若返回false, 表示已经可以开始"恢复"了, 就会返回checkAndRecoverDeadMaster,  并且isActionableRecovery也为true


### executeCheckAndRecoverFunction
```go
// It executes the function synchronuously
func executeCheckAndRecoverFunction(analysisEntry inst.ReplicationAnalysis, candidateInstanceKey *inst.InstanceKey, forceInstanceRecovery bool, skipProcesses bool) (recoveryAttempted bool, topologyRecovery *TopologyRecovery, err error) {
    atomic.AddInt64(&countPendingRecoveries, 1)
    defer atomic.AddInt64(&countPendingRecoveries, -1)

    checkAndRecoverFunction, isActionableRecovery := getCheckAndRecoverFunction(analysisEntry.Analysis, &analysisEntry.AnalyzedInstanceKey)
    analysisEntry.IsActionableRecovery = isActionableRecovery
    runEmergentOperations(&analysisEntry)

    ...省略部分代码
    // Initiate detection:
    // 向backend db topology_failure_detection表insert ignore into一条数据
    // 这里 skipProcesses = false, 所以会调用 OnFailureDetectionProcesses 钩子
    // 到这里我才明白, 原来 skipProcesses 意思是 是否跳过 钩子的执行
    registrationSuccess, _, err := checkAndExecuteFailureDetectionProcesses(analysisEntry, skipProcesses)
    if registrationSuccess {
        if orcraft.IsRaftEnabled() {
            _, err := orcraft.PublishCommand("register-failure-detection", analysisEntry)
            log.Errore(err)
        }
    }
    ...省略部分代码
    // We don't mind whether detection really executed the processes or not
    // (it may have been silenced due to previous detection). We only care there's no error.

    // We're about to embark on recovery shortly...

    // Check for recovery being disabled globally
    // 看是否全局禁用了恢复, 如通过API disable-global-recoveries 去禁用全局恢复
    if recoveryDisabledGlobally, err := IsRecoveryDisabled(); err != nil {
        // Unexpected. Shouldn't get this
        log.Errorf("Unable to determine if recovery is disabled globally: %v", err)
    } else if recoveryDisabledGlobally {
        // 这里checkAndRecover传的是false
        if !forceInstanceRecovery { // 所以如果全局禁用了recover, recoverTick是不会进行恢复的, 因为forceInstanceRecovery也是false. 至此退出
            log.Infof("CheckAndRecover: Analysis: %+v, InstanceKey: %+v, candidateInstanceKey: %+v, "+
                "skipProcesses: %v: NOT Recovering host (disabled globally)",
                analysisEntry.Analysis, analysisEntry.AnalyzedInstanceKey, candidateInstanceKey, skipProcesses)

            return false, nil, err
        }
        log.Infof("CheckAndRecover: Analysis: %+v, InstanceKey: %+v, candidateInstanceKey: %+v, "+
            "skipProcesses: %v: recoveries disabled globally but forcing this recovery",
            analysisEntry.Analysis, analysisEntry.AnalyzedInstanceKey, candidateInstanceKey, skipProcesses)
    }

    // 开始运行checkAndRecoverFunction. 本例中是运行checkAndRecoverDeadMaster
    // 我们先不看checkAndRecoverDeadMaster具体干了啥, 先往下看
    recoveryAttempted, topologyRecovery, err = checkAndRecoverFunction(analysisEntry, candidateInstanceKey, forceInstanceRecovery, skipProcesses) // 实参为analysisEntry, nil, false, false
    if !recoveryAttempted {
        return recoveryAttempted, topologyRecovery, err
    }
    if topologyRecovery == nil {
        return recoveryAttempted, topologyRecovery, err
    }
    if b, err := json.Marshal(topologyRecovery); err == nil {
        log.Infof("Topology recovery: %+v", string(b))
    } else {
        log.Infof("Topology recovery: %+v", *topologyRecovery)
    }
    // skipProcesses=false
    if !skipProcesses {
        // 没恢复成功, 调用 PostUnsuccessfulFailoverProcesses
        if topologyRecovery.SuccessorKey == nil {
            // Execute general unsuccessful post failover processes
            executeProcesses(config.Config.PostUnsuccessfulFailoverProcesses, "PostUnsuccessfulFailoverProcesses", topologyRecovery, false)
        } else {
            // 否则调用 PostFailoverProcesses
            // Execute general post failover processes
            inst.EndDowntime(topologyRecovery.SuccessorKey)
            executeProcesses(config.Config.PostFailoverProcesses, "PostFailoverProcesses", topologyRecovery, false)
        }
    }
  
    ...省略部分代码
    // 代码只要能走到这里, 就带表恢复成功了? recoveryAttempted肯定是true
    // 具体的恢复操作, 都在checkAndRecoverFunction里
    return recoveryAttempted, topologyRecovery, err
}
```
对于本文场景, 当主库被认定处于[DeadMaster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#deadmaster)状态后:
executeCheckAndRecoverFunction会先调用getCheckAndRecoverFunction获取:
- checkAndRecoverFunction: checkAndRecoverDeadMaster
- isActionableRecovery: true

然后检查是否"禁用"了全局故障恢复功能(如通过API [disable-global-recoveries](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Topology%20recovery.md#web-api-command-line)), 如果是, 则直接return

接下来, 开始执行checkAndRecoverFunction, 即checkAndRecoverDeadMaster

我们先不看checkAndRecoverDeadMaster具体逻辑, 继续往下看. 下面的代码就是根据checkAndRecoverFunction的执行情况(是否成功恢复), 调用对应的钩子, 最后将恢复结果return回去, 其实到这里, 本轮恢复就算做完了. 具体恢复是否成功, recoverTick也不关心. 具体如何恢复的, 还是要看checkAndRecoverFunction(本例中是checkAndRecoverDeadMaster)
> 关于钩子, 见:
> [OnFailureDetectionProcesses](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-%E2%85%A1.md#onfailuredetectionprocesses)
> [PostUnsuccessfulFailoverProcesses](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-%E2%85%A1.md#postunsuccessfulfailoverprocesses)
> [PostFailoverProcesses](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-%E2%85%A1.md#postfailoverprocesses)


### checkAndRecoverDeadMaster

```go
// checkAndRecoverDeadMaster checks a given analysis, decides whether to take action, and possibly takes action
// Returns true when action was taken.
func checkAndRecoverDeadMaster(analysisEntry inst.ReplicationAnalysis, candidateInstanceKey *inst.InstanceKey, forceInstanceRecovery bool, skipProcesses bool) (recoveryAttempted bool, topologyRecovery *TopologyRecovery, err error) {
    // forceInstanceRecovery=false
    // HasAutomatedMasterRecovery 是在GetReplicationAnalysis中会运行 a.ClusterDetails.ReadRecoveryInfo()
    // ReadRecoveryInfo中会执行 this.HasAutomatedMasterRecovery = this.filtersMatchCluster(config.Config.RecoverMasterClusterFilters)
    // RecoverMasterClusterFilters 我们配置的是 .* 所以这里为true
    // false || true = true
    // 所以这对于recoverTick来说, 目的是检查一下RecoverMasterClusterFilters配置, 看这个集群到底配没配自动故障恢复
    if !(forceInstanceRecovery || analysisEntry.ClusterDetails.HasAutomatedMasterRecovery) {
        return false, nil, nil
    }
    // 实参 &analysisEntry, true, true
    // 尝试注册一个恢复条目, 如果失败, 意味着 recovery is already in place.

    // 让我们检查一下这个实例是否最近刚刚被提升或集群刚刚经历failover，并且仍然处于活动期。如果是这样，我们就拒绝恢复注册，to avoid flapping. 
    // 就是说刚Failover没多久又Failover不行, 类似MHA 8小时限制
    // 时间由RecoveryPeriodBlockMinutes控制, 默认1小时. 也是在recoveryTick启动协程是会去清理in_active_period标记(update为0)
    // 如果一切没问题, 会向数据库topology_recovery插入一条记录, 并new一个topologyRecovery结构体返回.
    // 否则 topologyRecovery 是 nil
    topologyRecovery, err = AttemptRecoveryRegistration(&analysisEntry, !forceInstanceRecovery, !forceInstanceRecovery)
    if topologyRecovery == nil { // 如果 topologyRecovery = nil, 说明最近刚恢复完, 或者正在恢复, 直接return
        AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("found an active or recent recovery on %+v. Will not issue another RecoverDeadMaster.", analysisEntry.AnalyzedInstanceKey))
        return false, nil, err
    }

    // That's it! We must do recovery!
    AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("will handle DeadMaster event on %+v", analysisEntry.ClusterDetails.ClusterName))
    recoverDeadMasterCounter.Inc(1)

    // 正式开始恢复
    // 实参为 topologyRecovery, nil, false
    recoveryAttempted, promotedReplica, lostReplicas, err := recoverDeadMaster(topologyRecovery, candidateInstanceKey, skipProcesses)
    ...省略部分代码
    topologyRecovery.LostReplicas.AddInstances(lostReplicas) // lostReplicas 由于种种原因, 没有成功change到new master的从库
    if !recoveryAttempted {
        return false, topologyRecovery, err
    }
    
    // 代码运行到这里时, 可能已经选举出了new master, 即promotedReplica
    // 这个函数补充了一些判断条件, 判断这个promotedReplica是不是可以做new master
    // 主要就是这几个参数:
    // - PreventCrossDataCenterMasterFailover
    // - PreventCrossRegionMasterFailover
    // - FailMasterPromotionOnLagMinutes
    // - FailMasterPromotionIfSQLThreadNotUpToDate
    // - DelayMasterPromotionIfSQLThreadNotUpToDate
    overrideMasterPromotion := func() (*inst.Instance, error) {
        if promotedReplica == nil {
            // No promotion; nothing to override.
            return promotedReplica, err
        }
        // Scenarios where we might cancel the promotion.
        // 这个函数就是通过以下参数:
        // - PreventCrossDataCenterMasterFailover
        // - PreventCrossRegionMasterFailover
        // 判断 promotedReplica 是否可以做 new master
        // PreventCrossDataCenterMasterFailover:
        // 默认false . 当为true 时, orchestrator将只用与故障集群主库位于同一DC的从库替换故障的主库. 它将尽最大努力从同一DC中找到一个替代者, 如果找不到, 将中止（失败）故障转移.
        // PreventCrossRegionMasterFailover:
        // 默认false . 当为true 时, orchestrator将只用与故障集群主库位于同一region的从库替换故障的主库. 它将尽最大努力找到同一region的替代者, 如果找不到, 将中止（失败）故障转移.
        if satisfied, reason := MasterFailoverGeographicConstraintSatisfied(&analysisEntry, promotedReplica); !satisfied {
            return nil, fmt.Errorf("RecoverDeadMaster: failed %+v promotion; %s", promotedReplica.Key, reason)
        }
        AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("RecoverDeadMaster: promoted replica lag seconds: %+v", promotedReplica.ReplicationLagSeconds.Int64))
        if config.Config.FailMasterPromotionOnLagMinutes > 0 &&
            time.Duration(promotedReplica.ReplicationLagSeconds.Int64)*time.Second >= time.Duration(config.Config.FailMasterPromotionOnLagMinutes)*time.Minute {
            // candidate replica lags too much
            return nil, fmt.Errorf("RecoverDeadMaster: failed promotion. FailMasterPromotionOnLagMinutes is set to %d (minutes) and promoted replica %+v 's lag is %d (seconds)", config.Config.FailMasterPromotionOnLagMinutes, promotedReplica.Key, promotedReplica.ReplicationLagSeconds.Int64)
        }
        AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("RecoverDeadMaster: promoted replica sql thread up-to-date: %+v", promotedReplica.SQLThreadUpToDate()))
        if config.Config.FailMasterPromotionIfSQLThreadNotUpToDate && !promotedReplica.SQLThreadUpToDate() {
            return nil, fmt.Errorf("RecoverDeadMaster: failed promotion. FailMasterPromotionIfSQLThreadNotUpToDate is set and promoted replica %+v 's sql thread is not up to date (relay logs still unapplied). Aborting promotion", promotedReplica.Key)
        }
        if config.Config.DelayMasterPromotionIfSQLThreadNotUpToDate && !promotedReplica.SQLThreadUpToDate() {
            AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("DelayMasterPromotionIfSQLThreadNotUpToDate: waiting for SQL thread on %+v", promotedReplica.Key))
            if _, err := inst.WaitForSQLThreadUpToDate(&promotedReplica.Key, 0, 0); err != nil {
                return nil, fmt.Errorf("DelayMasterPromotionIfSQLThreadNotUpToDate error: %+v", err)
            }
            AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("DelayMasterPromotionIfSQLThreadNotUpToDate: SQL thread caught up on %+v", promotedReplica.Key))
        }
        // All seems well. No override done.
        AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("RecoverDeadMaster: found no reason to override promotion of %+v", promotedReplica.Key))
        return promotedReplica, err
    }
    // 运行 overrideMasterPromotion
    if promotedReplica, err = overrideMasterPromotion(); err != nil {
        AuditTopologyRecovery(topologyRecovery, err.Error())
    }
    // And this is the end; whether successful or not, we're done.
    resolveRecovery(topologyRecovery, promotedReplica)
    // Now, see whether we are successful or not. From this point there's no going back.
    
    if promotedReplica != nil { // 成功选举除了新主库
        // Success!
        recoverDeadMasterSuccessCounter.Inc(1)
        AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("RecoverDeadMaster: successfully promoted %+v", promotedReplica.Key))
        AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("- RecoverDeadMaster: promoted server coordinates: %+v", promotedReplica.SelfBinlogCoordinates))

        // 当为true 时, orchestrator 将在选举出的新主库上执行reset slave all 和set read_only=0 . 默认: true . 当该参数为true 时, 将覆盖MasterFailoverDetachSlaveMasterHost .
        // 所以这个if列操作就是执行在新主库执行 reset slave all 和set read_only=0
        if config.Config.ApplyMySQLPromotionAfterMasterFailover || analysisEntry.CommandHint == inst.GracefulMasterTakeoverCommandHint {
            // on GracefulMasterTakeoverCommandHint it makes utter sense to RESET SLAVE ALL and read_only=0, and there is no sense in not doing so.
            AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("- RecoverDeadMaster: will apply MySQL changes to promoted master"))
            {
                _, err := inst.ResetReplicationOperation(&promotedReplica.Key)
                if err != nil {
                    // Ugly, but this is important. Let's give it another try
                    _, err = inst.ResetReplicationOperation(&promotedReplica.Key)
                }
                AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("- RecoverDeadMaster: applying RESET SLAVE ALL on promoted master: success=%t", (err == nil)))
                if err != nil {
                    AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("- RecoverDeadMaster: NOTE that %+v is promoted even though SHOW SLAVE STATUS may still show it has a master", promotedReplica.Key))
                }
            }
            {
                _, err := inst.SetReadOnly(&promotedReplica.Key, false)
                AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("- RecoverDeadMaster: applying read-only=0 on promoted master: success=%t", (err == nil)))
            }
            // Let's attempt, though we won't necessarily succeed, to set old master as read-only
            go func() {
            // 并且尝试给旧主库打开只读, 成功与否无所
                _, err := inst.SetReadOnly(&analysisEntry.AnalyzedInstanceKey, true)
                AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("- RecoverDeadMaster: applying read-only=1 on demoted master: success=%t", (err == nil)))
            }()
        }

        ...省略部分代码
        // 当该参数为true 时, orchestrator将对被选举为新主库的节点执行detach-replica-master-host（这确保了即使旧主库"复活了", 新主库也不会试图从旧主库复制数据）. 默认值: false. 如果ApplyMySQLPromotionAfterMasterFailover为真，这个参数将失去意义.
        if config.Config.MasterFailoverDetachReplicaMasterHost {
            postponedFunction := func() error {
                AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("- RecoverDeadMaster: detaching master host on promoted master"))
                // 所以说 如果 ApplyMySQLPromotionAfterMasterFailover=true, 就已经完成了reset slave all了. 
                // 那么DetachReplicaMasterHost也就失去意义了
                inst.DetachReplicaMasterHost(&promotedReplica.Key)
                return nil
            }
            topologyRecovery.AddPostponedFunction(postponedFunction, fmt.Sprintf("RecoverDeadMaster, detaching promoted master host %+v", promotedReplica.Key))
        }
        ...省略部分代码

        if !skipProcesses {
            // Execute post master-failover processes
            // 执行钩子 PostMasterFailoverProcesses
            executeProcesses(config.Config.PostMasterFailoverProcesses, "PostMasterFailoverProcesses", topologyRecovery, false)
        }
    } else {
        // 否则, 说明恢复失败了
        recoverDeadMasterFailureCounter.Inc(1)
    }

    return true, topologyRecovery, err
}
```
checkAndRecoverDeadMaster 首先通过[RecoverMasterClusterFilters](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-%E2%85%A1.md#recovermasterclusterfilters)再次判断这个实例是否可以进行自动故障恢复.
接着, 调用AttemptRecoveryRegistration检查这个实例是否最近刚刚被提升或该实例所处集群刚刚经历failover, 并且仍然处于活动期. 
> 就是说刚Failover没多久又Failover不行, 类似MHA 8小时限制

活动期的时间由[RecoveryPeriodBlockMinutes](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-%E2%85%A1.md#recoveryperiodblockminutes)控制, 默认1小时. 也是在recoveryTick启动协程时会去清理in_active_period标记(update为0)
如果一切没问题, 会向数据库topology_recovery插入一条记录, 并new一个topologyRecovery结构体返回. 否则 topologyRecovery 是 nil

以上检查都没问题的话, 就正式开始执行恢复, 调用recoverDeadMaster. 

我们先不看recoverDeadMaster代码, 继续往后看.

recoverDeadMaster最重要的是会返回promotedReplica, 即选举出来的新主库
代码运行到这里时, 可能已经选举出了new master, 即promotedReplica. 随后会运行overrideMasterPromotion函数
这个函数补充了一些判断条件, 判断这个promotedReplica是不是可以做new master
主要就是这几个参数:
- [PreventCrossDataCenterMasterFailover](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-III.md#preventcrossdatacentermasterfailover)
- [PreventCrossRegionMasterFailover](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-III.md#preventcrossregionmasterfailover)
- [FailMasterPromotionOnLagMinutes](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-III.md#failmasterpromotiononlagminutes)
- [FailMasterPromotionIfSQLThreadNotUpToDate](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-III.md#failmasterpromotionifsqlthreadnotuptodate)
- [DelayMasterPromotionIfSQLThreadNotUpToDate](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-III.md#delaymasterpromotionifsqlthreadnotuptodate)

以上任意一个判断有问题, 都会终止后续恢复动作, 直接return

最后, checkAndRecoverDeadMaster会对new master和old master做一些收尾工作, 如:
- 如果[ApplyMySQLPromotionAfterMasterFailover](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-III.md#applymysqlpromotionaftermasterfailover)为true, 则会在新主库执行 reset slave all 和set read_only=0, 否则走[MasterFailoverDetachReplicaMasterHost](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-III.md#masterfailoverdetachreplicamasterhost)逻辑
- 尝试连接旧主库打开只读

至此, 恢复成功完成. 
具体的恢复逻辑, 还要看recoverDeadMaster


### recoverDeadMaster

```go
// recoverDeadMaster recovers a dead master, complete logic inside
func recoverDeadMaster(topologyRecovery *TopologyRecovery, candidateInstanceKey *inst.InstanceKey, skipProcesses bool) (recoveryAttempted bool, promotedReplica *inst.Instance, lostReplicas [](*inst.Instance), err error) {

    // 传进来的实参, topologyRecovery, nil, false

    topologyRecovery.Type = MasterRecovery
    analysisEntry := &topologyRecovery.AnalysisEntry
    failedInstanceKey := &analysisEntry.AnalyzedInstanceKey
    var cannotReplicateReplicas [](*inst.Instance)
    postponedAll := false

    inst.AuditOperation("recover-dead-master", failedInstanceKey, "problem found; will recover")
    if !skipProcesses { // 执行钩子
        if err := executeProcesses(config.Config.PreFailoverProcesses, "PreFailoverProcesses", topologyRecovery, true); err != nil {
            return false, nil, lostReplicas, topologyRecovery.AddError(err)
        }
    }
    ...省略部分代码
    // topologyRecovery.RecoveryType = MasterRecoveryGTID
    topologyRecovery.RecoveryType = GetMasterRecoveryType(analysisEntry)
    ...省略部分代码

    // 这个函数判断GetCandidateReplica返回的candidate是不是"理想的"
    // 如果是"理想的" 一些从库的恢复操作可以异步推迟执行
    // 否则要同步执行
    promotedReplicaIsIdeal := func(promoted *inst.Instance, hasBestPromotionRule bool) bool {
        if promoted == nil {
            return false
        }
        AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("RecoverDeadMaster: promotedReplicaIsIdeal(%+v)", promoted.Key))
        
        // candidateInstanceKey实参是nil, 所以走不到这个if
        if candidateInstanceKey != nil { //explicit request to promote a specific server
            return promoted.Key.Equals(candidateInstanceKey)
        }
        if promoted.DataCenter == topologyRecovery.AnalysisEntry.AnalyzedInstanceDataCenter &&
            promoted.PhysicalEnvironment == topologyRecovery.AnalysisEntry.AnalyzedInstancePhysicalEnvironment {
            if promoted.PromotionRule == inst.MustPromoteRule || promoted.PromotionRule == inst.PreferPromoteRule ||
                (hasBestPromotionRule && promoted.PromotionRule != inst.MustNotPromoteRule) {
                AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("RecoverDeadMaster: found %+v to be ideal candidate; will optimize recovery", promoted.Key))
                postponedAll = true
                return true
            }
        }
        return false
    }
    switch topologyRecovery.RecoveryType {
    case MasterRecoveryGTID:
        {
            AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("RecoverDeadMaster: regrouping replicas via GTID"))
            // promotedReplica 有可能==nil
            lostReplicas, _, cannotReplicateReplicas, promotedReplica, err = inst.RegroupReplicasGTID(failedInstanceKey, true, false, nil, &topologyRecovery.PostponedFunctionsContainer, promotedReplicaIsIdeal)
        }
    // case MasterRecoveryPseudoGTID:
    //  ...省略部分代码
    // case MasterRecoveryBinlogServer:
    //  ...省略部分代码
    }
    topologyRecovery.AddError(err)
    lostReplicas = append(lostReplicas, cannotReplicateReplicas...)
    ...省略部分代码

    if promotedReplica != nil && len(lostReplicas) > 0 && config.Config.DetachLostReplicasAfterMasterFailover {
        postponedFunction := func() error {
            AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("RecoverDeadMaster: lost %+v replicas during recovery process; detaching them", len(lostReplicas)))
            for _, replica := range lostReplicas {
                replica := replica
                inst.DetachReplicaMasterHost(&replica.Key)
            }
            return nil
        }
        // 对lostReplicas并发执行 DetachReplicaMasterHost
        topologyRecovery.AddPostponedFunction(postponedFunction, fmt.Sprintf("RecoverDeadMaster, detach %+v lost replicas", len(lostReplicas)))
    }

    ...省略部分代码

    AuditTopologyRecovery(topologyRecovery, fmt.Sprintf("RecoverDeadMaster: %d postponed functions", topologyRecovery.PostponedFunctionsContainer.Len()))

    if promotedReplica != nil && !postponedAll { // 有候选人, 并且不是理想的候选人
        // 这里的意思是, 我们一开始选了一个new master, 可能因为他有最全的日志, 但是, 他不是我们设置的prefer的实例, 那么我们就重新组织一下拓扑, 把prefer的再变成new master
        promotedReplica, err = replacePromotedReplicaWithCandidate(topologyRecovery, &analysisEntry.AnalyzedInstanceKey, promotedReplica, candidateInstanceKey)
        topologyRecovery.AddError(err)
    }

    if promotedReplica == nil {
        message := "Failure: no replica promoted."
        AuditTopologyRecovery(topologyRecovery, message)
        inst.AuditOperation("recover-dead-master", failedInstanceKey, message)
    } else {
        message := fmt.Sprintf("promoted replica: %+v", promotedReplica.Key)
        AuditTopologyRecovery(topologyRecovery, message)
        inst.AuditOperation("recover-dead-master", failedInstanceKey, message)
    }
    return true, promotedReplica, lostReplicas, err
}

```
recoverDeadMaster 主要做了几件事 ^a0c808
1. GetMasterRecoveryType, 确定到底用什么方式恢复, 是基于GTID? PseudoGTID? 还是BinlogServer?
2. 重组拓扑, 我们的案例是使用RegroupReplicasGTID. 
    但这里有一个问题, 可能现在我们的新主库并不是我们"期望"的实例, 就是说之所以选他做主库可能是因为他有最全的日志. 但不是我们设置的prefer的
    所以通过一个闭包promotedReplicaIsIdeal去做了判断和标记(通过postponedAll)
    如果postponedAll=false, 则需要重组拓扑, 选择prefer的replica作为新主库
    > postponedAll=true表是candidate就是"理想型", 所有replica恢复可以并行异步执行
3. 如果存在lostReplicas, 并且开启了[DetachLostReplicasAfterMasterFailover](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0%E8%AF%A6%E8%A7%A3-%E2%85%A1.md#detachlostreplicasaftermasterfailover), 那么会并行的对所有lostReplicas执行DetachReplicaMasterHost
    > 其实就是执行change master to master_host='// {host}'
4. 如果当前选举的new master不是我们prefer的实例, 重组拓扑, 用prefer做新主库

recoverDeadMaster还是干了挺多事儿的, 尤其是RegroupReplicasGTID中的逻辑还是很复杂的. 本文就先不继续展开了, 要知后事如何, 请看[Orchestrator Failover过程源码分析-III](http://fuxkdb.com/2022/05/05/2022-05-05-Orchestrator-Failover%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-III/)

## 初步流程总结
![](https://raw.githubusercontent.com/Fanduzi/Figure_bed/master/img/Downloads/2022/05/15/16-34-30-Orchestrator%20Failover%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-II-Excalidraw-26a12b.png)