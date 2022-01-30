# Gotchas
# [Gotchas](https://github.com/openark/orchestrator/blob/master/docs/gotchas.md)
* 默认情况下, `orchestrator` 每分钟仅轮询一次服务器（可通过 `InstancePollSeconds` 进行配置）. 这意味着您看到的任何状态本质上都是一种估计. 不同的实例在不同的时间被轮询. 例如, 您在集群页面上看到的状态不一定反映给定的时间点, 而是最后一分钟（或您使用的任何轮询间隔）中不同时间点的组合

The `problems` drop down to the right也是异步执行的. 因此, 您可能会在两个不同状态的两个位置（一次在集群页面中, 一次在问题下拉列表中）看到相同的实例.

> The `problems` drop down to the right is also executed asynchronously. You may therefore see the same instance in two places (once in the `cluster` page, once in the `problems` drop down) in two different states.

如果您想获得最新的实例状态, 请使用实例设置对话框中的“刷新”按钮

* `orchestrator`可能需要几分钟的时间来完全检测一个集群的拓扑结构. 这个时间取决于拓扑结构的深度(如果你有复制的复制, 时间会增加). 这是由于`orchestrator`独立地轮询实例, 对拓扑结构的了解必须在下次轮询时从主站传播到副本.
* 具体来说, 如果你故障切换到一个新的主站, 你可能会发现在几分钟内, 拓扑结构看起来是空的. 这可能发生, 因为实例曾经识别自己属于某个拓扑结构, 而这个拓扑结构现在正在被破坏. 这是自愈. 刷新并查看`Clusters`菜单, 查看随着时间推移新创建的集群(名称以新主机的名称命名).
* Specifically, if you fail over to a new master, you may find that for a couple minutes the topologies seem empty. This may happen because instances used to identify themselves as belonging to a certain topology that is now being destroyed. This is self-healing. Refresh and look at the `Clusters` menu to review the newly created cluster (names after the new master) over time.
* Don't restart `orchestrator` while you're running a seed (only applies to working with *orchestrator-agent*)

Otherwise `orchestrator` is non-intrusive and self-healing. You can restart it whenever you like.

