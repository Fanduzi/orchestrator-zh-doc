# Executing via command line
# [Executing via command line](https://github.com/openark/orchestrator/blob/master/docs/executing-via-command-line.md)
另请参阅[First Steps with Orchestrator](Quick%20guides/First%20Steps.md).

`orchestrator`支持两种从命令行运行操作的方式:

* 使用`orchestrator` 二进制文件(即orchestrator命令, 也是本文的主题)
   * 你将在运维/应用服务器上部署`orchestrator` (命令), 但不将其作为服务运行
   * 您将为`orchestrator`二进制文件部署配置文件, 以便能够连接到(orchestrator)后端数据库.
* 使用[orchestrator-client](Use/orchestrator-client.md)脚本.
   * 你只需要在运维/应用服务器上部署`orchestrator-client` 脚本即可.
   * 你即不需要配置文件, 也不需要二进制文件
>  You will not need any config file nor binaries.
   * 你需要设置`ORCHESTRATOR_API` 环境变量.

两者（大部分）是兼容的. 本文件讨论的是第一种选择(使用orchestrator命令).

以下是命令行示例的概要. 为简单起见, 我们假设 `orchestrator`命令路径已经配置到了`PATH` 中, 如果没有, 请用 `/path/to/orchestrator` 替换`orchestrator` .

> 下面的例子使用了一个测试的mysqlsandbox拓扑结构, 其中所有的实例都在同一个主机127.0.0.1上, 并且在不同的端口. 22987是主库, 22988、22989、22990是从库.

显示当前已知的集群（复制拓扑结构）:

```bash
orchestrator -c clusters
```
> 以上在 `/etc/orchestrator.conf.json`、`conf/orchestrator.conf.json`、`orchestrator.conf.json` 中按顺序查找配置. 通常是将配置放在`/etc/orchestrator.conf.json` 中.  由于它包含您的 MySQL 服务器的凭据, 您可能希望限制对该文件的访问.

您可以选择为配置文件使用不同的位置, 在这种情况下执行:

```bash
orchestrator -c clusters --config=/path/to/config.file
```
> `-c`代表`command`, 是必要参数.

发现一个新的实例（"教"`orchestrator`了解你的拓扑结构）. `Orchestrator`将自动递归地向上钻取主链（如果有的话）和向下钻取复制链（如果有的话）以检测整个拓扑结构

```bash
orchestrator -c discover -i 127.0.0.1:22987
```
> `-i` 代表`instance` , 必须是`hostname:port` 的形式.

用上面的命令相同, 但是包含更多信息:

```bash
orchestrator -c discover -i 127.0.0.1:22987 --debug
orchestrator -c discover -i 127.0.0.1:22987 --debug --stack
```
>  `--debug`在所有操作中都很有用. `--stack`在（大多数）错误上打印代码堆栈跟踪, 对于开发和测试目的或提交错误报告很有用

忘记一个实例（一个实例可以通过上面的discover命令手动或自动重新发现）

```bash
orchestrator -c forget -i 127.0.0.1:22987
```
打印拓扑实例的ASCII树. 通过`-i`传递一个集群名称（见上面的`clusters`命令）

```bash
orchestrator -c topology -i 127.0.0.1:22987
```
> Sample output:

```Plain Text
127.0.0.1:22987
+ 127.0.0.1:22989
  + 127.0.0.1:22988
+ 127.0.0.1:22990
```
在拓扑结构中移动副本:

```bash
orchestrator -c relocate -i 127.0.0.1:22988 -d 127.0.0.1:22987
```
> Resulting topology:

```Plain Text
127.0.0.1:22987
+ 127.0.0.1:22989
+ 127.0.0.1:22988
+ 127.0.0.1:22990
```
上面的情况是将复制体向上移动了一级. 然而, `relocate`命令接受任何有效的destination. `relocate`找出移动副本的最佳方式. 如果GTID被启用, 使用它. 如果Pseudo-GTID可用, 就使用它. 如果涉及binlog server, 则使用它. 如果`orchestrator`对所涉及的具体坐标有进一步的洞察力, 就使用它. 否则就使用普通的基于binlog file:pos的方式.

>  I `orchestrator` has further insight into the specific coordinates involved, use it.

与`relocate`类似, 你可以通过`relocate-replicas`移动多个副本. 这将把一个实例的副本移动到另一个服务器下面

> 假设:

```bash
10.0.0.1:3306
+ 10.0.0.2:3306
  + 10.0.0.3:3306
  + 10.0.0.4:3306
  + 10.0.0.5:3306
+ 10.0.0.6:3306
```
```bash
orchestrator -c relocate-replicas -i 10.0.0.2:3306 -d 10.0.0.6
```
> 结果:

```bash
10.0.0.1:3306
+ 10.0.0.2:3306
+ 10.0.0.6:3306
  + 10.0.0.3:3306
  + 10.0.0.4:3306
  + 10.0.0.5:3306
```
> 你可以使用`--pattern` 来匹配所需的副本.

其他命令让你对服务器的重新定位方式有更精细的控制. 考虑一下经典的基于file:pos的方式来重新指定副本.

将一个副本在拓扑结构中向上移动（使其成为其主副本，或其 "祖先 "的直接副本）.

```bash
orchestrator -c move-up -i 127.0.0.1:22988
```
上述命令只有在实例有祖先, 并且没有复制滞后等问题时才会成功

将副本移动到其同级下方:

```bash
orchestrator -c move-below -i 127.0.0.1:22988 -d 127.0.0.1:22990 --debug
```
> 上面的命令只有在 127.0.0.1:22988 和 127.0.0.1:22990 是兄弟姐妹（同一个 master 的副本）时才会成功, 它们都没有问题（例如副本滞后）, 并且兄弟姐妹可以称为一个新的 master（即有二进制 日志，有 log\_slave\_updates，没有版本冲突等）

让一个实例只读或可写:

```bash
orchestrator -c set-read-only -i 127.0.0.1:22988
orchestrator -c set-writeable -i 127.0.0.1:22988
```
