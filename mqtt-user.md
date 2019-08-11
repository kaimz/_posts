---
title: Mosquitto用户名密码配置
date: 2017-09-12 10:08:03
tags: [mq]
categories: 学习笔记
---


### 介绍
由于需要把`mqtt`部署到外网上面去，所以需要关闭`匿名登陆`，采取用户`认证模式`，而且还可能需要把主题加密。

<!--more-->
### 配置
#### 参数说明 
配置参数在`/etc/mosquitto/mosquitto.conf`中，

```bash
allow_anonymous 允许匿名登陆
password_file 账号密码文件
acl_file 访问控制列表
```

配置
```bash
[root@localhost mosquitto]# vim mosquitto.conf           
# Place your local configuration in /etc/mosquitto/conf.d/

pid_file /var/run/mosquitto.pid

persistence true
persistence_location /var/lib/mosquitto/

#log_dest file /var/log/mosquitto/mosquitto.log

include_dir /etc/mosquitto/conf.d
allow_anonymous false
password_file /etc/mosquitto/pwfile
acl_file /etc/mosquitto/aclfile
port 8001
```

#### 查看用户
```bash
[root@localhost mosquitto]# cat pwfile                   
agri:$6$NprvJLB/CkEomWGy$gNj5Mr6Wf+2Xz16P6dIZYD/ladZZtyKQMJ/tdpy7WLepj5akpPB+hF8zolrNd5IacbsAxXDWX1vS5I9Pj4fnCA==
```
添加用户
```bash
# mosquitto_passwd --help 了解到使用 -c 会覆盖密码文件中的内容，也就是添加一个用户后会覆盖以前的账户信息
mosquitto_passwd -c /etc/mosquitto/pwfile username
# mosquitto_passwd -d 新增一个用户密码
mosquitto_passwd -d /etc/mosquitto/pwfile username password
```
设置好密码

#### 添加Topic和用户的关系
```bash
[root@localhost ~]# vim /etc/mosquitto/aclfile
# This affects access control for clients with no username.
#topic read $SYS/#

# This only affects clients with username "roger".
#user roger
#topic foo/bar
#user zhang
user zhang
topic read mtopic/#

user zhang
topic write mtopic/#

# This affects all clients.
# pattern write $SYS/broker/connection/%c/state
```

#### 重启测试
```bash
[root@localhost mosquitto]# /etc/init.d/mosquitto restart
Restarting mosquitto (via systemctl):                      [  确定  ]
```
