# Status Checks
# [Status Checks](https://github.com/openark/orchestrator/blob/master/docs/status-checks.md)
There is a status endpoint located at `/api/status`, 它对系统进行健康检查, 如果一切正常, 则返回 HTTP 状态代码 200.  否则, 它会返回 HTTP 状态代码 500.

#### Custom Status Checks
由于公司可能会对其状态检查端点使用各种标准, 因此您可以通过设置进行自定义:

```json
{
  "StatusEndpoint": "/my_status"
}
```
或者你想要的任何endpoint.

#### Lightweight Health Check
这个状态检查是一个非常轻量级的检查, 因为我们假设您的负载均衡器可能会频繁地访问它或其他一些频繁的监控。. 如果您想要更丰富的检查来实际更改数据库, 您可以使用以下命令进行设置:

>  This status check is a very lightweight check because we assume your load balancer might be hitting it frequently or some other frequent monitoring. If you want a richer check that actually makes changes to the database you can set that with:

```json
{
  "StatusSimpleHealth": false
}
```
#### SSL Verification
最后, 如果您使用 SSL/TLS 运行, 我们不需要状态检查来提供有效的 OU 或客户端证书. 如果您正在使用更丰富的检查并希望打开验证,您可以设置:

> Lastly if you run with SSL/TLS we *don't* require the status check to have a valid OU or client cert to be presented. If you're using that richer check and would like to have the verification turned on you can set:

```json
{
  "StatusOUVerify": true
}
```
