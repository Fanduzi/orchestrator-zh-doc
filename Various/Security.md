# Security
# [Security](https://github.com/openark/orchestrator/blob/master/docs/security.md)
在HTTP模式（API或Web）下操作时, 可以通过以下两种方式限制对`orchestrator`的访问:

## basic authentication
添加一下内容到`orchestrator` 配置文件:

```yaml
 "AuthenticationMethod": "basic",
 "HTTPAuthUser":         "dba_team",
 "HTTPAuthPassword":     "time_for_dinner"
```
使用`basic` authentication(基本身份验证), 只有一个凭证, 没有角色

`Orchestrator`的配置文件包含了你的MySQL服务器的凭证, 以及上面指定的基本认证凭证. 保持它的安全性（例如`chmod 600`）.

>  with `basic` authentication there's just one single credential, and no roles.

## basic authentication, extended
添加一下内容到`orchestrator` 配置文件:

```yaml
 "AuthenticationMethod": "multi",
 "HTTPAuthUser":         "dba_team",
 "HTTPAuthPassword":     "time_for_dinner"
```
`multi` authentication的工作方式与*basic authentication*类似, 但也接收用户使用`readonly`用户并指定任意密码连接. `readonly`用户被允许查看所有内容, 但不能通过API进行写操作（如停止复制、重新指定(repointing)复制、发现新实例等）.

## Headers authentication
通过反向代理转发的header进行身份验证（例如，Apache2 将请求中继到协调器）.  需要配置：

```yaml
 "AuthenticationMethod": "proxy",
 "AuthUserHeader": "X-Forwarded-User",
```
你需要配置你的反向代理, 通过HTTP header发送认证用户的名字, 并使用与`AuthUserHeader`配置的相同的头名称.

For example, an Apache2 setup may look like the following:

```yaml
 RequestHeader unset X-Forwarded-User
 RewriteEngine On
 RewriteCond %{LA-U:REMOTE_USER} (.+)
 RewriteRule .* - [E=RU:%1,NS]
 RequestHeader set X-Forwarded-User %{RU}e
```


`proxy` authentication允许角色.  一些用户是高级用户(*Power users*), 其余的只是普通用户. 高级用户可以更改拓扑, 而普通用户处于只读模式. 要指定已知 DBA 的列表, 请使用:

```yaml
 "PowerAuthUsers": [
     "wallace", "gromit", "shaun"
     ],
```


或者, 无论如何, 您可以通过以下方式将整个`orchestrator`进程变为只读:

```yaml
    "ReadOnly": "true",
```
你可以将`ReadOnly`与你喜欢的任何认证方法结合起来.
