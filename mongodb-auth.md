---
title: MongoDB 3.4 的用户认证
date: 2018-03-03 10:08:03
tags: [mongodb]
categories: 学习笔记
---

### 前言

> 一般使用数据库都是习惯设置各类权限的账号进行操作，MongoDB支持基于角色的访问控制（RBAC）来管理对MongoDB系统的访问。一个用户可以被授权一个或者多个:ref:角色  以决定该用户对数据库资源和操作的访问权限。在权限以外，用户是无法访问系统的。
>
> 数据库角色在创建用户中的role参数中设置。角色分为内建角色和自定义角色。

### window 上使用
#### 新建一个数据库
新建一个名为 `learn` 的数据库： 
```
> use learn
switched to db learn
> show dbs
admin  0.000GB
local  0.000GB
> db.zhangkai.insert({"wuwii":"learn"})
WriteResult({ "nInserted" : 1 })
> show dbs
admin  0.000GB
learn  0.000GB
local  0.000GB
```
#### 创建用户相关信息
启动安装目录下 `/bin/mongo.exe` ，进入 MongoDB 环境模式，或者在 `powershell` 中直接输入 `mongo`。
```json
> use admin
switched to db admin
> db.createUser({"user":"wuwii","pwd":"123456","roles":["root"]})
Successfully added user: { "user" : "wuwii", "roles" : [ "root" ] }
> show dbs
admin  0.000GB
learn  0.000GB
local  0.000GB
> db.getCollectionNames()
[ "system.users", "system.version" ]
> use learn
switched to db learn
> db.createUser({"user":"wuwii","pwd":"123456","roles":[{role:"dbOwner",db:"learn"}]})
Successfully added user: {
        "user" : "wuwii",
        "roles" : [
                {
                        "role" : "dbOwner",
                        "db" : "learn"
                }
        ]
}
>
```
上面使用 `admin` 数据库 创建一个管理员 `wuwii`，后来我新建了一个数据库 `learn`，给它也新建一个用户 `wuwii`，它在 `learn`数据库的角色，也就是权限是 `dbOwner`，拥有者，最高权限，还可以设置 `read`，`readWrite`的读写权限等。


查看所有用户：
```json
> db.system.users.find()
{ "_id" : "admin.wuwii", "user" : "wuwii", "db" : "admin", "credentials" : { "SCRAM-SHA-1" : { "iterationCount" : 10000, "salt" : "Xu7Ez3U20+CGwQABVNen+g==", "storedKey" : "3eot4z0Epi3DaNROkyfFjz41RA8=", "serverKey" : "Qy6AcacWGbEB1J0LMPSqT/RKdcI=" } }, "roles" : [ { "role" : "root", "db" : "admin" } ] }
{ "_id" : "learn.wuwii", "user" : "wuwii", "db" : "learn", "credentials" : { "SCRAM-SHA-1" : { "iterationCount" : 10000, "salt" : "C+YReZNGaaMh8BsVd/uTQQ==", "storedKey" : "FjOBuGuHtmr4tZGeAY8c5LBhj2U=", "serverKey" : "Xs31qu6H0g8b/bqe/eZOEMMEEM8=" } }, "roles" : [ { "role" : "dbOwner", "db" : "learn" } ] }
```

#### 开启权限认证
启动 mongo 的命令参数加上 `--auth` 就能够开启认证模式，
```
 mongod --dbpath D:\mongodb\data\db --logpath=D:\mongodb\data\log\mongodb.log --logappend --auth  --directoryperdb
```

当我们进入 `mongo` 模式中，提示需要用户认证：
```
> use admin
switched to db admin
> show dbs
2018-03-03T09:59:48.214+0800 E QUERY    [main] Error: listDatabases failed:{
        "ok" : 0,
        "errmsg" : "not authorized on admin to execute command { listDatabases: 1.0 }",
        "code" : 13,
        "codeName" : "Unauthorized"
} :
```
现在我们进入登陆 `admin` 数据库：
```
> db.auth("wuwii","123456")
1
> show dbs
admin  0.000GB
learn  0.000GB
local  0.000GB
```
返回 `1` 表示登陆成功，账户错误会提示你，并且会返回 `0`：
```
Error: Authentication failed.
0
```

