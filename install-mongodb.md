---
title: 安装 MongoDB
date: 2018-03-1 10:08:03
tags: [mongodb]
categories: 学习笔记
---

### 前言

### Window 上安装 MongoDB
#### 下载
[MongoDB 官网](https://www.mongodb.com/download-center)

官网下载需要翻墙，
网上看到一个人挂在云存储上的链接，拿来用下 [地址](http://oaq0p7t2g.bkt.clouddn.com/mongodb-win32-x86_64-2008plus-ssl-3.4.1-signed.msi?attname=)

<!--more-->

#### 安装
一路安装就行，如果不喜欢安装在 C 盘的朋友，请选择 `custom` 选择路径。
#### 配置环境变量
在 path 中配置安装的 bin 目录
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/mongodb/1.png)
#### 创建数据库文件存放目录
首先创建数据库文件存储的目录，比如我建了一个新的文件夹 `D:\zhangkai\mongodb\data\db`，用来存储 MongoDB 的数据库文件，注意这个文件夹只能手动创建，启动 MongoDB 是不会帮助我们自动创建的，它会报一个错误，提示你找不到该文件夹。

#### 启动
window 下使用 powershell 或者 cmd 进入安装目录的 bin 文件夹下，执行启动命令：
```
mongod --dbpath D:\zhangkai\mongodb\data\db
```

执行完成命令出现连接上了 27017 端口。
```
2018-03-01T14:35:04.988+0800 I NETWORK  [thread1] waiting for connections on port 27017
```
然后可以在 `D:\zhangkai\mongodb\data\db` 目录中查看到一堆文件。
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/mongodb/2.png)

#### 创建日志文件存放目录
我们在使用 MongoDB 的时候需要使用日志，这个时候需要指定 log 目录，所以我们创建 log 目录 `D:\zhangkai\mongodb\data\log\`，
执行启动命令：
```
mongod --dbpath D:\zhangkai\mongodb\data\db --logpath=D:\zhangkai\mongodb\data\log\mongodb.log
```
然后系统帮助我们在日志目录下生成一个 `mongodb.log` 的文件，用来记录日志信息。

再次启用的时候，会自动备份上次的日志文件，新创建一个 `mongodb.log` 的文件。
我们发现带有日志启动后，以前命令行一堆日志信息，将记录到这个日志文件中，命令行是干净的。

如果不想覆盖上个日志需要在上个命令后面加上 `--logappend`
```
mongod --dbpath D:\zhangkai\mongodb\data\db --logpath=D:\zhangkai\mongodb\data\log\mongodb.log --logappend
```
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/mongodb/3.png)
#### 安装为 Windows 服务
每次启动 MongoDB 都输入那么一大串命令很麻烦，而且还容易输入错误，可以将它注册成服务，方便后面使用。

以`管理员`的身份启动 powershell 或 cmd，执行下面的命令：
```
mongod --dbpath D:\zhangkai\mongodb\data\db --logpath=D:\zhangkai\mongodb\data\log\mongodb.log --logappend --directoryperdb --serviceName MongoDB --install
```
参数解释：
* `dbpath`: 数据库文件目录
* `logpath`：日志文件目录
* `logappend`：日志文件以追加的方式输出，而不是新建文件
* `directoryperdb`：每个DB都会新建一个目录
* `serviceName`：window 服务名，我们用指定的名字来启动服务
* `install`：创建服务，相反 `remove`，移除服务，`reinstall` 重新构建

执行完可以在日志文件看到日志信息：
```yaml
2018-03-01T16:27:19.606+0800 I CONTROL [main] Service can be started from the command line with 'net start MongoDB'
```
尝试使用 `net start MongoDB` 启动服务：
```
PS C:\WINDOWS\system32> net start MongoDB
MongoDB 服务正在启动 .
MongoDB 服务已经启动成功。
```
可以使用 `net stop MongoDB` 停止服务。
#### 浏览器测试
浏览器输入链接
[http://127.0.0.1:27017/](http://127.0.0.1:27017/)
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/mongodb/4.png)

### Linux 安装 MongoDB
我使用的是 CentOS 7 ，ubuntu 系统安装方式也差不多。
#### 使用 yum 安装
由于官方镜像在国内被墙，需要设置国内的镜像，我使用的是阿里云的。
1. 最好先更新下软件包，开发人员多多更新软件，很有利的。
```bash
sudo yum -y update
```
2. 编辑一个 mongodb 镜像，版本随便自己选择吧，我使用的是3.4，我之前安装的是 3.2的差别不大
```bash
vim /etc/yum.repos.d/mongodb-org.repo
# 在文件中添加以下镜像内容
[mongodb-org]
name=MongoDB Repository
baseurl=http://mirrors.aliyun.com/mongodb/yum/redhat/7Server/mongodb-org/3.4/x86_64/
gpgcheck=0
enabled=1
```
3. 使用 yum 安装 ` yum install -y mongodb-org`

这样我们就安装完成了
#### 启动方式
1. 例如上面 window 命令行启动，只要在命令后街上相应的参数设置就行了。
```
mongod --dbpath=/opt/mongodb/data/db --logpath=/opt/mongodb/log/mongodb.log
```
2. 使用配置文件启动
```
mongod -f /etc/mongod.conf
```
或者使用
```
mongod --config /etc/mongod.conf
```

其实上面两种启动方式在 window 里也是适用的，可以试一试。
#### 配置文件
配置文件在 ` /etc/mongod.conf `，参考配置文件，资料不全，大多数参数配置暂时也用不到，所以没仔细研究这个东西。


```yaml
systemLog:
   # verbosity: 0  #日志等级，0-5，默认0
   # quiet: false  #限制日志输出，
   # traceAllExceptions: true  #详细错误日志
   # syslogFacility: user #记录到操作系统的日志级别，指定的值必须是操作系统支持的，并且要以--syslog启动
   path: /Users/mhq/projects/db/mongo/logs/log.txt  #日志路径。
   logAppend: false #启动时，日志追加在已有日志文件内还是备份旧日志后，创建新文件记录日志, 默认false
   logRotate: rename #rename/reopen。rename，重命名旧日志文件，创建新文件记录；reopen，重新打开旧日志记录，需logAppend为true
   destination: file #日志输出方式。file/syslog,如果是file，需指定path，默认是输出到标准输出流中
   timeStampFormat: iso8601-local #日志日期格式。ctime/iso8601-utc/iso8601-local, 默认iso8601-local
   # component: #各组件的日志级别
   #    accessControl:
   #       verbosity: <int>
   #    command:
   #       verbosity: <int>

