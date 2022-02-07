# Configuration: Discovery, Pseudo-GTID
# [Configuration: discovery, Pseudo-GTID](https://github.com/openark/orchestrator/blob/master/docs/configuration-discovery-pseudo-gtid.md#automated-pseudo-gtid-injection)
`orchestrator`将通过创建Pseudo-GTID标识, 使其能够像拥有GTID一样操作非GTID拓扑结构, 包括重新定位复制, 智能故障转移等.

> `orchestrator` will identify magic hints in the binary logs, making it able to manipulate a non-GTID topology as if it had GTID, including relocation of replicas, smart failovers end more.

另见[Pseudo GTID](Various/Pseudo%20GTID.md)

### Automated Pseudo-GTID injection
`orchestrator` 可以为你注入Pseudo-GTID条目, 省去你的麻烦:

```yaml
{
  "AutoPseudoGTID": true,
}
```
你可以忽略任何其他与Pseudo-GTID相关的配置(它们都将被`orchestrator`隐式覆盖).

你将进一步需要在你的MySQL上授予以下内容:

```sql
GRANT DROP ON _pseudo_gtid_.* to 'orchestrator'@'orch_host';
```
**NOTE**: 无需创建`_pseudo_gtid_` 库. `orchestrator` 只是通过运行下面形式的语句注入Pseudo-GTID:

```sql
drop view if exists `_pseudo_gtid_`.`_asc:5a64a70e:00000001:c7b8154ff5c3c6d8`
```
这些语句不会做什么, 但会作为二进制日志中的神奇标记.

`orchestrator`将只在允许的地方尝试注入Pseudo-GTID. 如果你想把Pseudo-GTID注入限制在特定的集群上, 你可以这样做, 只在那些你希望`orchestrator`注入Pseudo-GTID的集群上授予该权限. 您可以通过以下方式在特定集群上禁用Pseudo-GTID注入.

```sql
REVOKE DROP ON _pseudo_gtid_.* FROM 'orchestrator'@'orch_host';
```
自动Pseudo-GTID注入是一个较新开发的功能, 它取代了你运行自己的伪-GTID注入的需要.

如果你希望在运行手动Pseudo-GTID注入后启用自动Pseudo-GTID注入, 你会很高兴地注意到:

* 你将不再需要管理pseudo-GTID服务/事件调度器.
* 尤其是在主库故障切换时, 你不需要在旧的/晋升的主站上禁用/启用Pseudo-GTID

### Manual Pseudo-GTID injection
建议使用[Automated Pseudo-GTID injection](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Configuration%20%20Discovery%2C%20Pseudo-GTID.md#automated-pseudo-gtid-injection)方式.

如果你想自己注入Pseudo-GTID, 我们建议你应按以下方式进行配置:

```yaml
{
  "PseudoGTIDPattern": "drop view if exists `meta`.`_pseudo_gtid_hint__asc:",
  "PseudoGTIDPatternIsFixedSubstring": true,
  "PseudoGTIDMonotonicHint": "asc:",
  "DetectPseudoGTIDQuery": "select count(*) as pseudo_gtid_exists from meta.pseudo_gtid_status where anchor = 1 and time_generated > now() - interval 2 hour",
}
```
上述假设是:

* `orchestrator`管理的MySQL集群中有一个`meta` 数据库
* 你将通过[这个示例脚本](https://github.com/openark/orchestrator/tree/master/resources/pseudo-gtid)注入Pseudo-GTID条目











