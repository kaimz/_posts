---
title: CentOS 7 下安装mosquitto
date: 2017-09-11 22:24:03
tags: [linux,MQTT]
categories: 学习笔记
---
### 简介
>`MQTT`（Message Queuing Telemetry Transport，消息队列遥测传输）是IBM开发的一个`即时通讯协议`，有可能成为物联网的重要组成部分。该协议支持所有平台，几乎可以把所有联网物品和外部连接起来，被用来当做传感器和制动器（比如通过Twitter让房屋联网）的通信协议。

<!--more-->

### 特点
>`MQTT协议`是为大量计算能力有限，且工作在低带宽、不可靠的网络的远程传感器和控制设备通讯而设计的协议，它具有以下主要的几项特性：  
>1、使用发布/订阅消息模式，提供一对多的消息发布，解除应用程序耦合；  
>2、对负载内容屏蔽的消息传输；    
>3、使用 TCP/IP 提供网络连接；  
>4、有三种消息发布服务质量：  
>“至多一次”，消息发布完全依赖底层 TCP/IP   网络。会发生消息丢失或重复。这一级别可用于如下情况，环境传感器数据，丢失一次读记录无所谓，因为不久后还会有第二次发送。  
>“至少一次”，确保消息到达，但消息重复可能会发生。  
>“只有一次”，确保消息到达一次。这一级别可用于如下情况，在计费系统中，消息重复或丢失会导致不正确的结果。  
>5、小型传输，开销很小（固定长度的头部是 2 字节），协议交换最小化，以降低网络流量；  
>6、使用 Last Will 和 Testament 特性通知有关各方客户端异常中断的机制；

### CentOS 安装
#### 准备
在`/etc/yum.repos.d/`目录中新建一个`mosquitto.repo`文件

```bash
[home_oojah_mqtt]

name=mqtt (CentOS_CentOS-7)

type=rpm-md

baseurl=http://download.opensuse.org/repositories/home:/oojah:/mqtt/CentOS_CentOS-7/

gpgcheck=1

gpgkey=http://download.opensuse.org/repositories/home:/oojah:/mqtt/CentOS_CentOS-7//repodata/repomd.xml.key

enabled=1
```
执行 `yum search all mosquitto`
```bash
[root@localhost ~]# yum search all mosquitto
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * epel: mirrors.ustc.edu.cn
============================================================================================================= 匹配：mosquitto ==============================================================================================================
mosquitto-clients.x86_64 : Mosquitto command line publish/subscribe clients
mosquitto-debuginfo.x86_64 : Debug information for package mosquitto
mosquitto-devel.x86_64 : Development files for mosquitto
mosquitto.x86_64 : An Open Source MQTT v3.1/v3.1.1 Broker
libmosquitto-devel.x86_64 : MQTT C client library development files
libmosquitto1.x86_64 : MQTT C client library
libmosquittopp-devel.x86_64 : MQTT C++ client library development files
libmosquittopp1.x86_64 : MQTT C++ client library
```
#### 安装mosquitto客户端

执行 `yum install -y mosquitto-clients.x86_64`  

```bash
[root@localhost ~]# yum install -y mosquitto-clients.x86_64  
```
#### 安装mosquitto服务

执行命令 `yum install mosquitto.x86_64 `  

```bash
[root@localhost ~]# yum -y install mosquitto.x86_64 
```
#### 修改mosquitto.conf文件
文件在`/etc/mosquitto/mosquitto.conf`  
下面是可以选择的参数 在 `/etc/mosquitto/mosquitto.conf.example` 中 

