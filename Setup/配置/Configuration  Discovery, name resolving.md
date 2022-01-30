# Configuration: Discovery, name resolving
# [Configuration: discovery, name resolving](https://github.com/openark/orchestrator/blob/master/docs/configuration-discovery-resolve.md)
让orchestrator知道如何解析主机名(hostname).

Most people will want this:

```yaml
{
  "HostnameResolveMethod": "default",
  "MySQLHostnameResolveMethod": "@@hostname",
}
```
您的主机可以通过IP地址、和/或short names、和/或fqdn, 和/或VIP相互指代.

`orchestrator`需要唯一和一致地识别一个主机. 它通过解析目标主机名来做到这一点.

`"MySQLHostnameResolveMethod": "@@hostname"` 是最简单的方式. 可选项有:

* `"HostnameResolveMethod": "cname"`: do CNAME resolving on hostname

* `"HostnameResolveMethod": "default"`: no special resolving via net protocols

* `"MySQLHostnameResolveMethod": "@@hostname"`: issue a `select @@hostname`

* `"MySQLHostnameResolveMethod": "@@report_host"`: issue a `select @@report_host`, requires `report_host` to be configured

* `"HostnameResolveMethod": "none"` and `"MySQLHostnameResolveMethod": ""`: do nothing. Never resolve. This may appeal to setups where everything uses IP addresses at all times.
