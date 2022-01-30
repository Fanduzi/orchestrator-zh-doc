# Tags
# [Tags](https://github.com/openark/orchestrator/blob/master/docs/tags.md)
`orchestrator`支持对实例打标签并通过标签进行搜索.

标签是作为一种服务提供给用户的, 并不被`orchestrator`内部使用.

### Tag commands
支持以下命令. 细目如下:

* `orchestrator-client -c tag -i some.instance --tag name=value`
* `orchestrator-client -c tag -i some.instance --tag name`
* `orchestrator-client -c untag -i some.instance -t name`
* `orchestrator-client -c untag-all -t name=value`
* `orchestrator-client -c tags -i some.instance`
* `orchestrator-client -c tag-value -i some.instance -t name`
* `orchestrator-client -c tagged -t name`
* `orchestrator-client -c tagged -t name=value`
* `orchestrator-client -c tagged -t name=`

and these API endpoints:

* `api/tag/:host/:port/:tagName/:tagValue`
* `api/tag/:host/:port?tag=name`
* `api/tag/:host/:port?tag=name%3Dvalue`
* `api/untag/:host/:port/:tagName`
* `api/untag/:host/:port?tag=name`
* `api/untag-all/:tagName/:tagValue`
* `api/untag-all?tag=name%3Dvalue`
* `api/tags/:host/:port`
* `api/tag-value/:host/:port/:tagName`
* `api/tag-value/:host/:port?tag=name`
* `api/tagged?tag=name`
* `api/tagged?tag=name%3Dvalue`
* `api/tagged?tag=name%3D`

### Tags, general
一个标签可以是`name=value`的形式, 也可以是`name`的形式, 在这种情况下, 值被隐含地设置为空字符串. `name`可以是以下格式

* `word` 
* `some-other-word` 
* `some_word_word_yet` 

虽然没有严格限制, 但要避免使用特殊字符/标点符号.

### Tagging
`-c tag`或`api/tag`向实例添加或替换现有的标签. `orchestrator`不会指示该标签是否事先存在, 也不会提供以前的值（如果有）.

Example:

```bash
$ orchestrator-client -c tag -i db-host-01:3306 --tag vttablet_alias=dc1-0123456789
```
在上面的例子中, 我们选择了创建一个名为`vttablet_alias`的标签, 并带有一个值.

标签是打在实例进行上的. 实例本身不受此操作的影响. `orchestrator`将标签作为元数据进行维护. 实例不需要可用.

### Untagging: single instance
`-c untag`或`api/untag`从一个给定的实例中删除一个标签（如果存在的话）. 如果标签确实存在, `orchestrator`会输出实例名称, 如果标签不存在, 则输出空.

You may tags:

* 指定标签名称和标签值: 标签只有在等于该值时才被删除
* 只指定标签名称: 标签被删除, 而不考虑其价值

Example:

```bash
$ orchestrator-client -c untag -i db-host-01:3306 --tag vttablet_alias
```
### Untagging: multiple instances
`-c untag-all` 或 `api/untag-all` 会从值匹配的所有实例中删除一个标签. 注意, 必须提供标签值.

Example:

```bash
$ orchestrator-client -c untag-all --tag vttablet_alias=dc1-0123456789
```
### Listing instance tags
对于一个给定的实例, `-c tags` or `api/tags`列出所有已知的标签.

Example:

```bash
$ orchestrator-client -c tag -i db-host-01:3306 --tag vttablet_alias=dc1-0123456789
$ orchestrator-client -c tag -i db-host-01:3306 --tag old-hardware

$ orchestrator-client -c tags -i db-host-01:3306
old-hardware=
vttablet_alias=dc1-0123456789
```
列出的标签是按名称排序的. 请注意, 我们添加了不带值的`old-hardware`标签. 它以`old-hardware=`的形式导出, 并隐含空值.

### Listing instance tags for a cluster
对于一个给定的实例或集群别名, `-c topology-tags`或`api/topology-tags`列出了集群拓扑结构与每个实例的所有已知标签.

Example:

```bash
$ orchestrator-client -c tag -i db-host-01:3306 --tag vttablet_alias=dc1-0123456789
$ orchestrator-client -c tag -i db-host-01:3306 --tag old-hardware

$ orchestrator-client -c topology-tags -alias mycluster
db-host-01:3306     [0s,ok,5.7.23-log,rw,ROW,>>,GTID,P-GTID] [vttablet_alias=dc1-0123456789, old-hardware]
+ db-host-02:3306   [0s,ok,5.7.23-log,ro,ROW,>>,GTID,P-GTID] []

$ orchestrator-client -c topology-tags -i db-host-01:3306
db-host-01:3306     [0s,ok,5.7.23-log,rw,ROW,>>,GTID,P-GTID] [vttablet_alias=dc1-0123456789, old-hardware]
+ db-host-02:3306   [0s,ok,5.7.23-log,ro,ROW,>>,GTID,P-GTID] []
```
### Getting the value of a specific tag
`-c tag-value` 或 `api/tag-value` 返回一个实例上特定标签的值

Example:

```bash
$ orchestrator-client -c tag -i db-host-01:3306 --tag vttablet_alias=dc1-0123456789
$ orchestrator-client -c tag -i db-host-01:3306 --tag old-hardware
$ orchestrator-client -c tag-value -i db-host-01:3306 --tag vttablet_alias
dc1-0123456789
$ orchestrator-client -c tag-value -i db-host-01:3306 --tag old-hardware

# <empty value>
$ orchestrator-client -c tag-value -i db-host-01:3306 --tag no-such-tag
tag no-such-tag not found for db-host-01:3306
# in stderr
```
### Searching instances by tags
`-c tagged` 或 `api/tagged` 按标签列出实例, 如下所示.

* `-c tagged -tag name=value`: 列出`name`存在且等于`value`的实例.
* `-c tagged -tag name`: 列出存在这个`name` 的实例, 无论`value`是什么.
* `-c tagged -tag name=`: 列出`name`存在且值为空的实例.
* `-c tagged -tag name,role=backup`: 列出包含`name`标签(无论其值如何), 并且也用 `role=backup` 标记的实例.
* `-c tagged -tag !name`: 列出不存在名为 `name` 的标签的实例, 无论其值如何
* `-c tagged -tag ~name`: `~` 是 `!` 的同义词.
* `-c tagged -tag name,~role`: 列出包含`name`标签(无论其值如何), 并且不包含`role`标签(无论其值如何)的实例.
* `-c tagged -tag ~role=backup`: 列出包含`role` 标签, 但`value` 不是`backup` 的实例. 请注意这与 `-c tagged -tag ~role` 有何不同. 后者将首先列出没有`role`标签的实例.

### Tags, internal
标签与实例关联, 但关联是记录在`orchestrator`内部的, 不会影响实际的 MySQL 实例.  标签内部存储在(orchestrator的)后端数据库表中. 标签由 `raft` 协议发布; 由`raft` leader执行的标记操作(`tag`, `untag`)将被应用于`raft` followers.

### Use cases
对标签的需求来自不同用户的不同使用情况.

* [Vitess](http://github.com/vitess.io/vitess)用户的一个常见用例是需要将一个实例与`vttablet` alias联系起来.
* 用户可能希望根据标签来应用`promotion`逻辑. 虽然`orchestrator`在任何决策中都不使用内部标记, 但用户可以根据标记设置`promotion-rule`, 或应用不同的failover操作.





