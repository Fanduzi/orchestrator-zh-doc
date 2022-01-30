# Configuration: Failure detection
# [Configuration: failure detection](https://github.com/openark/orchestrator/blob/master/docs/configuration-failure-detection.md)
`orchestrator`将始终检测您的拓扑结构的故障. 作为一个配置问题, 您可以设置轮询频率和具体方式, 以便`orchestrator`在检测到故障时通知您.

>  [[Failure detection id=&#39;78787f6a-1f80-4d86-a3ba-e1a0e5993eae&#39;]]

恢复将在[[Configuration: Recovery id=&#39;1e244518-4a10-46c4-81b5-2da1c8998295&#39;]]中讨论

```yaml
{
  "FailureDetectionPeriodBlockMinutes": 60,
}
```
`orchestrator`每秒钟运行一次检测

`FailureDetectionPeriodBlockMinutes`是一种反垃圾邮件机制, 它可以阻止`orchestrator`一次又一次地通知同一检测.

### Hooks
配置`orchestrator`以在发现时采取行动:

```yaml
{
  "OnFailureDetectionProcesses": [
    "echo 'Detected {failureType} on {failureCluster}. Affected replicas: {countReplicas}' >> /tmp/recovery.log"
  ],
}
```
有许多神奇的变量(如上面的`{failureCluster}`), 你可以发送给你的外部钩子. 完整列表请见[[Topology recovery id=&#39;963e0044-a9d3-4110-9731-3a736cf82441&#39;]]

### MySQL configuration
由于故障检测使用MySQL拓扑结构本身作为信息来源, 因此建议你设置你的MySQL复制, 以便错误将被清楚地显示或快速地缓解.

* `set global slave_net_timeout = 4` 详见[文档](https://dev.mysql.com/doc/refman/5.7/en/replication-options-replica.html#sysvar_slave_net_timeout). 这在从库和它的主库之间设置了一个很短的（2秒）心跳间隔, 并将使从库迅速识别故障. 如果没有这个设置, 些情况下可能需要一分钟才能检测到.
>  `MASTER_HEARTBEAT_PERIOD` The heartbeat interval defaults to half the value of the `slave_net_timeout` system variable
* `CHANGE MASTER TO MASTER_CONNECT_RETRY=1, MASTER_RETRY_COUNT=86400` .  在复制失败的情况下, 让复制体每隔1秒尝试重新连接(默认为60秒). 在短暂的网络问题下, 该设置尝试快速恢复复制, 如果成功, 将避免`orchestrator`的运行failure/recovery操作.













