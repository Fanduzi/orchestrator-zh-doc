# Configuration: Raft
# [Configuration: raft](https://github.com/openark/orchestrator/blob/master/docs/configuration-raft.md)
本文讲述如何配置一个[Orchestrator/raft, consensus cluster](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/%E9%83%A8%E7%BD%B2/Orchestrator%20raft%2C%20consensus%20cluster.md)

 假设你要运行一个`3` 节点的`orchestrator/raft` 集群, 你将需要在每个节点进行以下配置:

```yaml
  "RaftEnabled": true,
  "RaftDataDir": "<path.to.orchestrator.data.directory>",
  "RaftBind": "<ip.or.fqdn.of.this.orchestrator.node>",
  "DefaultRaftPort": 10008,
  "RaftNodes": [
    "<ip.or.fqdn.of.orchestrator.node1>",
    "<ip.or.fqdn.of.orchestrator.node2>",
    "<ip.or.fqdn.of.orchestrator.node3>"
  ],
```
详细说明:

* `RaftEnabled` 必须设置为`true` , 否则`orchestrator` 将使用shared-backend模式.
* `RaftDataDir` 必须设置为一个`orchestrator` 有写入权限的目录. 如果目录不存在`orchestrator` 会尝试创建目录.
* `RaftBind` 必须设置, 使用本地主机的 IP 地址或完整主机名.  这个 IP 或主机名也将被列为 RaftNodes 变量之一( This IP or hostname will also be listed as one of the `RaftNodes` variable.)
* `DefaultRaftPort` 可以设置为任何端口, 但必须在所有部署中保持一致.
* `RaftNodes` 应该列出raft集群的所有节点. 这个列表将由IP地址或主机名组成, 并将包括`RaftBind`中提出的这个主机本身的值.

例如, 下面可能是一个生产环境配置:

```yaml
  "RaftEnabled": true,
  "RaftDataDir": "/var/lib/orchestrator",
  "RaftBind": "10.0.0.2",
  "DefaultRaftPort": 10008,
  "RaftNodes": [
    "10.0.0.1",
    "10.0.0.2",
    "10.0.0.3"
  ],
```
以及这个:

```yaml
  "RaftEnabled": true,
  "RaftDataDir": "/var/lib/orchestrator",
  "RaftBind": "node-full-hostname-2.here.com",
  "DefaultRaftPort": 10008,
  "RaftNodes": [
    "node-full-hostname-1.here.com",
    "node-full-hostname-2.here.com",
    "node-full-hostname-3.here.com"
  ],
```
### NAT, firewalls, routing
如果你的orchestrator/raft节点需要通过NAT网关进行通信, 你可以另外设置:

* `"RaftAdvertise": "<ip.or.fqdn.visible.to.other.nodes>"`

为其他节点应该联系的 IP 或主机名. 否则其他节点会尝试与“RaftBind”地址通信并失败.

raft节点将反向代理HTTP请求, `orchestrator`将尝试启发式地计算领导者的URL, 以重定向请求. 如果在NAT后面, 重新路由端口等( If behind NAT, rerouting ports etc.)，`orchestrator`可能无法计算出该URL。你可以配置:

* `"HTTPAdvertise": "scheme://hostname:port"`

明确指定节点（假设它是领导者）将通过 HTTP API 访问的位置.  例如, 您可以配置:  `"HTTPAdvertise": "http://my.public.hostname:3000"`

### Backend DB
raft模式支持`MySQL`或`SQLite`作为后端数据库. 详见[Configuration: Backend](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Setup/配置/Configuration%20%20Backend.md). 阅读[Orchestrator高可用](https://github.com/Fanduzi/orchestrator-zh-doc/blob/master/Deployment/Orchestrator高可用.md), 了解使用这两种方法的情景、可能性和理由.

### Single raft node setups
在生产中, 你会希望使用多个raft节点, 如`3`个或`5`个.

在测试环境中, 你可以运行一个由单个节点组成的orchestrator/raft集群. 这个节点将隐含地成为领导者, 并将向自己发布raft信息.

要运行一个单节点的`orchestrator/raft`, 配置一个空的`RaftNodes` .

```yaml
"RaftNodes": [],
```
or, alternatively, specify a single node, which is identical to `RaftBind` or `RaftAdvertise`:

```yaml
  "RaftEnabled": true,
  "RaftBind": "127.0.0.1",
  "DefaultRaftPort": 10008,
  "RaftNodes": [
    "127.0.0.1"
  ],
```