```bash
# =================================================================
  # General configuration
  # =================================================================
  # 客户端心跳的间隔时间
  #retry_interval 20
  # 系统状态的刷新时间
  #sys_interval 10
  # 系统资源的回收时间，0表示尽快处理
  #store_clean_interval 10
  # 服务进程的PID
 #pid_file /var/run/mosquitto.pid
 # 服务进程的系统用户
 #user mosquitto
 # 客户端心跳消息的最大并发数
 #max_inflight_messages 10
 # 客户端心跳消息缓存队列
 #max_queued_messages 100
 # 用于设置客户端长连接的过期时间，默认永不过期
 #persistent_client_expiration
# =================================================================
# Default listener
# =================================================================
# 服务绑定的IP地址
#bind_address
# 服务绑定的端口号
#port 1883
# 允许的最大连接数，-1表示没有限制
#max_connections -1
# cafile：CA证书文件
# capath：CA证书目录
# certfile：PEM证书文件
# keyfile：PEM密钥文件
#cafile
#capath
#certfile
#keyfile
# 必须提供证书以保证数据安全性
#require_certificate false
# 若require_certificate值为true，use_identity_as_username也必须为true
#use_identity_as_username false
# 启用PSK（Pre-shared-key）支持
#psk_hint
# SSL/TSL加密算法，可以使用“openssl ciphers”命令获取
# as the output of that command.
#ciphers
# =================================================================
# Persistence
# =================================================================
# 消息自动保存的间隔时间
#autosave_interval 1800
# 消息自动保存功能的开关
#autosave_on_changes false
# 持久化功能的开关
persistence true
# 持久化DB文件
#persistence_file mosquitto.db
# 持久化DB文件目录
#persistence_location /var/lib/mosquitto/
# =================================================================
# Logging
# =================================================================
# 4种日志模式：stdout、stderr、syslog、topic
# none 则表示不记日志，此配置可以提升些许性能
log_dest none
# 选择日志的级别（可设置多项）
#log_type error
#log_type warning
#log_type notice
#log_type information
# 是否记录客户端连接信息
#connection_messages true
# 是否记录日志时间
#log_timestamp true
# =================================================================
# Security
# =================================================================
# 客户端ID的前缀限制，可用于保证安全性
#clientid_prefixes
# 允许匿名用户
#allow_anonymous true
# 用户/密码文件，默认格式：username:password
#password_file
# PSK格式密码文件，默认格式：identity:key
#psk_file
# pattern write sensor/%u/data
# ACL权限配置，常用语法如下：
# 用户限制：user <username>
# 话题限制：topic [read|write] <topic>
# 正则限制：pattern write sensor/%u/data
#acl_file
# =================================================================
# Bridges
# =================================================================
# 允许服务之间使用“桥接”模式（可用于分布式部署）
#connection <name>
#address <host>[:<port>]
#topic <topic> [[[out | in | both] qos-level] local-prefix remote-prefix]
# 设置桥接的客户端ID
#clientid
  # 桥接断开时，是否清除远程服务器中的消息
  #cleansession false
  # 是否发布桥接的状态信息
  #notifications true
  # 设置桥接模式下，消息将会发布到的话题地址
  # $SYS/broker/connection/<clientid>/state
  #notification_topic
  # 设置桥接的keepalive数值
  #keepalive_interval 60
  # 桥接模式，目前有三种：automatic、lazy、once
  #start_type automatic
  # 桥接模式automatic的超时时间
  #restart_timeout 30
  # 桥接模式lazy的超时时间
  #idle_timeout 60
  # 桥接客户端的用户名
  #username
  # 桥接客户端的密码
  #password
  # bridge_cafile：桥接客户端的CA证书文件
  # bridge_capath：桥接客户端的CA证书目录
  # bridge_certfile：桥接客户端的PEM证书文件
  # bridge_keyfile：桥接客户端的PEM密钥文件
  #bridge_cafile
  #bridge_capath
  #bridge_certfile
  #bridge_keyfile
  # 自己的配置可以放到以下目录中
  include_dir /etc/mosquitto/conf.d
```
#### CentOS 中第二种方式安装

```bash
$ yum install epel-release
$ yum install mosquitto
$ service mosquitto start
```



### Ubuntu 安装

#### 安装 mqtt

```bash
$ sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa
$ sudo apt-get update
$ sudo apt-get install mosquitto
$ sudo apt-get install mosquitto-clients
```

#### 配置文件

在 `/etc/mosquitto/` 下，跟 上面 CentOS 中一样配置就行，它没生成 `pwfile`和  `aclfile`，需要自己配置。



### 启动服务

```bash
mosquitto -c /etc/mosquitto/mosquitto.conf -d
```
```bash
[root@localhost mosquitto]# service mosquitto start
Restarting mosquitto (via systemctl):                      [  确定  ]
```
重启
### 测试
一个完整的MQTT示例包括一个代理服务器器，一个发布者和一个订阅者。
我使用的是一台服务机，只是开了三个控制台窗口。
* 订阅者通过mosquitto_sub订阅指定主题的消息。
* 发布者通过mosquitto_pub发布指定主题的消息。
* 代理服务器把该主题的消息推送到订阅者。 

1. 第一个控制台启动代码服务器；mosquitto -v
```bash
[root@localhost yum.repos.d]# mosquitto -v
1516350767: mosquitto version 1.4.14 (build date 2017-09-14 18:40:30+0000) starting
1516350767: Using default config.
1516350767: Opening ipv4 listen socket on port 1883.
1516350767: Opening ipv6 listen socket on port 1883.
```
2. 第二个窗口订阅主题：
```bash
[root@localhost ~]# mosquitto_sub  -v -t wuwii
```
并且可以在第一个窗口发现订阅连接上，并且发送心跳消息：
```bash
1516351070: New connection from ::1 on port 1883.
1516351070: New client connected from ::1 as mosqsub|1964-localhost (c1, k60).
1516351070: Sending CONNACK to mosqsub|1964-localhost (0, 0)
1516351070: Received SUBSCRIBE from mosqsub|1964-localhost
1516351070:     wuwii (QoS 0)
1516351070: mosqsub|1964-localhost 0 wuwii
1516351070: Sending SUBACK to mosqsub|1964-localhost
1516351130: Received PINGREQ from mosqsub|1964-localhost
1516351130: Sending PINGRESP to mosqsub|1964-localhost
```
3. 第三个窗口发布主题：
```bash
[root@localhost ~]# mosquitto_pub -t wuwii -m 我发送消息了
```
在第二个窗口（订阅者）查看到：
![](https://ws1.sinaimg.cn/large/ece1c17dgy1fnm0ed7suuj20bd01at8i.jpg)

在第一个窗口（服务器）查看到：
![](https://ws1.sinaimg.cn/large/ece1c17dgy1fnm0j558vjj20mg09owfm.jpg)

可以看到连接、消息发布和心跳等调试信息，好了测试完成。

主要还是根据需求来配置好用户和加密，用的时候需要了解mqtt协议。 

