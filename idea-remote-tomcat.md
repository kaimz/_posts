---
title: 使用IDEA远程部署tomcat和调试
date: 2017-12-23 21:08:03
tags: [idea]
categories: 学习笔记
---

环境：
* CentOS 7 
* Tomcat 9.0.1
* jdk-9.0.1
* IntelliJ IDEA 2017.3

<!--more-->

### Tomcat中的配置
1. 在``catalina.sh``文件中加入以下的配置
```bash
CATALINA_OPTS="-Dcom.sun.management.jmxremote 
-Dcom.sun.management.jmxremote.port=1099 
-Dcom.sun.management.jmxremote.ssl=false 
-Dcom.sun.management.jmxremote.authenticate=false 
-Djava.rmi.server.hostname=192.168.19.200
-agentlib:jdwp=transport=dt_socket,address=15833,suspend=n,server=y"
export CATALINA_OPTS
```
* 以上端口可以随意改动，但是必要的是后续的设置必须保持一致，并且务必保证端口没有被占用，这些设置的端口在防火墙中是开放状态；
* 其中1099的是tomcat远程部署连接端口；
* 15833 是远程调试的端口；
* 192.168.19.200是远程的服务器的Ip。

2. 启动tomcat
使用命令启动
```bash
./bin/catalina.sh run &
```


### IDEA中的配置
#### 新建远程tomcat
![img](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwy19cp08j20d10fwt9i.jpg)

#### 配置远程服务
![img](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwy1n90ssj20kf0jodgc.jpg)

![img](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwy1utpsnj20be069q2y.jpg)

![img](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwy21y3y9j20mo0ix0th.jpg)

#### 配置连接tomcat的一些属性

![img](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwy27lmh3j20uc0nlmyl.jpg)

![img](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwy2en68yj20uc0nlgmg.jpg)

![img](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwy2kn4vzj20uc0nlab2.jpg)

#### debug启动测试
连接：
```
[2017-12-23 08:47:03,592] Artifact devframe-server:war exploded: Artifact is not deployed. Press 'Deploy' to start deployment
[2017-12-23 08:47:03,650] Artifact devframe-server:war exploded: Artifact is being deployed, please wait...
Connected to server
Connected to the target VM, address: '192.168.19.200:15833', transport: 'socket'
[2017-12-23 08:47:11,434] Artifact devframe-server:war exploded: Error during artifact deployment. See server log for details.
```

文件传输：
```
[2017/12/23 20:47] Uploading to 192.168.19.200 completed in less than a minute: 357 files transferred (8 Mbit/s)
```

这样就能够成功远程部署并且调试了。

使用的技巧：
![img](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwy2rv1g0j20dc071dgc.jpg)
这样每次修改完，点一下就可以热部署，是不是很方便。

### 容易出现的问题
* 如果远程没有连接上，两个端口被占用或者防火墙屏蔽。除了JMX server指定的监听端口号外，JMXserver还会监听一到两个随机端口号，这个如果防火墙关闭了的话就不用考虑，如果使用了防火墙，还需要查看它监听的端口。
* 账号的相应的读写权限一定要有；