processManagement:
   fork: true #以守护进程运行 默认false
   # pidFilePath: <string> #PID 文件位置

net:
   port: 27017 #监听端口，默认27017
   bindIp: 127.0.0.1 #绑定监听的ip，deb和rpm包里有默认的配置文件(/etc/mongod.conf)里面默认配置为127.0.0.1,若不限制IP，务必确保认证安全，多个Ip用逗号分隔
   maxIncomingConnections: 65536 #最大连接数，可接受的连接数还受限于操作系统配置的最大连接数
   wireObjectCheck: true #校验客户端的请求，防止错误的或无效BSON插入,多层文档嵌套的对象会有轻微性能影响,默认true
   ipv6: false #是否启用ipv6,3.0以上版本始终开启
   unixDomainSocket: #unix socket监听，仅适用于基于unix的系统
      enabled: false #默认true
      pathPrefix: /tmp #路径前缀，默认/temp
      filePermissions: 0700 #文件权限 默认0700
   http: #警告 确保生产环境禁用HTTP status接口、REST API以及JSON API以防止数据暴露和漏洞攻击
      enabled: false #是否启用HTTP接口、启用会增加网络暴露。3.2版本后停止使用HTTP interface
      JSONPEnabled: false #JSONP的HTTP接口
      RESTInterfaceEnabled: false #REST API接口
   # ssl: # ssl 证书的配置，没研究
   #    sslOnNormalPorts: <boolean>  # deprecated since 2.6
   #    mode: <string>
   #    PEMKeyFile: <string>
   #    PEMKeyPassword: <string>
   #    clusterFile: <string>
   #    clusterPassword: <string>
   #    CAFile: <string>
   #    CRLFile: <string>
   #    allowConnectionsWithoutCertificates: <boolean>
   #    allowInvalidCertificates: <boolean>
   #    allowInvalidHostnames: <boolean>
   #    disabledProtocols: <string>
   #    FIPSMode: <boolean>

