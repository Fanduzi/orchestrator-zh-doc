# orchestrator-client
# [orchestrator-client](https://github.com/openark/orchestrator/blob/master/docs/orchestrator-client.md)
[orchestrator-client](https://github.com/openark/orchestrator/blob/master/resources/bin/orchestrator-client)是一个脚本, 用方便的命令行界面包装API调用.

它可以自动确定`orchestrator`集群的leader, 并在这种情况下将所有请求转发给leader.

它非常接近于`orchestrator` command line interface.

使用`orchestrator-client`, 您:

* 不需要到处安装`orchestrator`二进制文件; 仅在服务运行的主机上安装即可.
* 不需要在所有地方部署`orchestrator`配置(配置文件); 只需要服务运行的主机上部署即可.
* 不需要访问后端数据库
* 需要访问 HTTP api
* 需要设置 `ORCHESTRATOR_API` 环境变量.
   * 要么为代理提供单个端点, 例如

```bash
export ORCHESTRATOR_API=https://orchestrator.myservice.com:3000/api
```
   * 或者提供所有`orchestrator`端点, `orchestrator-client`将自动选择leader（不需要代理）, 例如

```bash
export ORCHESTRATOR_API="https://orchestrator.host1:3000/api https://orchestrator.host2:3000/api https://orchestrator.host3:3000/api"
```
* 您可以在 `/etc/profile.d/orchestrator-client.sh` 中设置环境变量. 如果此文件存在, 它将被 `orchestrator-client` 内联(应该是使用的意思, 就是`orchestrator-client` 脚本里使用了`/etc/profile.d/orchestrator-client.sh` )
>  You may set up the environment in `/etc/profile.d/orchestrator-client.sh`. If this file exists, it will be inlined by `orchestrator-client`.

### Sample usage
显示当前已知的集群（复制拓扑结构）:

```bash
orchestrator-client -c clusters
```
发现, 遗忘一个实例:

```bash
orchestrator-client -c discover -i 127.0.0.1:22987
orchestrator-client -c forget -i 127.0.0.1:22987
```
打印拓扑实例的ASCII树. 通过`-i`传递一个集群名称(见上面的集群命令):

```bash
orchestrator-client -c topology -i 127.0.0.1:22987
```
> Sample output:

```python
127.0.0.1:22987
+ 127.0.0.1:22989
  + 127.0.0.1:22988
+ 127.0.0.1:22990
```
在拓扑结构中移动副本:

```bash
orchestrator-client -c relocate -i 127.0.0.1:22988 -d 127.0.0.1:22987
```
> Resulting topology:

```bash
127.0.0.1:22987
+ 127.0.0.1:22989
+ 127.0.0.1:22988
+ 127.0.0.1:22990
```
等等.

### Behind the scenes 幕后花絮
命令行界面为API调用提供了一个很好的包装, 其输出被从JSON格式转换为文本格式.

例如, 命令:

```bash
orchestrator-client -c discover -i 127.0.0.1:22987
```
被转化为（为方便起见在此简化）

> Translates to (simplified here for convenience):

```bash
curl "$ORCHESTRATOR_API/discover/127.0.0.1/22987" | jq '.Details | .Key'
```
### Meta commands
* `orchestrator-client -c help` : 列出所有可用的命令.
* `orchestrator-client -c which-api` : 输出`orchestrator-client`用来调用命令的API端点. 当通过`$ORCHESTRATOR_API`提供多个端点时, 这很有用.
* `orchestrator-client -c api -path clusters` : 调用一个generic HTTP API call（in this case `clusters`）并返回原始的JSON响应.











