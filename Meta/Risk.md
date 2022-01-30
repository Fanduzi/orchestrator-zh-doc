# Risk
# [Risks](https://github.com/openark/orchestrator/blob/master/docs/risks.md)
大多数情况下, `orchestrator` 仅从您的拓扑中读取状态.  默认配置是每分钟轮询一次每个实例

`orchestrator`将连接到您的拓扑服务器, 并将限制并发连接的数量.

你可以用`orchestrator`来重构你的拓扑结构: 移动复制, 改变复制树. `orchestrator` 将尽最大努力:

1. 确保您只将实例移动到它可以复制的有效位置(例如, 您不要将 5.5 服务器放在 5.6 服务器之下)
2. 确保您在正确的时间移动实例(即实例和受影响的服务器没有严重滞后, 以便操作能够及时完成)
3. Do the math correctly: stop the replica at the right time, roll it forward to the right position, `CHANGE MASTER` to the correct location & position.

上述内容经过了很好的检验.

你可以使用`orchestrator`来故障转移你的拓扑结构. 你会关注到:

* `orchestrator`不会在没有需要时进行故障转移.
* 当有需要时, `orchestrator`会进行故障转移.
* 故障转移不会损失太多服务器.
> A failover doesn't lose too many servers
* 故障转移以一致的拓扑结构结束.
> Failover ends with a consistent topology

故障转移是通过钩子与你的部署内在地联系在一起的

> Failovers are inherently tied to your deployments through hooks.

故障转移总是有风险的. 请确保对其进行测试.

Please make sure to read the [LICENSE](https://github.com/openark/orchestrator/blob/master/LICENSE), and especially the "WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND" part.