security: # 开启用户认证模式
   authorization: enabled # enabled/disabled #开启客户端认证
   javascriptEnabled:  true #启用或禁用服务器端JavaScript执行
   # keyFile: <string> #密钥路径
   # clusterAuthMode: <string> #集群认证方式
   # enableEncryption: <boolean>
   # encryptionCipherMode: <string>
   # encryptionKeyFile: <string>
   # kmip:
   #    keyIdentifier: <string>
   #    rotateMasterKey: <boolean>
   #    serverName: <string>
   #    port: <string>
   #    clientCertificateFile: <string>
   #    clientCertificatePassword: <string>
   #    serverCAFile: <string>
   # sasl:
   #    hostName: <string>
   #    serviceName: <string>
   #    saslauthdSocketPath: <string>
   

# setParameter: #设置参数
#    <parameter1>: <value1>
#    <parameter2>: <value2>

storage:
   dbPath: /Users/mhq/projects/db/mongo/test/ #数据库，默认/data/db,如果使用软件包管理安装的查看/etc/mongod.conf
   indexBuildRetry: true #重启时，重建不完整的索引
   # repairPath: <string>  #--repair操作时的临时工作目录，默认为dbPath下的一个_tmp_repairDatabase_<num>的目录
   # 对应 journal 启用操作日志，以确保写入持久性和数据的一致性，会在dbpath目录下创建journal目录
   journal: 
      enabled: true #启动journal,64位系统默认开启，32位默认关闭
      # commitIntervalMs: <num> #journal操作的最大时间间隔，默认100或30
   directoryPerDB: false #使用单独的目录来存储每个数据库的数据,默认false,如果需要更改，要备份数据，删除掉dbPath下的文件，重建后导入数据
   # syncPeriodSecs: 60 #使用fsync来将数据写入磁盘的延迟时间量,建议使用默认值
   engine: wiredTiger #存储引擎，mmapv1/wiredTiger/inMemory 默认wiredTiger
   # mmapv1:
   #    preallocDataFiles: <boolean>
   #    nsSize: <int>
   #    quota:
   #       enforced: <boolean>
   #       maxFilesPerDB: <int>
   #    smallFiles: <boolean>
   #    journal:
   #       debugFlags: <int>
   #       commitIntervalMs: <num>
   # wiredTiger:
   #    engineConfig:
   #       cacheSizeGB: <number>  #缓存大小
   #       journalCompressor: <string> #数据压缩格式 none/snappy/zlib
   #       directoryForIndexes: <boolean> #将索引和集合存储在单独的子目录下，默认false
   #    collectionConfig:
   #       blockCompressor: <string> #集合数据压缩格式 
   #    indexConfig:
   #       prefixCompression: <boolean> #启用索引的前缀压缩
   # inMemory:
   #    engineConfig:
   #       inMemorySizeGB: <number>
 
operationProfiling: #性能分析
   slowOpThresholdMs: 100 #认定为查询速度缓慢的时间阈值，超过该时间的查询即为缓慢查询，会被记录到日志中, 默认100
   mode: off #operationProfiling模式 off/slowOp/all 默认off

# replication: #复制集相关
#    oplogSizeMB: <int>
#    replSetName: <string>
#    secondaryIndexPrefetch: <string>
#    enableMajorityReadConcern: <boolean>
# sharding: #集群分片相关
#    clusterRole: <string>
#    archiveMovedChunks: <boolean>

# auditLog:
#    destination: <string>
#    format: <string>
#    path: <string>
#    filter: <string>

# snmp:
#    subagent: <boolean> #当设置为true，SNMP作为代理运行
#    master: <boolean> #当设置为true，SNMP作为主服务器运行

# basisTech:
#    rootDirectory: <string>

```

### 参考文章
* [MongoDB初体验-配置文件mongod.conf](https://www.jianshu.com/p/f179ce608391)
