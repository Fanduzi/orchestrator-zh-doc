# FAQ
# [Frequently Asked Questions](https://github.com/openark/orchestrator/blob/master/docs/faq.md)
### 谁应该使用orchestrator?
面对不仅仅是一主一从集群架构的DBA和运维人员.

### Orchestrator能为我做什么?
`Orchestrator` 分析你的复制拓扑结构, 并提供信息和行动: 你将能够可视化&操作这些拓扑结构(refactoring replication paths). `Orchestrator`可以监控和恢复拓扑结构的故障, 包括主库宕机.

### 这又是一个监控工具吗?
不. 严格来说, `Orchestrator`不是一个监控工具. 它并不打算这样做; 没有警报或电子邮件. 但它确实提供了拓扑状态的在线可视化, 并需要一些自己的阈值来管理拓扑.

### Orchestrator支持什么类型的复制?
`Orchestrator`支持 "plain-old-MySQL-replication", 即使用二进制日志文件和位置的那种. 如果你不知道你在用什么, 可能就是这个.

### Orchestrator支持Row Based Replication吗?
支持. Statement Based Replication和Row Based Replication都被支持(而这种区别实际上与`orchestrator`无关)

### Orchestrator是否支持半同步复制?
支持.

### Orchestrator是否支持Master-Master (ring) Replication?
支持, for a ring of two masters (active-active, active-passive).

Master-Master-Master\[-Master...\] 拓扑结构, 即由3个或更多的master组成的环不被支持, 也没有测试. 而且不鼓励使用. 并且是一种可憎的行为.

### Orchestrator支持Galera Replication吗?
是的, 也不是. `Orchestrator`不知道Galera复制的情况. 如果你有三个Galera master, 每个master下有不同的复制拓扑结构, 那么`Orchestrator`会把这些看作是三个不同的拓扑结构.

### Orchestrator支持基于GTID的复制吗?
支持. Oracle GTID和MariaDB GTID都被支持.

### Orchestrator支持5.6版本的并行复制吗(thread per schema)?
不支持. 这是因为`START SLAVE UNTIL`在并行复制中不被支持, 而且`SHOW SLAVE STATUS`的输出是不完整的. 在这方面没有预期的工作.

> No. This is because `START SLAVE UNTIL` is not supported in parallel replication, and output of `SHOW SLAVE STATUS` is incomplete. There is no expected work on this.

### Orchestrator支持5.7版本的并行复制吗?
支持. 只要使用GTID就可以. 当使用Pseudo-GTID时, 你必须启用in-order-replication(set [slave\_preserve\_commit\_order](http://dev.mysql.com/doc/refman/5.7/en/replication-options-slave.html#sysvar_slave_preserve_commit_order)).

### Does orchestrator support Multi-Master Replication?
这应该指的是多源复制吧. 不支持

No. Multi Master Replication (e.g. as in MariaDB 10.0) is not supported.

### Does orchestrator support Tungsten Replication?
不支持.

> [Tungsten-Replicator ](https://www.continuent.com/products/tungsten-replicator)是第三方的MySQL数据复制引擎，是个商业产品，同时提供开源版本。类似于MySQL 自身的replication，基于日志复制模式，不同的是 Tungsten 通过Extractor控件读取mysql主库的binlog 解析成自己的日志格式--THL（Transaction History Log）, 在从库上通过Applier控件写入数据库。

> Tungsten-Replicator 具有以下特性：

> * 支持高版本MySQL向低版本复制，如：MySQL5.1 --> MySQL5.0;
> * 支持跨数据库系统的复制，如：MySQL --> PostgreSQL
> * 支持多主库向单台Slave 的复制，Multi-Master --> Slave
> * Ganji-Replicator提取数据的更新记录，写到MySQL 队列表 Queue;基于这个队列，可以为其他应用服务提供便利，如检索系统数据更新，跨机房半同步。 MySQL --> Queue

### Orchestrator支持MySQL Group Replication吗?
部分支持. 在MySQL8.0版本支持single primary mode的组复制. 支持的范围是:

* `Orchestrator` 了解所有组成员都是同一集群的一部分, 检索复制组信息作为实例发现的一部分, 将其存储在其数据库中, 并通过 API 公开
> Orchestrator understands that all group members are part of the same cluster, retrieves replication group information as part of instance discovery, stores it in its database, and exposes it via the API.
* Orchestrator Web UI 显示single primary group members. 它们显示如下:
> The orchestrator web UI displays single primary group members. They are shown like this:
   * 所有secondary成员都是从primary复制的
> All secondary group members as replicating from the primary.
   * 所有组成员都有一个图标, 显示他们是组成员(相对于传统的异步/半同步复制).
> All group members have an icon that shows they are group members (as opposed to traditional async/semi-sync replicas).
   * 将鼠标悬停在上述图标上可提供有关数据库实例在组中的状态和角色的信息.  对于组成员来说, 一些重新定位操作是被禁止的. 特别是, orchestrator将拒绝重新定位一个secondary节点, 因为根据定义, 它总是从primary节点进行复制. 它也会拒绝在同一组的secondary节点下重新定位一个primary节点的尝试
> Hovering over the icon mentioned above provides information about the state and role of the DB instance in the group.\* Some relocation operations are forbidden for group members. In particular, orchestrator will refuse to relocate a secondary group member, as it, by definition, replicates always from the group primary. It will also reject an attempt to relocate a group primary under a secondary of the same group.
* 来自失败组成员的传统异步/半同步副本将被重新定位到MGR集群的其他组成员上.
> Traditional async/semisync replicas from failed group members are relocated to a different group member.

### Does orchestrator support Yet Another Type of Replication?
No.

### Does orchestrator support...
No.

### Is orchestrator open source?
Yes. `Orchestrator` is released as open source under the Apache License 2.0 and is available at: [https://github.com/openark/orchestrator](https://github.com/openark/orchestrator)

### Who develops orchestrator and why?
`Orchestrator` is developed by [Shlomi Noach](https://github.com/shlomi-noach) at [GitHub](http://github.com/) (previously at [Booking.com](http://booking.com/) and [Outbrain](http://outbrain.com/)) to assist in managing multiple large replication topologies; time and human errors saved so far are almost priceless.

## Orchestrator 是否可以与包含多个版本的MySQL集群一起使用?
部分支持. 这经常出现在升级一个不能停机的集群时. 每个副本将被脱机并升级到新的主要版本, 然后再添加到集群中, 直到所有副本都被升级. Orchestrator知道MySQL的版本, 并将允许一个主版本较高的副本移动到一个主版本或中间主版本较低的副本下, 但不允许反过来，因为这通常不被上游供应商支持, 即使它可能实际工作. 在大多数情况下, 协调器会做正确的事情, 它将允许在拓扑结构中安全地移动这些复制, 如果这是可能的. 这在MySQL 5.5/5.6中被广泛使用, 在5.6/5.7中也有使用, 但在MariaDB 10中却没有这么多. 如果你看到可能与此有关的问题, 请报告它们.

