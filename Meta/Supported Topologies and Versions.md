# Supported Topologies and Versions
# [Supported Topologies and Versions](https://github.com/openark/orchestrator/blob/master/docs/supported-topologies-and-versions.md)
# Supported Topologies and Versions
The following setups are supported by `orchestrator`:

* Plain-old MySQL replication; the *classic* one, based on log file + position
* GTID replication. Both Oracle GTID and MariaDB GTID are supported.
* Statement based replication (SBR)
* Row based replication (RBR)
* Semi-sync replication
* Single master (aka standard) replication
* Master-Master (two node in circle) replication
* 5.7 Parallel replication
* When using GTID there's no further constraints.
* When using Pseudo-GTID in-order-replication must be enabled (see [slave\_preserve\_commit\_order](http://dev.mysql.com/doc/refman/5.7/en/replication-options-slave.html#sysvar_slave_preserve_commit_order)).

The following setups are *unsupported*:

* Master-master...-master (circular) replication with 3 or more nodes in ring.
* 5.6 Parallel (thread per schema) replication
* Multi master replication (one replica replicating from multiple masters)
* Tungsten replicator

Also note:

Master-master (ring) replication is supported for two master nodes. Topologies of three master nodes or more in a ring are unsupported.

Galera/XtraDB Cluster replication is not strictly supported: `orchestrator` will not recognize that co-masters in a Galera topology are related. Each such master would appear to `orchestrator` to be the head of its own distinct topology.

支持在同一主机上有多个MySQL实例的复制拓扑结构. 例如, `orchestrator`的测试环境是由四个实例组成的, 它们都运行在同一台机器上，由 MySQLSandbox 提供. 然而, MySQL在复制体和主站之间缺乏信息共享, 使得`orchestrator`无法自上而下地分析拓扑结构, 因为主站不知道其复制体在监听哪些端口. 默认的假设是, 复制体的监听端口与主站相同. 如果在一台机器上有多个实例（并且在同一个网络上）, 这是不可能的. 在这种情况下, 你必须配置你的MySQL实例的`report_host`和`report_port`（[The importance of report\_host &amp; report\_port](http://code.openark.org/blog/mysql/the-importance-of-report_host-report_port)）参数, 并将`orchestrator`的配置参数`DiscoverByShowSlaveHosts`设置为`true`.
