# Execution
# [Execution](https://github.com/openark/orchestrator/blob/master/docs/execution.md)
#### Executing as web/API service
假设你已经在`/usr/local/orchestrator`下安装了`orchestrator` :

```bash
cd /usr/local/orchestrator && ./orchestrator http
```
`Orchestrator`将开始监听`3000`端口. 将你的浏览器指向`http://your.host:3000/`, 你就可以开始了. 你可以跳到下一节.

如果您喜欢调试消息:

```bash
cd /usr/local/orchestrator && ./orchestrator --debug http
```
或者, 在出现错误时显示更详细的信息:

```bash
cd /usr/local/orchestrator && ./orchestrator --debug --stack http
```
以上在 `/etc/orchestrator.conf.json`、`conf/orchestrator.conf.json`、`orchestrator.conf.json` 中按顺序查找配置. 通常是将配置放在`/etc/orchestrator.conf.json` 中.  由于它包含您的 MySQL 服务器的凭据, 您可能希望限制对该文件的访问.  您可以选择为配置文件使用不同的位置, 在这种情况下执行:

```bash
cd /usr/local/orchestrator && ./orchestrator --debug --config=/path/to/config.file http
```
默认情况下, Web/API 服务将对所有已知服务器发出连续、无限的轮询.  这使`orchestrator`的数据保持最新. 你通常想要这种行为. 但你可以禁用它, 使`orchestrator`只为 API/Web 提供服务, 但从不更新实例状态

```bash
cd /usr/local/orchestrator && ./orchestrator --discovery=false http
```
上述内容对开发和测试很有用. 你可能希望保持默认状态

>  The above is useful for development and testing purposes. You probably wish to keep to the defaults.