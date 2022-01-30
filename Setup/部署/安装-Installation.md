# 安装-Installation
# [Installation](https://github.com/openark/orchestrator/blob/master/docs/install.md)
关于生产环境部署, 见[在生产环境中部署Orchestrator](Deployment/在生产环境中部署Orchestrator.md). 下面的文字将引导你通过手动方式安装和必要的配置来使其工作.

以下内容假设您将使用同一台机器来运行`orchestrator`和后端MySQL数据库. 如果不是, 请用适当的主机名替换`127.0.0.1`. 将`orch_backend_password`替换为您自己的密码.

#### Extract orchestrator binary and files
* Extract from tarball
从[https://github.com/openark/orchestrator/releases](https://github.com/openark/orchestrator/releases) 下载tar包. 例如, 假设你想在`/usr/local/orchestrator`下安装`orchestrator` :

```bash
sudo mkdir -p /usr/local
sudo cd /usr/local
sudo tar xzfv orchestrator-1.0.tar.gz
```
* Install from `RPM` 
会安装到`/usr/local/orchestrator` 

```bash
sudo rpm -i orchestrator-1.0-1.x86_64.rpm
```
* Install from `DEB` 
会安装到`/usr/local/orchestrator` 

```bash
sudo dpkg -i orchestrator_1.0_amd64.deb
```
* Install from repository
`orchestrator` packages can be found in [https://packagecloud.io/github/orchestrator](https://packagecloud.io/github/orchestrator)



#### Setup backend MySQL server
设置一个MySQL作为后端, 执行以下命令:

```sql
CREATE DATABASE IF NOT EXISTS orchestrator;
CREATE USER 'orchestrator'@'127.0.0.1' IDENTIFIED BY 'orch_backend_password';
GRANT ALL PRIVILEGES ON `orchestrator`.* TO 'orchestrator'@'127.0.0.1';
```
`Orchestrator`使用一个配置文件, 位于`/etc/orchestrator.conf.json`或`/${orchestrator软件路径}/conf/orchestrator.conf.json` 或 `/${orchestrator软件路径}/conf/orchestrator.conf.json`.

提示: 安装的软件包包括一个名为`orchestrator.conf.json.sample`的文件, 其中有一些基本设置, 你可以将其作为`orchestrator.conf.json`的基线. 它可以在`/usr/local/orchestrator/orchestrator-sample.conf.json`中找到, 你也可以找到`/usr/local/orchestrator/orchestrator-sample-sqlite.conf.json`, 它有一个面向SQLite的配置. 这些样本文件也可以在[orchestrator资源库](https://github.com/openark/orchestrator/tree/master/conf)中找到.

Edit `orchestrator.conf.json` to match the above as follows:

```yaml
...
"MySQLOrchestratorHost": "127.0.0.1",
"MySQLOrchestratorPort": 3306,
"MySQLOrchestratorDatabase": "orchestrator",
"MySQLOrchestratorUser": "orchestrator",
"MySQLOrchestratorPassword": "orch_backend_password",
...
```
#### Grant access to orchestrator on all your MySQL servers
为了使`orchestrator`能够检测到你的复制拓扑结构, 还必须在每个拓扑结构上拥有一个账户. 所有的拓扑结构都必须是同一个账户（同一个用户，同一个密码）. 在你的每个主库上, 发布以下内容

```sql
CREATE USER 'orchestrator'@'orch_host' IDENTIFIED BY 'orch_topology_password';
GRANT SUPER, PROCESS, REPLICATION SLAVE, RELOAD ON *.* TO 'orchestrator'@'orch_host';
GRANT SELECT ON mysql.slave_master_info TO 'orchestrator'@'orch_host';
GRANT SELECT ON ndbinfo.processes TO 'orchestrator'@'orch_host'; -- Only for NDB Cluster
```
> `REPLICATION SLAVE` 对于执行`SHOW SLAVE HOSTS` 和 扫描二进制日志以支持[Pseudo GTID](Various/Pseudo%20GTID.md)是必须的,

> `RELOAD` 对于执行`RESET SLAVE` 是必须的

> `PROCESS` 对于执行`SHOW PROCESSLIST` 是必须的(5.6和以上版本). 如果`master_info_repository = 'TABLE'` 请给orchestrator访问`mysql.slave_master_info` 的权限. 这将允许orchestrator在需要时抓取复制凭证.

Replace `orch_host` with hostname or orchestrator machine (or do your wildcards thing). Choose your password wisely. Edit `orchestrator.conf.json` to match:

```yaml
"MySQLTopologyUser": "orchestrator",
"MySQLTopologyPassword": "orch_topology_password",
```
Consider moving `conf/orchestrator.conf.json` to `/etc/orchestrator.conf.json` (both locations are valid)

要在命令行模式或仅在HTTP API中执行`orchestrator`, 你所需要的只是`orchestrator`二进制文件. 要享受丰富的网络界面, 包括拓扑可视化和拖放拓扑变化, 你将需要`resources`目录和它下面的所有内容. 如果你不确定, 不要碰, 东西已经到位了.

> To execute `orchestrator` in command line mode or in HTTP API only, all you need is the `orchestrator` binary. To enjoy the rich web interface, including topology visualizations and drag-and-drop topology changes, you will need the `resources` directory and all that is underneath it. If you're unsure, don't touch; things are already in place.