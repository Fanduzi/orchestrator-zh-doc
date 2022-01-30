# Configuration: Basic Discovery
# [Configuration: basic discovery](https://github.com/openark/orchestrator/blob/master/docs/configuration-discovery-basic.md)
让`orchestrator`知道如何查询MySQL拓扑结构, 提取哪些信息.

```yaml
{
  "MySQLTopologyCredentialsConfigFile": "/etc/mysql/orchestrator-topology.cnf",
  "InstancePollSeconds": 5,
  "DiscoverByShowSlaveHosts": false,
}
```
`MySQLTopologyCredentialsConfigFile`遵循与`MySQLOrchestratorCredentialsConfigFile`类似的规则. 你可以选择使用纯文本凭证.

```Plain Text
[client]
user=orchestrator
password=orc_topology_password
```
或者, 你可以选择使用纯文本凭证

```yaml
{
  "MySQLTopologyUser": "orchestrator",
  "MySQLTopologyPassword": "orc_topology_password",
}
```
`orchestrator`将每隔`InstancePollSeconds`秒进行一次探测.

请在所有的MySQL节点, 授予以下权限:

```sql
CREATE USER 'orchestrator'@'orc_host' IDENTIFIED BY 'orc_topology_password';
GRANT SUPER, PROCESS, REPLICATION SLAVE, REPLICATION CLIENT, RELOAD ON *.* TO 'orchestrator'@'orc_host';
GRANT SELECT ON meta.* TO 'orchestrator'@'orc_host';
GRANT SELECT ON ndbinfo.processes TO 'orchestrator'@'orc_host'; -- Only for NDB Cluster
GRANT SELECT ON performance_schema.replication_group_members TO 'orchestrator'@'orc_host'; -- Only for Group Replication / InnoDB cluster
```


















