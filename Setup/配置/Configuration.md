# Configuration
# [Configuration](https://github.com/openark/orchestrator/blob/master/docs/configuration.md)
文档化和解释所有配置变量变成了一项艰巨的任务, 就像用文字来解释代码一样. 目前正在进行修剪和简化配置的工作.

The de-facto configuration list is located in [config.go](https://github.com/openark/orchestrator/blob/master/go/config/config.go).

> 配置解析的代码位于[config.go](https://github.com/openark/orchestrator/blob/master/go/config/config.go).

你无疑对配置一些基本组件感兴趣: 后端数据库、主机发现. 你可以选择使用Pseudo-GTID. 你可能希望`orchestrator`在故障时发出通知, 或者你可能希望运行全面的自动恢复.

Use the following small steps to configure `orchestrator`:

* [Configuration: Backend](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/配置/Configuration%20%20Backend.md)
* [Configuration: Basic Discovery](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Configuration%20%20Basic%20Discovery.md)
* [Configuration: Discovery, name resolving](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Configuration%20%20Discovery%2C%20name%20resolving.md)
* [Configuration: Discovery, classifying servers](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/配置/Configuration%20%20Discovery%2C%20classifying%20servers.md)
* [Configuration: Discovery, Pseudo-GTID](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/配置/Configuration%20%20Discovery%2C%20Pseudo-GTID.md)
* [Configuration: Topology control](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/配置/Configuration%20%20Topology%20control.md)
* [Configuration: Failure detection](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/配置/Configuration%20%20Failure%20detection.md)  
* [Configuration: Recovery](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/配置/Configuration%20%20Recovery.md)
* [Configuration: Raft](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Configuration%20%20Raft.md) configure a[Orchestrator/raft, consensus cluster](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/%E9%83%A8%E7%BD%B2/Orchestrator%20raft%2C%20consensus%20cluster.md)for high availability

*  Security: See[Security](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Various/Security.md)section.
* [Configuration: Key-Value stores](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/配置/Configuration%20%20Key-Value%20stores.md)configure and use key-value stores for master discovery.
* [Orchestrator configuration in larger environments](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Orchestrator%20configuration%20in%20larger%20environments.md)

### Configuration sample file
为方便起见, 这个[示例配置文件](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/%E7%A4%BA%E4%BE%8B%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6.md)是GitHub生产环境中使用的`orchestrator`配置.
