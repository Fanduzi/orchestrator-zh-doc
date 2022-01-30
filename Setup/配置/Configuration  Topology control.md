# Configuration: Topology control
# [Configuration: topology control](https://github.com/openark/orchestrator/blob/master/docs/configuration-topology-control.md)
The following configuration affects how `orchestrator` applies changes to topology servers:

`orchestrator` will figure out the name of the cluster, data center, and more.

```yaml
{
  "UseSuperReadOnly": false,
}
```
### UseSuperReadOnly
默认为`false`. 当为`true`时, 每当`orchestrator`被要求set/clear `read_only`时, 它也会将更改应用到`super_read_only`. `super_read_only`仅在Oracle MySQL和Percona Server上可用.