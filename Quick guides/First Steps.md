# First Steps
# [First Steps with Orchestrator](https://github.com/openark/orchestrator/blob/master/docs/first-steps.md)
你已经安装、部署和配置了Orchestrator. 你能用它做什么？

介绍一下常用命令, 主要是CLI侧的命令

> A walk through of common commands, mostly on the CLI side

#### Must
##### Discover
You need to discover your MySQL hosts. 要么浏览你的`http://orchestrator:3000/web/discover` 页面并提交一个实例以供发现, 要么:

```bash
$ orchestrator-client -c discover -i some.mysql.instance.com:3306
```
`:3306` 不是必须的, 因为`DefaultInstancePort` 默认就是`3306` . You may also:

```bash
$ orchestrator-client -c discover -i some.mysql.instance.com
```
这就发现了一个单一的实例. 但是: 你是否也有一个正在运行的`orchestrator`服务? 它将从这里开始, 询问这个实例的主站和副本, 递归进行, 直到整个拓扑结构被揭示.

> This discovers a single instance. But: do you also have an `orchestrator` service running? It will pick up from there and will interrogate this instance for its master and replicas, recursively moving on until the entire topology is revealed.

#### Information
我们现在假设你有被`orchestrator`知道的拓扑结构（你已经发现了它）. 假设`some.mysql.instance.com`属于一个拓扑结构. `a.replica.3.instance.com`属于另一个. 你可以问以下问题.

> We now assume you have topologies known to `orchestrator` (you have *discovered* it). Let's say `some.mysql.instance.com` belongs to one topology. `a.replica.3.instance.com` belongs to another. You may ask the following questions:

```bash
$ orchestrator-client -c clusters
topology1.master.instance.com:3306
topology2.master.instance.com:3306

$ orchestrator-client -c which-master -i some.mysql.instance.com
some.master.instance.com:3306

$ orchestrator-client -c which-replicas -i some.mysql.instance.com
a.replica.instance.com:3306
another.replica.instance.com:3306

$ orchestrator-client -c which-cluster -i a.replica.3.instance.com
topology2.master.instance.com:3306

$ orchestrator-client -c which-cluster-instances -i a.replica.3.instance.com
topology2.master.instance.com:3306
a.replica.1.instance.com:3306
a.replica.2.instance.com:3306
a.replica.3.instance.com:3306
a.replica.4.instance.com:3306
a.replica.5.instance.com:3306
a.replica.6.instance.com:3306
a.replica.7.instance.com:3306
a.replica.8.instance.com:3306

$ orchestrator-client -c topology -i a.replica.3.instance.com
topology2.master.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.1.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.2.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
  + a.replica.3.instance.com:3306 [OK,5.6.17-log,STATEMENT]
  + a.replica.4.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
  + a.replica.5.instance.com:3306 [OK,5.6.17-log,STATEMENT]
+ a.replica.6.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
  + a.replica.7.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.8.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]

```
#### Move stuff around
您可以使用各种命令来移动服务器. 通用的“自动解决问题”命令是 `relocate` 和 `relocate-replicas`：

> You may move servers around using various commands. The generic "figure things out automatically" commands are `relocate` and `relocate-replicas`:

```Plain Text
# Move a.replica.3.instance.com to replicate from a.replica.4.instance.com

$ orchestrator-client -c relocate -i a.replica.3.instance.com:3306 -d a.replica.4.instance.com
a.replica.3.instance.com:3306<a.replica.4.instance.com:3306

$ orchestrator-client -c topology -i a.replica.3.instance.com
topology2.master.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.1.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.2.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
  + a.replica.4.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
    + a.replica.3.instance.com:3306 [OK,5.6.17-log,STATEMENT]
  + a.replica.5.instance.com:3306 [OK,5.6.17-log,STATEMENT]
+ a.replica.6.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
  + a.replica.7.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.8.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]

# Move the replicas of a.replica.2.instance.com to replicate from a.replica.6.instance.com

$ orchestrator-client -c relocate-replicas -i a.replica.2.instance.com:3306 -d a.replica.6.instance.com
a.replica.4.instance.com:3306
a.replica.5.instance.com:3306

$ orchestrator-client -c topology -i a.replica.3.instance.com
topology2.master.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.1.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.2.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.6.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
  + a.replica.4.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
    + a.replica.3.instance.com:3306 [OK,5.6.17-log,STATEMENT]
  + a.replica.5.instance.com:3306 [OK,5.6.17-log,STATEMENT]
  + a.replica.7.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.8.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]

```
`relocate`和`relocate-replicas`会自动计算出如何重新指向一个副本. 也许是通过GTID; 也许是正常的binlog file:pos. 或者也许有Pseudo GTID, 或者有一个binlog server参与？也支持其他变化.

If you want to have greater control:

* Normal file:pos operations are done via `move-up`, `move-below`
* Pseudo-GTID specific replica relocation, use `match`, `match-replicas`, `regroup-replicas`.
* Binlog server operations are typically done with `repoint`, `repoint-replicas`

#### Replication control
You are easily able to see what the following do:

```Plain Text
$ orchestrator-client -c stop-replica -i a.replica.8.instance.com
$ orchestrator-client -c start-replica -i a.replica.8.instance.com
$ orchestrator-client -c restart-replica -i a.replica.8.instance.com
$ orchestrator-client -c set-read-only -i a.replica.8.instance.com
$ orchestrator-client -c set-writeable -i a.replica.8.instance.com

```
Break replication by messing with a replica's master host:

```bash
$ orchestrator-client -c detach-replica -i a.replica.8.instance.com
```
Don't worry, this is reversible:

```bash
$ orchestrator-client -c reattach-replica -i a.replica.8.instance.com
```
#### Crash analysis & recovery
Are your clusters healthy?

```Plain Text
$ orchestrator-client -c replication-analysis
some.master.instance.com:3306 (cluster some.master.instance.com:3306): DeadMaster
a.replica.6.instance.com:3306 (cluster topology2.master.instance.com:3306): DeadIntermediateMaster

$ orchestrator-client -c topology -i a.replica.6.instance.com
topology2.master.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.1.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.2.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.6.instance.com:3306 [last check invalid,5.6.17-log,STATEMENT,>>]
  + a.replica.4.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
    + a.replica.3.instance.com:3306 [OK,5.6.17-log,STATEMENT]
  + a.replica.5.instance.com:3306 [OK,5.6.17-log,STATEMENT]
  + a.replica.7.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.8.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]

```
Ask `orchestrator` to recover the above dead intermediate master:

```Plain Text
$ orchestrator-client -c recover -i a.replica.6.instance.com:3306
a.replica.8.instance.com:3306

$ orchestrator-client -c topology -i a.replica.8.instance.com
topology2.master.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.1.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.2.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
+ a.replica.6.instance.com:3306 [last check invalid,5.6.17-log,STATEMENT,>>]
+ a.replica.8.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
  + a.replica.4.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]
    + a.replica.3.instance.com:3306 [OK,5.6.17-log,STATEMENT]
  + a.replica.5.instance.com:3306 [OK,5.6.17-log,STATEMENT]
  + a.replica.7.instance.com:3306 [OK,5.6.17-log,STATEMENT,>>]

```
#### More
The above should get you up and running. For more please consult the [TOC](https://github.com/Fanduzi/orchestrator-chn-doc/blob/master/TOC.md). For CLI commands listing just run:

```Plain Text
orchestrator-client -help
```
