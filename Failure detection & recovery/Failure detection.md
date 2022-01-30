# Failure detection
# [Failure detection](https://github.com/openark/orchestrator/blob/master/docs/failure-detection.md#masterwithtoomanysemisyncreplicas)
`orchestrator`使用整体方法来检测master和intermediate master的故障.

> `orchestrator` uses a holistic approach to detect master and intermediate master failures.

例如, 在一种简单的方法中, 监控工具会探测主库状态, 并在它无法连接主库或在主库执行查询时发出警报. 这种方法容易受到由网络故障引起的误报的影响.  这种方法通过运行以 `t` 时间间隔的 `n` 个测试来进一步减少误报.  在某些情况下, 这会减少误报的可能, 但在真正失败的情况下增加响应时间.

`orchestrator`利用了复制拓扑结构. 它不仅观察(被监控的)server本身, 而且还观察其replicas. 例如, 为了诊断一个dead master scenario, `orchestrator`必须同时

* 未能与上述主库联系
* 能够联系 master 的副本, 并确认他们也看不到 master. (是联系部分还是所有副本, 还需确认)

`orchestrator`不是按时间对错误进行分类, 而是由多个观察者、复制拓扑服务器本身进行分类.  事实上, 当一个 master 的所有副本都同意他们不能联系他们的 master 时, 复制拓扑实际上被破坏了, 并且故障转移是合理的.

众所周知, `orchestrator` 的整体故障检测方法在生产中非常可靠.

