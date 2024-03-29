---
title: redies安装
date: 2017-09-26 22:08:03
tags: [linux,redis]
categories: 学习笔记
---


###  使用文件安装
#### 下载
直接去官网下载最新版的，前面使用`wget`下载半天下载下来的全是错误文件，浪费了大把时间，最重要的是浪费心情。<!--more-->

Redis官网(redis.io)

#### 解压
```bash
[root@localhost bin]# tar xzf redis-4.0.2.tar.gz 
```

#### 编译
进入解压的文件中

```bash
cd redis-4.0.2
```

编译
```bash
make
cd src
make install PREFIX=/usr/local/redis
```
#### 文件管理
新建文件夹
```bash
mkdir -p /usr/local/redis/etc
```
移动config文件

```bash
mv ./redis.conf /usr/local/redis/etc/
```
### 启动Redis服务

```bash
/usr/local/redis/bin/redis-server
```

前台运行中

想要后台运行修改配置文件
```bash
 vim /usr/local/redis/etc/redis.conf
```
将`daemonize`的值改为`yes`

后台运行命令
```bash
　/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf
```
查看端口
```bash
[root@localhost bin]# netstat -tunpl | grep 6379                    
tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN      5769/./redis-server 
```
启动成功。  

客户端登陆
```bash
[root@localhost bin]# /usr/local/redis/bin/redis-cli 
127.0.0.1:6379> set foo bar
OK
127.0.0.1:6379> get foo
"bar"
```
关闭服务
```bash
/usr/local/redis/bin/redis-cli shutdown
```
redis开机自启  

``vim /etc/rc.local``  
底部添加  
　``　/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf``

### 了解

```bash
redis-benchmark：redis性能测试工具

　　redis-check-aof：检查aof日志的工具

　　redis-check-dump：检查rdb日志的工具

　　redis-cli：连接用的客户端

　　redis-server：redis服务进程
```

### 剩下就是配置


```bash
    daemonize：如需要在后台运行，把该项的值改为yes

　　pdifile：把pid文件放在/var/run/redis.pid，可以配置到其他地址

　　bind：指定redis只接收来自该IP的请求，如果不设置，那么将处理所有请求，在生产环节中最好设置该项

　　port：监听端口，默认为6379

　　timeout：设置客户端连接时的超时时间，单位为秒

　　loglevel：等级分为4级，debug，revbose，notice和warning。生产环境下一般开启notice

　　logfile：配置log文件地址，默认使用标准输出，即打印在命令行终端的端口上

　　database：设置数据库的个数，默认使用的数据库是0

　　save：设置redis进行数据库镜像的频率

　　rdbcompression：在进行镜像备份时，是否进行压缩

　　dbfilename：镜像备份文件的文件名

　　dir：数据库镜像备份的文件放置的路径

　　slaveof：设置该数据库为其他数据库的从数据库

　　masterauth：当主数据库连接需要密码验证时，在这里设定

　　requirepass：设置客户端连接后进行任何其他指定前需要使用的密码

　　maxclients：限制同时连接的客户端数量

　　maxmemory：设置redis能够使用的最大内存

　　appendonly：开启appendonly模式后，redis会把每一次所接收到的写操作都追加到appendonly.aof文件中，当redis重新启动时，会从该文件恢复出之前的状态

　　appendfsync：设置appendonly.aof文件进行同步的频率

　　vm_enabled：是否开启虚拟内存支持

　　vm_swap_file：设置虚拟内存的交换文件的路径

　　vm_max_momery：设置开启虚拟内存后，redis将使用的最大物理内存的大小，默认为0

　　vm_page_size：设置虚拟内存页的大小

　　vm_pages：设置交换文件的总的page数量

　　vm_max_thrrads：设置vm IO同时使用的线程数量
```



### （推荐）使用 yum 安装

1. redis在第三方的源里，首先添加源，添加 epel 源

```bash
yum install epel-release -y
```

2. 安装 redis 

```bash
yum install redis -y
```

3. 配置文件在 `/etc/redis.conf `，配置文件和上面一样的。
4. 启动命令 `systemctl start redis`，或者使用指定配置文件的方式 `redis-server /etc/redis.conf`。
5. 开机自启 `systemctl enable redis`。
6. 使用 redis 客户端  `redis-cli`
7. 将 6379 端口打开 ``firewall-cmd --add-port=6379/tcp` `

