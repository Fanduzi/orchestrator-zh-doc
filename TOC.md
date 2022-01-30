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
* [Executing via command line](Use/Executing via command line.md)
* [[Using the Web interface id=&#39;551296b1-e3c1-4d75-9cfb-638a3ddc3dfd&#39;]]
* [[Using the web API id=&#39;2ddaa83e-960a-4045-a0c3-e855064bb811&#39;]]: 通过HTTP GET请求实现自动化
*  Using [[orchestrator-client id=&#39;071296f1-6834-4c7c-849b-73f30c8b0fe2&#39;]]: a no binary/config needed script that wraps API calls
* [[Scripting samples id=&#39;e7857a23-5d93-471c-b4d4-99444b5034a6&#39;]]

#### Deployment
* [[Orchestrator高可用 id=&#39;ed5e7b21-c508-44b8-817e-d7c782082cf3&#39;]]: making `orchestrator` highly available
* [[在生产环境中部署Orchestrator id=&#39;758fdd72-feac-4c95-84ea-86c51c0fafe9&#39;]]
* [[shard backend模式部署 id=&#39;aa2b3871-6aaa-4826-8dc8-6eda2f56045f&#39;]]
* [[raft模式部署 id=&#39;ccba5554-290a-4e44-9c9b-ada1fbcb3e1a&#39;]]

#### Failure detection & recovery
* [[Failure detection id=&#39;78787f6a-1f80-4d86-a3ba-e1a0e5993eae&#39;]]: how `orchestrator` detects failure, types of failures it can handle
* [[Topology recovery id=&#39;963e0044-a9d3-4110-9731-3a736cf82441&#39;]]: recovery process, promotion and hooks.
* [[Key-Value stores id=&#39;df7404f7-5427-4861-af1c-f5a706280b7d&#39;]]: master discovery for your apps

#### Operation
* [[Status Checks id=&#39;4b6601be-ae51-4f07-a0f6-36b6acbb4ce2&#39;]]
* [[Tags id=&#39;da6fd9fe-c0aa-403b-a6f3-5ad97d87c0c8&#39;]]

#### Developers
* [[Understanding CI id=&#39;2e904033-4a72-4c29-86de-d4c1cf7d2647&#39;]]
* [[Building and testing id=&#39;b967fce4-e108-4fe9-b12f-a790de7d0519&#39;]]
* [[System test environment id=&#39;42fbcec8-85d6-4313-8019-c482f9368b4c&#39;]]
* [[Docker id=&#39;7c127815-2807-4786-8308-13ec158af438&#39;]]
* [[Contributions id=&#39;810c807c-7f64-4918-989e-ed5cc10f29f0&#39;]]

#### Various
* [[Security id=&#39;b9023aa8-211e-4a05-b09d-19e11cef3268&#39;]]
* [[SSL and TLS id=&#39;f0c1c523-8161-4969-9e64-db5f21f1211c&#39;]]
* [[Pseudo GTID id=&#39;704951b3-e680-4398-83af-4d478608b808&#39;]]: refactoring and high availability without using GTID.
* [[Agents id=&#39;cce6fd11-0ddd-4d12-bf7e-2414454d4b32&#39;]]

#### Meta
* [[Risk id=&#39;25653394-5bc5-45d7-b6fd-5ecb1b7d4f74&#39;]]
* [[Gotchas id=&#39;f5388b06-1268-4c94-95db-6c8489208554&#39;]]
* [[Supported Topologies and Versions id=&#39;1885103b-b8a6-4fe2-b75d-6ebc8d639941&#39;]]
* [[Bugs id=&#39;4e0bab51-9e53-41f5-bade-1c3d98af7d13&#39;]]
* [[Who uses Orchestrator? id=&#39;cb663d66-b167-4319-9938-883f0e7542bb&#39;]]
* [[Presentations id=&#39;7fee802c-5fe6-4e44-9d0e-bf8b0248bf4f&#39;]]

#### Quick guides
* [[FAQ id=&#39;5ee4e41c-455d-4943-8d56-63233dd3f26b&#39;]]
* [[First Steps id=&#39;c7073e45-3ad8-4ba8-a48e-3597a2c8c820&#39;]], a quick introduction to `orchestrator`
