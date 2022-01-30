# Configuration: Backend
# [Configuration: backend](https://github.com/openark/orchestrator/blob/master/docs/configuration-backend.md)
让orchestrator知道在哪里可以找到后端数据库. 在此设置中, `orchestrator`将在端口:3000上提供HTTP服务

```yaml
{
  "Debug": false,
  "ListenAddress": ":3000",
}
```
你可以选择`MySQL`后端或`SQLite`后端. 参见[[Orchestrator高可用 id=&#39;ed5e7b21-c508-44b8-817e-d7c782082cf3&#39;]], 了解使用这两种方式的场景、可能性和原因。

## MySQL backend
你需要设置提供给`orchestrator` 使用的库名和用户密码:

```yaml
{
  "MySQLOrchestratorHost": "orchestrator.backend.master.com",
  "MySQLOrchestratorPort": 3306,
  "MySQLOrchestratorDatabase": "orchestrator",
  "MySQLOrchestratorCredentialsConfigFile": "/etc/mysql/orchestrator-backend.cnf",
}
```
`MySQLOrchestratorCredentialsConfigFile` 示例:

```Plain Text
[client]
user=orchestrator_srv
password=${ORCHESTRATOR_PASSWORD}
```
其中`user`或`password`可以是明文, 也可以是环境变量.

另外, 你可以选择在配置文件中使用纯文本形式的用户名和密码.

```yaml
{
  "MySQLOrchestratorUser": "orchestrator_srv",
  "MySQLOrchestratorPassword": "orc_server_password",
}
```
#### MySQL backend DB setup
对于MySQL后端数据库, 你将需要授予必要的权限.

```sql
CREATE USER 'orchestrator_srv'@'orc_host' IDENTIFIED BY 'orc_server_password';
GRANT ALL ON orchestrator.* TO 'orchestrator_srv'@'orc_host';
```
## SQLite backend
默认使用的后端数据库是`MySQL`. 要设置使用`SQLite`, 请设置:

```yaml
{
  "BackendDB": "sqlite",
  "SQLite3DataFile": "/var/lib/orchestrator/orchestrator.db",  
}
```
`SQLite`被嵌入到`orchestrator`中.

> `SQLite` is embedded within `orchestrator`. 

如果`SQLite3DataFile`参数指定的文件不存在, `orchestrator`将创建它. `orchestrator`需要对给定的路径/文件有写权限.











