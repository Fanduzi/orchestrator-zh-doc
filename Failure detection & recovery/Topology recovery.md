# Topology recovery
# [Topology recovery](https://github.com/openark/orchestrator/blob/master/docs/topology-recovery.md)
`orchestrator`能够从一系列的故障场景([Failure detection scenarios 故障检测场景](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md#failure-detection-scenarios-%E6%95%85%E9%9A%9C%E6%A3%80%E6%B5%8B%E5%9C%BA%E6%99%AF))中恢复。值得注意的是，它可以恢复a failed master or a failed intermediate master.

`orchestrator` 支持:

* [Automated recovery](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Failure%20detection%20%26%20recovery/Topology%20recovery.md#automated-recovery)(takes action on unexpected failures)
* [Graceful master promotion 在线切换](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Failure%20detection%20%26%20recovery/Topology%20recovery.md#graceful-master-promotion-%E5%9C%A8%E7%BA%BF%E5%88%87%E6%8D%A2)
* [Manual recovery](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Failure%20detection%20%26%20recovery/Topology%20recovery.md#manual-recovery-%E6%89%8B%E5%8A%A8%E6%81%A2%E5%A4%8D)
* [Manual, forced/panic failovers.](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Failure%20detection%20%26%20recovery/Topology%20recovery.md#manual-forced-failover-%E6%89%8B%E5%8A%A8%E5%BC%BA%E5%88%B6failover)

## Requirements
要运行任何类型的故障转移, 您的拓扑必须支持:

* Oracle GTID (with `MASTER_AUTO_POSITION=1`)
* MariaDB GTID
* [Pseudo GTID](Various/Pseudo%20GTID.md)
* Binlog Servers

更多细节见[MySQL Configuration](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Configuration%20%20Recovery.md#mysql-configuration)

自动恢复是可以选的, 请参阅 [Configuration: Recovery](Setup/配置/Configuration%20%20Recovery.md)

### What's in a recovery?
基于[Failure detection](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md), 一连串的事件构成了一个恢复过程:

* 恢复前的钩子(执行外部进程)
> Pre-recovery hooks (external processes execution)
* 拓扑修复
> Healing of topology
* 恢复后的钩子
> Post-recovery hooks

其中:

* 恢复前的钩子由用户配置
> Pre-recovery hooks are configured by the user.
   * 按顺序执行
> Executed sequentially
   * 任何这些钩子的失败（非零退出代码）都将中止故障转移
> Failure of any of these hooks (nonzero exit code) will abort the failover.
* 拓扑修复由`orchestrator`管理并且基于状态(state-based)而不是基于配置(configuration-based). `orchestrator` 试图在考虑到现有的拓扑结构、版本、服务器配置等因素的情况下, 使坏情况得到最好的解决.
> Topology healing is managed by `orchestrator` and is state-based rather than configuration-based. `orchestrator` tries to make the best out of a bad situation, taking into consideration the existing topology, versions, server configurations, etc.
* 恢复后钩子也由用户配置.
> Post-recovery hooks are, again, configured by the user.

### Discussion: recovering a dead intermediate master
下面重点介绍了恢复的一些复杂性.

一个"简单"的恢复案例是 `DeadIntermediateMaster` 的案例. 它的副本是孤立的, 但是当使用 GTID 或 Pseudo-GTID 它们仍然可以重新连接到拓扑. 我们可能会选择:

* 找到dead intermediate master的同级节点, 并将无主的副本移动到该节点下面
> Find a sibling of the dead intermediate master, and move orphaned replicas below said sibling
* 从孤立的副本中提升一个副本, 使其成为其兄弟副本的intermediate master, 然后, 将提升的副本连接到拓扑
> Promote a replica from among the orphaned replicas, make it intermediate master of its siblings, then
connect promoted replica up the topology
* 重新安置所有孤儿副本
> relocate all orphaned replicas
* 将上述部分结合起来
> Combine parts of the above

具体的实现在很大程度上取决于拓扑结构的设置(哪些实例有`log-slave-updates`？实例是否滞后？它们有复制过滤器吗？MySQL版本是？等等）. 您的拓扑很（很）可能至少支持上述之一（特别是, 除非有复制过滤器，否则匹配副本是一个微不足道的解决方案）。

> It is very (very) likely your topology will support at least one of the above (in particular, matching-up the replicas is a trivial solution, unless replication filters are in place). 每太搞懂这个怎么翻译

### Discussion: recovering a dead master
由于各种原因, 恢复一个dead master是一个复杂得多的操作:

* There is outage implied and recovery is expected to be as fast as possible.
* 一些服务器可能会在这个过程中丢失. `orchestrator`必须确定是哪些, 如果有的话
> Some servers may be lost in the process. `orchestrator` must determine which, if any.
* State of the topology may be such that the user wishes to prevent the recovery.
* Master service discovery must take place: the app must be able to communicate with the new master (potentially be *told* that the master has changed).
* Find the best replica to promote.
   * 一种天真的方法是选择最新的副本, 但这可能并不总是正确的选择
> A naive approach would be to pick the most up-to-date replica, but that may not always be the right choice.
   * 最新的副本可能没有必要的配置来充当其他副本的主节点(例如, binlog 格式、MySQL 版本控制、复制过滤器等). 一味地推广最新的副本可能会丢失副本容量
> It may so happen that the most up-to-date replica will not have the necessary configuration to act as master to other replicas (e.g. binlog format, MySQL versioning, replication filters and more). By blindly promoting the most up-to-date replica one may lose replica capacity.
   * `orchestrator` 尝试提升将保留最多服务容量的副本.
> `orchestrator` attempts to promote a replica that will retain the most serving capacity.
* 提升所述副本, 接管其同级
> Promote said replica, taking over its siblings.
* Bring siblings up to date
* 可能的话, 做第二阶段选举提升; 如果可能的话, 用户可能已经标记了要提升的特定服务器(见 `register-candidate` 命令)
> Possibly, do a 2nd phase promotion; the user may have tagged specific servers to be promoted if possible (see `register-candidate` command).
* Call upon hooks (read further)

新主库的发现很大程度是需要用户自行实现的. 常见的解决方案是:

> Master service discovery is largely the user's responsibility to implement. Common solutions are:

* 基于DNS 的发现;  `orchestrator` 将需要调用一个修改 DNS 条目的钩子.
> DNS based discovery; `orchestrator` will need to invoke a hook that modifies DNS entries.
* ZooKeeper/Consul KV/etcd/其他基于键值的发现; `orchestrator` 内置了对 Consul KV 的支持, 否则外部钩子必须更新 KV 存储.
> ZooKeeper/Consul KV/etcd/other key-value based discovery; `orchestrator` has built-in support for Consul KV, otherwise an external hook must update KV stores
* 基于Proxy的发现;  `orchestrator` 将调用修改代理配置的外部钩子, 或者将更新如上所述的 Consul/Zk/etcd, 这本身将触发对代理配置的更新.
> Proxy based discovery; `orchestrator` will call upon external hook that modifies proxy config, or will update Consul/Zk/etcd as above, which will itself trigger an update to proxy configuration.
* Other solutions...

`orchestrator`试图成为通用的解决方案, 因此对您的服务发现方法不采取任何立场.



## Automated recovery
可选. 自动恢复可以应用于所有(`"*"`)集群或特定集群.

恢复遵循检测, 并且假设恢复没有被阻塞(阅读下文):

> Recovery follows detection, and assuming nothing blocks recovery (read further below)

For greater resolution, different configuration applies for master recovery and for intermediate-master recovery. Detailed breakdown of recovery-related configuration follows. 与恢复相关的配置的详细分类如下.

该分析机制一直在运行, 并定期检查failure/recovery的情况. 它将为以下情况做出自动恢:

> The analysis mechanism runs at all times, and checks periodically for failure/recovery scenarios. It will make an automated recovery for:

* 一个可操作性的情景类型.
> An actionable type of scenario
* 对于未停机的实例.
> For an instance that is not downtimed
* 对于属于通过配置显式启用恢复的集群的实例.
> For an instance belonging to a cluster for which recovery is explicitly enabled via configuration
* 对于最近未恢复的集群中的实例,除非此类最近的恢复得到确认
> For an instance in a cluster that has not recently been recovered, unless such recent recoveries were acknowledged
* 在启用全局恢复的情况下
> Where global recoveries are enabled

## Graceful master promotion 在线切换
*a.k.a graceful master takeover*

**TL;DR** 用这个方法, 在有序的、有计划的操作中切换主库.

通常, 出于升级、主机维护等目的, 您可能希望做一个主从切换操作. 这是一个优雅的切换(aka graceful master takeover). 它本质上是角色的转换: 现有的副本变成新的主, 旧的主变成副本.

In a graceful takeover:

*  用户或`orchestrator`选择一个现有的副本作为指定的新主库.
> either the user or `orchestrator` choose an existing replica as the designated new master.
* `orchestrator`确保指定的副本takes over its siblings as intermediate master.
> `orchestrator` ensures the designated replica takes over its siblings as intermediate master
* `orchestrator` 将打开主库的`read-only` (通常也会打开 `super-read-only`).
> `orchestrator` turns the master to be `read-only` (possibly also `super-read-only`)
* `orchestrator` 确保你指定的server caught up with replication.
> `orchestrator` makes sure your designated server is caught up with replication.
* `orchestrator` 将您指定的server提升为新的主库.
> `orchestrator` promotes your designated server as the new master.
* `orchestrator` 将提升的server变为可写.
> `orchestrator` turns promoted server to be writable.
* `orchestrator`将旧主库降级, 并将其作为新主库的从库.
> `orchestrator` demotes the old master and places it as a direct replica of the new master
* 如果可能, `orchestrator`为降级的主库设置复制用户/密码
> if possible, `orchestrator` sets replication user/password for the demoted master
* 在`graceful-master-takeover-auto` 变体(见下文)中, `orchestrator` 在降级的master 上开始复制
> in the `graceful-master-takeover-auto` variant (see following), `orchestrator` starts replication on demoted master.

该操作可能需要几秒钟, 在此期间, 你的应用程序预计会抱怨, 因为看到主库是`read-only`只读的.

`orchestrator` 为您提供了专门的钩子来运行优雅的接管:

* `PreGracefulTakeoverProcesses`
* `PostGracefulTakeoverProcesses`

这些钩子是在标准钩子之外运行的. `orchestrator` 将运行 `PreGracefulTakeoverProcesses`, 然后通过 `DeadMaster` 流程, 为 DeadMaster 运行正常的 pre-、post-hook, 最后跟进 `PostGracefulTakeoverProcesses`

我们发现, 有些操作在优雅接管(graceful-takeover)和真正的故障转移之间是相似的, 而有些则不同. 例如, `PreGracefulTakeoverProcesses`和`PostGracefulTakeoverProcesses`钩子可以用来屏蔽告警. 你可能想在计划中的故障切换期间屏蔽告警. 高级用法可能包括在代理层停滞流量.

在正常的pre-, post-故障转移过程中, 你可以使用`{command}`占位符, 或者`ORC_COMMAND`环境变量来检查这是否是一个优雅的接管. 你会看到 `graceful-master-takeover` 这个值.

优雅接管有两种变体:

> There are two variations of graceful takeover:

* `graceful-master-takeover`:
   * 用户必须指定要提升的副本
> The user must indicate the designated replica to be promoted
   * 或者, 设置拓扑结构, 使master只有一个直接的从库, 这隐式地使其成为指定副本(candidate master)
> or, setup topology such that the master only has a single direct replica, which implicitly makes it the designated replica
   * 降级的old master被放置为new master的从库, 但`orchestrator`并没有在old master上执行`start slave` .
> the demoted master is placed as replica to the new master, but `orchestrator` does not start replication on the server.
* `graceful-master-takeover-auto`:
   * 用户可以指定要提升的指定副本, `orchestrator`必须遵守.
> The user *may* indicate the designated replica to be promoted, which `orchestrator` must respect
   * 或者, 用户可以不指定new master, 在这种情况下, `orchestrator`会选择最佳的副本进行提升.
> or, the user may omit the identity of designated replica, in which case `orchestrator` picks the best replica to promote
   * 降级的old master作为new master的从库, 并且 `orchestrator` 会在old master上执行`start slave` .
> the demoted master is placed as replica to the new master, and `orchestrator` starts replication on the server.

通过以下方式调用优雅接管:

> Invoke graceful takeover via:

* Command line; examples:
   * `orchestrator-client -c graceful-master-takeover -alias mycluster -d designated.master.to.promote:3306`: 指定new master; `orchestrator` 不会在old master上启动复制(start slave)
   * `orchestrator-client -c graceful-master-takeover-auto -alias mycluster -d designated.master.to.promote:3306`: 指定new master; `orchestrator` 会在old master上启动复制(start slave)
   * `orchestrator-client -c graceful-master-takeover-auto -alias mycluster`: 让 `orchestrator` 自己选择new master; `orchestrator` 会在old master上启动复制(start slave)
* Web API; examples:
   * `/api/graceful-master-takeover/:clusterHint/:designatedHost/:designatedPort`: gracefully promote a new master (planned failover), 指示要提升的new master.
   * `/api/graceful-master-takeover/:clusterHint`: gracefully promote a new master (planned failover). 没有指定new master, 当master只有一个直接从库时工作.
   * `/api/graceful-master-takeover-auto/:clusterHint`: gracefully promote a new master (planned failover). `orchestrator` 选择一个new master. `orchestrator` 会在old master上启动复制(start slave)
* Web interface: 拖动master的直接从库到master box的左半边. web interface会使用`graceful-master-takeover` ; 降级的master上的复制将不会启动.
> Web interface: drag a direct master's replica onto the left half of the master's box. The web interface uses the `graceful-master-takeover` variation; the replication on demoted master will not kick in.

## Manual recovery 手动恢复
TL;DR 当主库故障, 但自动恢复被禁用或blocked时使用此选项.

你可以通过指定故障实例来要求`orchestrator` 进行故障恢复. 该实例必须被确认为发生了故障. 可以请求恢复已停机的实例（由于这是手动恢复，它覆盖了自动假设）.

通过以下方式恢复:

* Command line: `orchestrator-client -c recover -i dead.instance.com:3306 --debug`
* Web API: `/api/recover/dead.instance.com/:3306`
* Web: instance is colored black; click the `Recover` button

手动恢复不会在`RecoveryPeriodBlockSeconds`上阻塞(请在下一节阅读更多内容). 它们也覆盖了`RecoverMasterClusterFilters`和`RecoverIntermediateMasterClusterFilters`. 因此, 人们总是可以根据需求调用恢复. 一个恢复可能只会阻塞同时在同一数据库实例上运行的另一个恢复

> A recovery may only block on yet another recovery running at that time on the same database instance.(这个意思应该是说, 同时对两个集群进行恢复是可以的, 但是一个实例同一时间只能运行一个恢复操作, 后者会被阻塞).

## Manual, forced failover 手动强制failover
TL;DR 强制主库failover, 无论`orchestrator` 怎么想

也许`orchestrator`没有看到这个实例是失败的, 或者你有一些应用逻辑要求主库必须马上failover, 或者也许失败的类型是`orchestrator`不确定的. 你希望现在就启动一个master failover. 你将运行:

* Command line: `orchestrator-client -c force-master-failover --alias mycluster`

or `orchestrator-client -c force-master-failover -i instance.in.that.cluster`

* Web API: `/api/force-master-failover/mycluster`

or `/api/force-master-failover/instance.in.that.cluster/3306`



## Web, API, command line
通过以下方式对恢复情况进行审计:

> Recoveries are audited via:

* `/web/audit-recovery`
* `/api/audit-recovery`
* `/api/audit-recovery-steps/:uid`

细微差别审计和控制可通过:

> Nuance auditing and control available via:

* `/api/blocked-recoveries`: see blocked recoveries
* `/api/ack-recovery/cluster/:clusterHint`: acknowledge a recovery on a given cluster
* `/api/ack-all-recoveries`: acknowledge all recoveries
* `/api/disable-global-recoveries`: global switch to disable `orchestrator` from running any recoveries
* `/api/enable-global-recoveries`: re-enable recoveries
* `/api/check-global-recoveries`: check is global recoveries are enabled

Running manual recoveries (see next sections):

* `/api/recover/:host/:port`: recover specific host, assuming `orchestrator` agrees there is failure.
* `/api/recover-lite/:host/:port`: same, do not invoke external hooks (can be useful for testing)
* `/api/graceful-master-takeover/:clusterHint/:designatedHost/:designatedPort`: gracefully promote a new master (planned failover), indicating the designated master to promote.
* `/api/graceful-master-takeover/:clusterHint`: gracefully promote a new master (planned failover). Designated server not indicated, works when the master has exactly one direct replica.
* `/api/force-master-failover/:clusterHint`: panic, force master failover for given cluster

一些相应的命令行调用:

> Some corresponding command line invocations:

* `orchestrator-client -c recover -i some.instance:3306`
* `orchestrator-client -c graceful-master-takeover -i some.instance.in.somecluster:3306`
* `orchestrator-client -c graceful-master-takeover -alias somecluster`
* `orchestrator-client -c force-master-takeover -alias somecluster`
* `orchestrator-client -c ack-cluster-recoveries -alias somecluster`
* `orchestrator-client -c ack-all-recoveries`
* `orchestrator-client -c disable-global-recoveries`
* `orchestrator-client -c enable-global-recoveries`
* `orchestrator-client -c check-global-recoveries`

## Blocking, acknowledgements, anti-flapping
`orchestrator` 通过引入阻塞周期来避免抖动(级联故障导致持续中断和资源消除), 在任何给定的集群上, 除非人工允许, 否则 `orchestrator` 不会在小于所述周期的间隔内启动自动恢复

block period由 `RecoveryPeriodBlockSeconds` 指示. 它仅适用于同一集群上的恢复. 没有什么可以阻止在不同集群上运行的并发恢复.

一旦`RecoveryPeriodBlockSeconds`过了, 或者恢复被确认了, Pending中的recoveries就会unblocked

可以通过 Web API/interface（see audit/recovery page）或通过command line interface确认恢复(Acknowledging a recovery )(`orchestrator-client -c ack-cluster-recoveries -alias somealias`).

请注意, 手动恢复 (e.g. `orchestrator-client -c recover` or `orchestrator-client -c force-master-failover`) 会忽略阻塞期( blocking period).



## Adding promotion rules
有些服务器在故障转移的情况下是更好的晋升人选. 有些服务器不是很好的人选. 比如说:

* 硬件配置较差的server. 你不希望他被选为new leader.
* 在另一个数据中心的server. 你不希望他被选为new leader.
* 作为备份节点的server: 如会运行LVM快照备份, mysqldump, xtrabackup等等. 你不希望他被选为new leader.
* 一个配置较好的server, 是理想的候选人. 你希望他被选为new leader.
* 任何一个状态正常的server, 你没有特别的选择意见

您将以下列方式向`orchestrator`协调者宣布您对某一特定服务器的偏爱:

```Plain Text
orchestrator-client -c register-candidate -i ${::fqdn} --promotion-rule ${promotion_rule}
```
Supported promotion rules are:

* `prefer`
* `neutral` 中立
* `prefer_not`
* `must_not`

Promotion rules在一小时后失效.  That's the dynamic nature of `orchestrator`. 你需要设置一个cron作业来宣布服务器的promotion rule:

```bash
*/2 * * * * root "/usr/bin/perl -le 'sleep rand 10' && /usr/bin/orchestrator-client -c register-candidate -i this.hostname.com --promotion-rule prefer"
```
此设置来自生产环境. cron entries通过`puppet` 更新来设置新的`promotion_rule` . 一个server可能现在是`prefer`的, 但5分钟后就是`prefer_not`

> 这取决于你们公司自己的选主逻辑, 如, prefer的服务器要是是例行维护了, 那么就要更改`promotion_rule`

你可以整合你自己的服务发现方法、你自己的脚本, 以提供你最新的`promotion_rule`

## Downtime
所有的failure/recovery情况都得到了分析. 然而, 也考虑到了实例的停机状态.

一个实例的停机状态. 一个实例可以被停机（通过`orchestrator-client -c begin-downtime`）, 这将在分析摘要中指出. 在考虑自动恢复时, 会跳过停机的服务器.

> All failure/recovery scenarios are analyzed. However also taken into consideration is the downtime status of
an instance. An instance can be downtimed (via `orchestrator-client -c begin-downtime`) and this is noted in the analysis summary. When considering automated recovery, downtimed servers are skipped.

事实上, 停机时间正是为了这个目的而明确创建的, 它允许DBA有办法抑制自动故障转移和特定服务器

> Downtime was, in fact, explicitly created for this very purpose, and allows the DBA a way to suppress automated failover and a specific server.

请注意, 手动恢复(例如 `orchestrator-client -c recover`)会覆盖停机时间.

## Recovery hooks
`orchestrator`支持钩子--通过恢复过程调用的外部脚本. 这些是通过shell, 特别是bash调用的命令数组. 参见恢复配置中的[Hooks](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Configuration%20%20Recovery.md#hooks)

* `OnFailureDetectionProcesses`: described in [Failure detection](Failure%20detection%20%26%20recovery/Failure%20detection.md).
* `PreGracefulTakeoverProcesses`: 在old master进入只读状态之前, 在`graceful-master-takeover`命令中调用.
* `PreFailoverProcesses`
* `PostMasterFailoverProcesses`
* `PostIntermediateMasterFailoverProcesses`
* `PostFailoverProcesses`
* `PostUnsuccessfulFailoverProcesses`
* `PostGracefulTakeoverProcesses`: executed on planned, graceful master takeover, 在old master被置于new master之下后.
