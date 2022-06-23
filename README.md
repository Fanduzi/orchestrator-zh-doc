# orchestrator-zh-doc
英语不好, 语文不好, 凑合看吧.

允许转载, 但请标明出处. 本项目所有文档遵循CC BY 4.0协议

For more information, please see
<https://creativecommons.org/licenses/by/4.0/>





# TOC
# [Table of Contents](https://github.com/openark/orchestrator/tree/master/docs#introduction)
#### Introduction
* [About](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Introduction/About.md)
* [License](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Introduction/License.md)
* [Download](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Introduction/Download.md)
* [Requirements](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Introduction/Requirements.md)

#### Setup
* [安装-Installation](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%83%A8%E7%BD%B2/%E5%AE%89%E8%A3%85-Installation.md): installing the service/binary
* [Configuration](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%85%8D%E7%BD%AE/Configuration.md): breakdown of major configuration variables by topic.

#### Use
* [Execution](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Use/Execution.md): running the `orchestrator` service.
* [Executing via command line](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Use/Executing%20via%20command%20line.md)
* [Using the Web interface](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Use/Using%20the%20Web%20interface.md)
* [Using the web API](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Use/Using%20the%20web%20API.md): 通过HTTP GET请求实现自动化
*  Using [orchestrator-client](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Use/orchestrator-client.md): a no binary/config needed script that wraps API calls
* [Scripting samples](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Use/Scripting%20samples.md)

#### Deployment
* [Orchestrator高可用](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Deployment/Orchestrator高可用.md): making `orchestrator` highly available
* [在生产环境中部署Orchestrator](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Deployment/在生产环境中部署Orchestrator.md)
* [shard backend模式部署](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Deployment/shard%20backend模式部署.md)
* [raft模式部署](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Deployment/raft模式部署.md)

#### Failure detection & recovery
* [Failure detection](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Failure%20detection.md): how `orchestrator` detects failure, types of failures it can handle
* [Topology recovery](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Topology%20recovery.md): recovery process, promotion and hooks.
* [Key-Value stores](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Failure%20detection%20%26%20recovery/Key-Value%20stores.md): master discovery for your apps

#### Operation
* [Status Checks](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Operation/Status%20Checks.md)
* [Tags](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Operation/Tags.md)

#### Developers
* [Understanding CI](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Developers/Understanding%20CI.md)
* [Building and testing](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Developers/Building%20and%20testing.md)
* [System test environment](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Developers/System%20test%20environment.md)
* [Docker](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Developers/Docker.md)
* [Contributions](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Developers/Contributions.md)

#### Various
* [Security](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Various/Security.md)
* [SSL and TLS](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Various/SSL%20and%20TLS.md)
* [Pseudo GTID](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Various/Pseudo%20GTID.md): refactoring and high availability without using GTID.
* [Agents](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Various/Agents.md)

#### Meta
* [Risk](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Meta/Risk.md)
* [Gotchas](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Meta/Gotchas.md)
* [Supported Topologies and Versions](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Meta/Supported%20Topologies%20and%20Versions.md)
* [Bugs](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Meta/Bugs.md)
* [Who uses Orchestrator?](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Meta/Who%20uses%20Orchestrator%20.md)
* [Presentations](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Meta/Presentations.md)

#### Quick guides
* [FAQ](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Quick%20guides/FAQ.md)
* [First Steps](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Quick%20guides/First%20Steps.md), a quick introduction to `orchestrator`


#### 配置文件参数详解
* [配置参数详解-Ⅰ](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/配置/配置参数详解-Ⅰ.md)
* [配置参数详解-Ⅱ](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/配置/配置参数详解-Ⅱ.md)
* [配置参数详解-Ⅲ](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/配置/配置参数详解-Ⅲ.md)


#### 命令详解
* [relocate](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Use/commands/relocate.md)

#### 源码分析
* [Orchestrator Failover过程源码分析-I](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/Orchestrator%20Failover%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-I.md)
* [Orchestrator Failover过程源码分析-II](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/Orchestrator%20Failover%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-II.md)
* [Orchestrator Failover过程源码分析-III](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/Orchestrator%20Failover%E8%BF%87%E7%A8%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-III.md)