# Configuration
# [Configuration](https://github.com/openark/orchestrator/blob/master/docs/configuration.md)
文档化和解释所有配置变量变成了一项艰巨的任务, 就像用文字来解释代码一样. 目前正在进行修剪和简化配置的工作.

The de-facto configuration list is located in [config.go](https://github.com/openark/orchestrator/blob/master/go/config/config.go).

> 配置解析的代码位于[config.go](https://github.com/openark/orchestrator/blob/master/go/config/config.go).

你无疑对配置一些基本组件感兴趣: 后端数据库、主机发现. 你可以选择使用Pseudo-GTID. 你可能希望`orchestrator`在故障时发出通知, 或者你可能希望运行全面的自动恢复.

Use the following small steps to configure `orchestrator`:

* [[Configuration: Backend id=51d20469-439a-451a-a336-6726cff3a142]] 
* [[Configuration: Basic discovery id=427c7b47-af1a-4fbd-991d-150be63cb385]] 
* [[Configuration: Discovery, name resolving id=40d5828c-7e46-42d7-8143-1e80a5b08bbc]] 
* [[Configuration: Discovery, classifying servers id=b169c092-dcf3-4fe9-87ce-8cbe95b331e1]] 
* [[Configuration: Discovery, Pseudo-GTID id=&#39;f65504cf-f002-4cc4-beb7-acfac2b125c6&#39;]]
* [[Configuration: Topology control id=&#39;74887b53-dedf-4af0-bbc9-d8826af2455e&#39;]]
* [Configuration: Failure detection](Setup/配置/Configuration%20%20Failure%20detection.md)  
* [Configuration: Recovery](Setup/配置/Configuration%20%20Recovery.md)
* [[Configuration: Raft id=02ad8b7b-1cf4-4925-9233-1bd744018d25]] configure a[[Orchestrator/raft, consensus cluster id=355cf04c-56ad-4501-943a-39bbbc59e3bf]]for high availability

*  Security: See[[Security id=b9023aa8-211e-4a05-b09d-19e11cef3268]]section.
* [[Configuration: Key-Value stores id=&#39;351cd58b-23ad-4077-9e72-08491b671195&#39;]]configure and use key-value stores for master discovery.
* [[Orchestrator configuration in larger environments id=&#39;3d01d4fe-4a1a-4415-97e5-322717537a45&#39;]]

### Configuration sample file
为方便起见, 这个[[示例配置文件 id=f463a4e0-87bb-49e1-b14b-2296e480c0f3]]是GitHub生产环境中使用的`orchestrator`配置.