### Detection and recovery 探测和恢复
检测并不总是导致[[Topology recovery id=&#39;963e0044-a9d3-4110-9731-3a736cf82441&#39;]]. 有些情况下, 恢复是不可取的

> Detection does not always lead to [recovery](https://github.com/openark/orchestrator/blob/master/docs/topology-recovery.md). There are scenarios where a recovery is undesired:

* 该集群没有被列入自动失效器.
> The cluster is not listed for auto-failovers.
* admin用户表示不应该在特定的服务器上进行恢复.
> The admin user has indicated a recovery should not take place on the specific server.
* admin用户已全局禁用恢复功能.
> The admin user has globally disabled recoveries.
* 前不久刚刚对该集群完成了恢复, 并且正处于anti-flapping状态
> A previous recovery on same cluster completed shortly before, and anti-flapping takes place.
* 该故障类型被认为不值得恢复.
> The failure type is not deemed worthy of recovery.

在理想的情况下, 检测到故障后立即恢复. 在其他情况下, such as blocked recoveries, 恢复可能在检测后的许多分钟后进行.

检测是独立于恢复的, 并且总是被启用.` OnFailureDetectionProcesses`钩子在每次检测时执行, 详见[[Configuration: Failure detection id=&#39;4124be88-14f6-4c3a-87c8-f4bb76e0b692&#39;]]

### Failure detection scenarios 故障检测场景
请注意以下潜在故障列表:

* DeadMaster
* DeadMasterAndReplicas
* DeadMasterAndSomeReplicas
* DeadMasterWithoutReplicas
* UnreachableMasterWithLaggingReplicas
* UnreachableMaster
* LockedSemiSyncMaster
* MasterWithTooManySemiSyncReplicas
* AllMasterReplicasNotReplicating
* AllMasterReplicasNotReplicatingOrDead
* DeadCoMaster
* DeadCoMasterAndSomeReplicas
* DeadIntermediateMaster
* DeadIntermediateMasterWithSingleReplicaFailingToConnect
* DeadIntermediateMasterWithSingleReplica
* DeadIntermediateMasterAndSomeReplicas
* DeadIntermediateMasterAndReplicas
* AllIntermediateMasterReplicasFailingToConnectOrDead
* AllIntermediateMasterReplicasNotReplicating
* UnreachableIntermediateMasterWithLaggingReplicas
* UnreachableIntermediateMaster
* BinlogServerFailingToConnectToMaster

简单地看一下一些例子, 以下是`orchestrator`如何得出失败的结论:

> Briefly looking at some examples, here is how `orchestrator` reaches failure conclusions:

#### `DeadMaster`:
1. 主库访问失败
> Master MySQL access failure
2. 所有主库的副本复制失败
> All of master's replicas are failing replication

这就形成了一个潜在的恢复过程

> This makes for a potential recovery process

#### `DeadMasterAndSomeReplicas`:
1. 主库访问失败
> Master MySQL access failure
2. 该主库的一些副本也是unreachable的
> Some of its replicas are also unreachable
3. 该主库的其余副本处于复制失败状态
> Rest of the replicas are failing replication

这就形成了一个潜在的恢复过程

> This makes for a potential recovery process

#### `UnreachableMaster`:
1. 主库访问失败
> Master MySQL access failure
2. 但它有复制状态正常的副本
> But it has replicating replicas.

这并不会导致恢复过程. 然而, 为了改进分析, `orchestrator`将发出对副本(从库)的紧急重读, 以弄清它们是否真的对主站满意（在这种情况下，也许`orchestrator`由于网络故障而无法看到它）, 还是实际上正在花时间弄清楚他们的复制失败了.

> This does not make for a recovery process. However, to improve analysis, `orchestrator` will issue an emergent re-read of the replicas, to figure out whether they are really happy with the master (in which case maybe `orchestrator` cannot see it due to a network glitch) or were actually taking their time to figure out they were failing replication.

#### `DeadIntermediateMaster`:
1. intermediate master 无法访问
> An intermediate master (replica with replicas) cannot be reached
2. 该intermediate master所有的副本复制失败
> All of its replicas are failing replication

这就形成了一个潜在的恢复过程

> This makes for a potential recovery process.

#### `UnreachableMasterWithLaggingReplicas`:
1. 主库无法访问
> Master cannot be reached
2. 所有其直属从库(除延迟从库)都是滞后的(复制延迟)
> All of its immediate replicas (excluding SQL delayed) are lagging

当主服务器过载时, 可能会发生这种情况.  客户端会收到"Too many connections"错误, 而很久以前就连接的副本却会声称主库没有问题(因为io\_thread是长连接). 类似地, 如果 master 由于某些元数据操作而被锁定, 则客户端将在连接时被阻塞, 而副本可能会声称一切正常. 但是,  由于应用程序无法连接到主库, 因此不会写入任何实际数据, 并且当使用诸如 `pt-heartbeat` 之类的心跳机制时, 我们可以观察到副本的延迟越来越大.

`orchestrator`对这种情况的反应是重新启动主库的所有直属从库(猜测是restart slave io\_thread). 这将关闭这些副本上的旧客户端连接, 并试图启动新的连接. 现在, 这些连接可能会失败, 导致所有从库复制失败(复制状态异常). 这将导致`orchestrator`分析出一个`DeadMaster` (This will next lead `orchestrator` to analyze a `DeadMaster`.).

> 貌似`DeadMaster` 会触发failover, 这与MHA机制不符. Too many connections 不会导致MHA触发failover.

#### `LockedSemiSyncMaster`
1. 主库启用了半同步(`rpl_semi_sync_master_enabled=1`)
> Master is running with semi-sync enabled (`rpl_semi_sync_master_enabled=1`)
2. 返回ack的从库数量小于`rpl_semi_sync_master_wait_for_slave_count`
> Number of connected semi-sync replicas falls short of expected `rpl_semi_sync_master_wait_for_slave_count` 
3. `rpl_semi_sync_master_timeout` 的值足够高, 主库的写入会被阻塞, 不会退化到异步复制.
> `rpl_semi_sync_master_timeout` is high enough such that master locks writes and does not fall back to asynchronous replication

这个条件只在`ReasonableLockedSemiSyncMasterSeconds`过后触发. 如果没有设置`ReasonableLockedSemiSyncMasterSeconds`, 则在`ReasonableReplicationLagSeconds`之后触发.

这种情况的补救措施可以是在主库上禁用半同步, 或者启动（或启用）足够的半同步副本.

如果启用了 `EnforceExactSemiSyncReplicas`, 那么 `orchestrator` 将确定所需的半同步拓扑并enable/disable副本上的半同步(参数)以匹配它. 所需的拓扑由优先级顺序（见下文）和`rpl_semi_sync_master_wait_for_slave_count`值定义的.

如果启用了 `RecoverLockedSemiSyncMaster`, 那么 `orchestrator` 将按优先级顺序在副本上启用（但永远不会禁用）半同步, 直到半同步副本的数量与`rpl_semi_sync_master_wait_for_slave_count`匹配. 请注意, 如果设置了 `EnforceExactSemiSyncReplicas`, 则 `RecoverLockedSemiSyncMaster` 无效.

优先级顺序由 `DetectSemiSyncEnforcedQuery`（数字越大优先级越高）、提升规则 ([[DetectPromotionRuleQuery id=112feef8-f2b2-4612-856d-fb4f76ab2a73]]) 和主机名（fallback）定义.

如果 `EnforceExactSemiSyncReplicas` 和 `RecoverLockedSemiSyncMaster` 均已禁用（默认）, 则 `orchestrator` 不会为此类分析调用任何恢复过程.

另请参阅[[Semi-sync topology id=b169c092-dcf3-4fe9-87ce-8cbe95b331e1]]文档以获取更多详细信息.

#### `MasterWithTooManySemiSyncReplicas`
1. 主库启用了半同步(rpl\_semi\_sync\_master\_enabled=1)
> Master is running with semi-sync enabled (`rpl_semi_sync_master_enabled=1`)
2. 返回ack的从库数量大于rpl\_semi\_sync\_master\_wait\_for\_slave\_count
> Number of connected semi-sync replicas is higher than the expected `rpl_semi_sync_master_wait_for_slave_count`
3. 启用`EnforceExactSemiSyncReplicas`（如果该标志未被启用, 则不会触发该分析）
> `EnforceExactSemiSyncReplicas` is enabled (this analysis is not triggered if this flag is not enabled)

如果启用了 `EnforceExactSemiSyncReplicas`, 那么 `orchestrator` 将确定所需的半同步拓扑并启用/禁用副本上的半同步(参数)以匹配它. 所需的拓扑由优先级顺序和`rpl_semi_sync_master_timeout`定义.

优先级顺序由 `DetectSemiSyncEnforcedQuery`（数字越大优先级越高）、提升规则 ([[DetectPromotionRuleQuery id=112feef8-f2b2-4612-856d-fb4f76ab2a73]]) 和主机名（fallback）定义.

如果`EnforceExactSemiSyncReplicas`被禁用（默认）, `orchestrator`不会为这种类型的分析调用任何恢复进程。

另请参阅[[Semi-sync topology id=b169c092-dcf3-4fe9-87ce-8cbe95b331e1]]文档以获取更多详细信息.

### Failures of no interest
以下场景对 `orchestrator` 来说不感兴趣, 虽然信息和状态可供 `orchestrator` 使用, 但它本身并不识别此类场景为故障; 没有调用检测钩子, 显然也没有尝试恢复:

* 简单的从库复制异常. 例外: 半同步副本导致 `LockedSemiSyncMaster`
> Failure of simple replicas (*leaves* on the replication topology graph) Exception: semi sync replicas causing `LockedSemiSyncMaster`
* 复制延迟, 甚至更严重.
> Replication lags, even severe.

### Visibility 可见性
最新的分析可通过以下方式获得:

> An up-to-date analysis is available via:

* Command line: `orchestrator-client -c replication-analysis` or `orchestrator -c replication-analysis`
* Web API: `/api/replication-analysis`
* Web: `/web/clusters-analysis/` page (`Clusters`->`Failure analysis`). 这提供了一个不完整的问题列表, 只突出了可操作的问题.

Read next: [[Topology recovery id=&#39;963e0044-a9d3-4110-9731-3a736cf82441&#39;]]