一般我们使用驱动认证连接是在 url 中配置
```
mongodb://name:pwd@localhost:27017/database
```
多个IP集群配置:
```
mongodb://name:pwd@ip1:port1,ip2:port2/database
```
#### Linux 上使用
启用安全认证，需要更改配置文件参数 `authorization`，也可以简写为 `auth`。
```
auth = true
```
然后进入 mongo 环境进行跟上面相同的操作。

### 附录

#### MongoDB内建角色包括以下几类
1. 数据库用户角色
```
read：允许用户读取指定数据库
readWrite：允许用户读写指定数据库
```
2. 数据库管理员角色
```
dbAdmin：允许用户进行索引创建、删除，查看统计或访问system.profile，但没有角色和用户管理的权限
userAdmin：提供了在当前数据库中创建和修改角色和用户的能力
dbOwner： 提供对数据库执行任何管理操作的能力。这个角色组合了readWrite、dbAdmin和userAdmin角色授予的特权。
```
3. 集群管理角色
```
clusterAdmin ： 提供最强大的集群管理访问。组合clusterManager、clusterMonitor和hostManager角色的能力。还提供了dropDatabase操作
clusterManager ： 在集群上提供管理和监视操作。可以访问配置和本地数据库，这些数据库分别用于分片和复制
clusterMonitor ： 提供对监控工具的只读访问，例如MongoDB云管理器和Ops管理器监控代理。
hostManager ： 提供监视和管理服务器的能力。
```
4. 备份恢复角色
```
backup ： 提供备份数据所需的能力，使用MongoDB云管理器备份代理、Ops管理器备份代理或使用mongodump
restore ： 提供使用mongorestore恢复数据所需的能力
```
5. 所有数据库角色
```
readAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读权限 
readWriteAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读写权限 
userAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的userAdmin权限 
dbAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的dbAdmin权限。 
```
6. 超级用户角色
```
root：提供对readWriteAnyDatabase、dbAdminAnyDatabase、userAdminAnyDatabase、clusterAdmin、restore和backup的所有资源的访问
```
7. 内部角色
```
__system : 提供对数据库中任何对象的任何操作的特权
```
8. 自定义角色
    除了使用内建的角色之外，MongoDB还支持使用 `db.createRole()` 方法（**只能在admin数据库中创建角色，否则会失败**）来自定义角色:
* role: 自定义角色的名称
* privileges: 权限操作
* roles：继承的角色。如果没有继承的角色，可以设置为空数组　
```json
db.createRole(
   {
     role: "myClusterwideAdmin",
     privileges: [
       { resource: { cluster: true }, actions: [ "addShard" ] },
       { resource: { db: "config", collection: "" }, actions: [ "find", "update", "insert", "remove" ] },
       { resource: { db: "users", collection: "usersCollection" }, actions: [ "update", "insert", "remove" ] },
       { resource: { db: "", collection: "" }, actions: [ "find" ] }
     ],
     roles: [
       { role: "read", db: "admin" }
     ]
   },
   { w: "majority" , wtimeout: 5000 }
)
```
#### 用户管理命令
1. 创建用户
```
db.createUser({user: "...",pwd: "...",customDate:"...",roles:[{role: "...",db: "..."}]})
```
* 使用createUser命令来创建用户
* user: 用户名 
* pwd: 密码
* customData: 对用户名密码的说明(可选项)
* roles: {role:继承自什么角色类型，db:数据库名称}

2. 重新登录数据库，并验证权限
```
db.auth("user","pwd")
```
3. 添加普通用户
    一旦经过认证的用户管理员，可以使用db.createUser()去创建额外的用户。 可以分配mongodb内置的角色或用户自定义的角色给用户。
>管理员用户需要在 admin 数据库才能认证成功，同理，给别的数据库设置的账户，只能在该数据库下才能认证成功，否则认证失败。

4. 查看用户
```
db.system.users.find()
```
5. 删除用户
```
db.dropUser("username")
```
6. 添加用户权限
```
db.grantRolesToUser("username",[{role:"roleName":db:"database"} ……])
```
7. 修改密码
```
db.changeUserPassword("username","pwd")
```

### 参考博客
* [MongoDB安全介绍及配置身份认证](https://www.centos.bz/2017/08/mongodb-secure-intro-user-auth/)
