# orchestrator-chn-doc
英语不好, 语文不好, 没时间校对.
文档从我个人[为知笔记](https://www.wiz.cn/zh-cn/)导出, 由于使用了[双向连接](https://www.wiz.cn/zh-cn/wiznote-wiki-links.html), 所以文档中很多链接是失效的.
我暂时没时间改了, 这个工作量太大了.

允许转载, 但请标明出处. 本项目所有文档遵循CC BY 4.0协议

For more information, please see
<https://creativecommons.org/licenses/by/4.0/>





# TOC
# [Table of Contents](https://github.com/openark/orchestrator/tree/master/docs#introduction)
#### Introduction
* [About](Introduction/About.md)
* [License](Introduction/License.md)
* [Download](Introduction/Download.md)
* [Requirements](Introduction/Requirements.md)

#### Setup
* [安装-Installation](Setup/部署/安装-Installation.md): installing the service/binary
* [Configuration](Setup/配置/Configuration.md): breakdown of major configuration variables by topic.

#### Use
* [Execution](Use/Execution.md): running the `orchestrator` service.
* [Executing via command line](Use/Executing%20via%20command%20line.md)
* [Using the Web interface](Use/Using%20the%20Web%20interface.md)
* [Using the web API](Use/Using%20the%20web%20API.md): 通过HTTP GET请求实现自动化
*  Using [orchestrator-client](Use/orchestrator-client.md): a no binary/config needed script that wraps API calls
* [Scripting samples](Use/Scripting%20samples.md)

#### Deployment
* [Orchestrator高可用](Deployment/Orchestrator高可用.md): making `orchestrator` highly available
* [在生产环境中部署Orchestrator](Deployment/在生产环境中部署Orchestrator.md)
* [shard backend模式部署](Deployment/shard%20backend模式部署.md)
* [raft模式部署](Deployment/raft模式部署.md)

#### Failure detection & recovery
* [Failure detection](Failure%20detection%20%26%20recovery/Failure%20detection.md): how `orchestrator` detects failure, types of failures it can handle
* [Topology recovery](Failure%20detection%20%26%20recovery/Topology%20recovery.md): recovery process, promotion and hooks.
* [Key-Value stores](Failure%20detection%20%26%20recovery/Key-Value%20stores.md): master discovery for your apps

#### Operation
* [Status Checks](Operation/Status%20Checks.md)
* [Tags](Operation/Tags.md)

#### Developers
* [Understanding CI](Developers/Understanding%20CI.md)
* [Building and testing](Developers/Building%20and%20testing.md)
* [System test environment](System%20test%20environment.md)
* [Docker](Developers/Docker.md)
* [Contributions](Developers/Contributions.md)

#### Various
* [Security](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Various/Security.md)
* [SSL and TLS](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Various/SSL%20and%20TLS.md)
* [Pseudo GTID](Various/Pseudo%20GTID.md): refactoring and high availability without using GTID.
* [Agents](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Various/Agents.md)

#### Meta
* [Risk](Meta/Risk.md)
* [Gotchas](Meta/Gotchas.md)
* [Supported Topologies and Versions](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/Meta/Supported%20Topologies%20and%20Versions.md)
* [Bugs](Meta/Bugs.md)
* [Who uses Orchestrator?](Meta/Who%20uses%20Orchestrator%20.md)
* [Presentations](Meta/Presentations.md)

#### Quick guides
* [FAQ](Quick%20guides/FAQ.md)
* [First Steps id](Quick%20guides/First%20Steps.md), a quick introduction to `orchestrator`


#### 配置文件参数详解
* [配置参数详解-Ⅰ](Setup/配置/配置参数详解-Ⅰ.md)
* [配置参数详解-Ⅱ](Setup/配置/配置参数详解-Ⅱ.md)
* [配置参数详解-III](Setup/配置/配置参数详解-III.md)
