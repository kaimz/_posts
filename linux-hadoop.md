---
title: CentOS 7 上安装Hadoop V 2.8.1集群及配置
date: 2017-10-19 16:08:03
tags: [java]
categories: 学习笔记
---

>Hadoop是一个由Apache基金会所开发的分布式系统基础架构。
用户可以在不了解分布式底层细节的情况下，开发分布式程序。充分利用集群的威力进行高速运算和存储。  
 Hadoop实现了一个分布式文件系统（Hadoop Distributed File System），简称HDFS。HDFS有高容错性的特点，并且设计用来部署在低廉的（low-cost）硬件上；而且它提供高吞吐量（high throughput）来访问应用程序的数据，适合那些有着超大数据集（large data set）的应用程序。HDFS放宽了（relax）POSIX的要求，可以以流的形式访问（streaming access）文件系统中的数据。  
Hadoop的框架最核心的设计就是：HDFS和MapReduce。HDFS为海量的数据提供了存储，则MapReduce为海量的数据提供了计算。(百科)

![image](https://gss1.bdstatic.com/9vo3dSag_xI4khGkpoWK1HF6hhy/baike/w%3D268%3Bg%3D0/sign=98010877b33533faf5b6942890e89a22/3c6d55fbb2fb4316ecfbfb0322a4462308f7d3e7.jpg)

<!--more-->
## 下载Hadoop
本次使用的是2.8.1版本的Hadoop，官网地址
<a rel="external nofollow" target="_blank" href="http://www.apache.org/dyn/closer.cgi/hadoop/common/hadoop-2.8.1/hadoop-2.8.1.tar.gz">http://www.apache.org/dyn/closer.cgi/hadoop/common/hadoop-2.8.1/hadoop-2.8.1.tar.gz</a>

点击（不用进官网直接点这个链接就能下载）

<a rel="external nofollow" target="_blank" href=" http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-2.8.1/hadoop-2.8.1.tar.gz "> http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-2.8.1/hadoop-2.8.1.tar.gz </a>

## 安装3个虚拟机并实现ssh免密码登录
### 修改host
使用的Linux系统是CentOS 7 ，修改三台机器的Hosts，让它们能相互映射到，能ping t通 
参考我的上一篇文章 

http://blog.wuwii.com/linux-hostname.html

添加Hosts，这是我的三台机器
```bash
192.168.19.185  server1
192.168.19.184  server2
192.168.19.199  server3
```


ping 结果都能ping 通
```bash
[root@server2 ~]# ping -c 4 server1
PING server1 (192.168.19.185) 56(84) bytes of data.
64 bytes from server1 (192.168.19.185): icmp_seq=1 ttl=64 time=0.536 ms
64 bytes from server1 (192.168.19.185): icmp_seq=2 ttl=64 time=0.388 ms
64 bytes from server1 (192.168.19.185): icmp_seq=3 ttl=64 time=0.309 ms
64 bytes from server1 (192.168.19.185): icmp_seq=4 ttl=64 time=0.368 ms

--- server1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3001ms
rtt min/avg/max/mdev = 0.309/0.400/0.536/0.084 ms
```

### 生成密钥
密钥三台机器都需要生成，就以一台 server1 机器为例

使用命令 ``ssh-keygen -t rsa`` 一路 enter 
```bash
[root@server1 ~]# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
e0:ea:e3:5d:95:be:c5:9a:dc:90:99:22:d1:cf:99:49 root@server1
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|      .          |
|     . o   .     |
|      o S E      |
|     . . * O     |
|    . . o % o    |
|   ... o o B     |
|   .o..   = .    |
+-----------------+
```
出现上面的 ，可以在本帐户的根目录看到一个 .ssh 文件夹 

```bash
[root@server1 ~]#  ll -a
总用量 68
dr-xr-x---.  6 root root   256 10月 18 15:00 .
dr-xr-xr-x. 20 root root  4096 10月 18 10:35 ..
-rw-------.  1 root root  1456 8月  14 08:44 anaconda-ks.cfg
-rw-------.  1 root root 24538 10月 18 10:35 .bash_history
-rw-r--r--.  1 root root    18 12月 29 2013 .bash_logout
-rw-r--r--.  1 root root   176 12月 29 2013 .bash_profile
-rw-r--r--.  1 root root   176 12月 29 2013 .bashrc
-rw-r--r--.  1 root root   100 12月 29 2013 .cshrc
-rw-r--r--   1 root root   223 9月  27 10:47 dump.rdb
drwxr-xr-x. 11 root root   270 8月  15 15:57 fastdfs
drwxr-xr-x.  2 root root    40 8月  15 15:04 .oracle_jre_usage
drwxr-----.  3 root root    19 8月  15 15:53 .pki
-rw-------   1 root root   571 9月  27 16:58 .rediscli_history
drwx------   2 root root    38 10月 18 14:56 .ssh
-rw-r--r--.  1 root root   129 12月 29 2013 .tcshrc
-rw-------   1 root root  7372 10月 18 11:35 .viminfo
```
注意它是个隐藏的文件，我是用的是secureFx 显示隐藏文件，需要 视图 -> 文件 勾选上就行
```bash
[root@server1 .ssh]# ll
总用量 8
-rw------- 1 root root 668 10月 18 15:12 id_rsa
-rw-r--r-- 1 root root 602 10月 18 15:12 id_rsa.pub
```

打开 ``/root/.ssh/id_rsa.pub`` 
```
[root@server1 .ssh]# cat id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCpRge0XRJya0rXjaMs7VQ5uHrmaVxzFekB/gNoFNUsJ7cjWfFUpUao8zZpioCUceUWdI4sL0doQGriTXBjwrhDtcaO0IZujG2oyD1OGfOVbn7Yuhc6EZz0fed5soj6AZrGIgTMrweRpD268bvcJCcWOPV7U2iAjOqYSmP2Z/1ckYwJ983qSLvHPhPVnFBENmo9Evgzfa/6QM+j2UbVIIjfiUPxo4BNWxcvVruxJV+pEFa1ycAT8ORvLxirgafctdfw+Md1Epuna0RIE59H3382COUjC/UonAya5ebl1z5JGY65dREIdRDcvYfwnMcpeF5mkEuowyX/1Ev3y+JFENBV root@server1
```

查看到了我们生成的密钥成功了

然后我们把三个机器都生成密钥，然后把他们合并成一个文件创建一个``/root/.ssh/authorized_keys`` 文件保存着。

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwwfjadmhj20o604674a.jpg)

使用命令
```bash
[root@server1 ~]# cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
```
其他机器的公共密钥也复制到到这个文件里来（补充，不要连着复制，上一行后面打个空格，再换行。）

所以最后是这样的
```

[root@server1 .ssh]# vim authorized_keys

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCpRge0XRJya0rXjaMs7VQ5uHrmaVxzFekB/gNoFNUsJ7cjWfFUpUao8zZpioCUceUWdI4sL0doQGriTXBjwrhDtcaO0IZujG2oyD1OGfOVbn7Yuhc6EZz0fed5soj6AZrGIgTMrweRpD268bvcJCcWOPV7U2iAjOqYSmP2Z/1ckYwJ983qSLvHPhPVnFBENmo9Evgzfa/6QM+j2UbVIIjfiUPxo4BNWxcvVruxJV+pEFa1ycAT8ORvLxirgafctdfw+Md1Epuna0RIE59H3382COUjC/UonAya5ebl1z5JGY65dREIdRDcvYfwnMcpeF5mkEuowyX/1Ev3y+JFENBV root@server1

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDFwe2pGEMWt+X0FXFPotOQrUAJFDOLflMjtwBIJxTSFBPQuVhoEtJHkacnpsPAtT4zOJxjieLOrsC/G5fKZVpSgYRwmMw6iobe3IsL5uElVfRYoO+HIr/BDep1imVFkmj0DTMUj0q+UYz3wiEaFQk4zh7Gas2qIdgyOtfSQcYN3T7qNh4dPDfdOrBIqZq/fP33UFDBgbUqGZUZhL6mHc8LRHo9+eby3ZPtiEudfeczvi3pI0Dcp0zX+WSuqPK/z47hBN2XlGMIDO2Ta5sAu9WfECe0WcxsPLOPsKPCRsakyMrYlnGk3hEQ9Ci1YsNKUX8j1RhBi3YLKsl5rjhQR67r root@server2

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDFPaRkR/0i51MORrPVnsEZR60t7FZDmJ3DlhVKdt4crCHO+QhsHr5ZwbcLT/9vTBAdRoveuwHJreEO5MLnlcG0dxFjVDduip5M84zGjmKI1k7/tyeNT1bHUhoMWRAaDEk9RUx/rrYzR/DzHvkdXPwPK+uENFCFBo0RTEGxAMkrXkex7SFNITh8t48sto23D20v7O4A+h4Fbe4oiEjlFBeK6H+dJxZVqYE5Xof1Y4Nc0Xh0YfEg9rUT4BS1AdYWZB9ptVyuSzsbmBd1mve8GcR8cf0M75uSIovc3ww/z/sVpx+hluldhVN9wXyUtFZdWcbklJcq6oTMfejY7ISv2lKh root@server3
~                                                                               
~                                                                               
~                                                                               
~                                                                               
~                                                                               
~                                                                               
~                                                                               
"authorized_keys" [新] 5L, 1183C 已写入
```
每个电脑都需要这个``/root/.ssh/authorized_keys``文件，所以直接把它复制到对应位置就行了。

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwwgjfvy1j20o604674a.jpg)

**注意**
我们需要给它们相应的权限，一般默认的就是这个权限，下面是root 用户的。  
``chmod 700 ~/.ssh`` #注意：这两条权限设置特别重要，决定成败。
``chmod 600 ~/.ssh/authorized_keys``

测试使用ssh 密钥无密码登陆

首先测试下localhost ，看能否无密码登陆自己
```
[root@server1 ~]# ssh localhost
Last login: Thu Oct 19 09:01:34 2017 from 192.168.19.207
[root@server1 ~]# 
```


演示下server2 电脑上进行登陆 server1 并进行操作，
```bash
[root@server2 ~]# ssh server1
The authenticity of host 'server1 (192.168.19.185)' can't be established.
ECDSA key fingerprint is bd:50:b8:e7:b3:69:ad:6c:14:6b:a9:fb:18:43:b9:c9.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'server1,192.168.19.185' (ECDSA) to the list of known hosts.
Last login: Wed Oct 18 16:46:53 2017 from server1
[root@server1 ~]# exit
logout
Connection to server1 closed.
```
之前配置 ``authorized_keys`` 搞了半天，虽然用了三行，但是后面没空格，导致 密钥不能使用，每次登陆还需要密码。
```bash
[root@server1 ~]# ssh server2
root@server2's password: 
```
没百度出来，自己最后卡了一个小时 恍然大悟，每行后面空出空格就好了。

在其余的电脑相互登陆试试，都能无密登陆，说明，配置成功。

**但是一定要注意的是，每次ssh登陆完成后，都要执行 ``exit``，否则你的后续命令是在另外一台机器上执行的。**

## 安装JDK 和Hadoop

### 安装jdk
三台机器都需要安装jdk，CentOS7 安装JDK参考 http://blog.wuwii.com/linux-jdk.html

### 安装hadoop
首先 三台机器都需要安装hadoop，都需要执行下面所有的操作。

#### 上载文件，并且解压
```bash
[root@server1 opt]# tar -xvf hadoop-2.8.1.tar.gz 
```
解压缩后得到``hadoop-2.8.1`` 文件夹。
#### 新建目录
在/root 目录下新建
```bash
mkdir /root/hadoop
mkdir /root/hadoop/tmp
mkdir /root/hadoop/var
mkdir /root/hadoop/dfs
mkdir /root/hadoop/dfs/name
mkdir /root/hadoop/dfs/data
```
#### 修改配置文件
配置文件都在 解压后的文件夹 ``hadoop-2.8.1/etc/hadoop`` 下。
##### 修改core-site.xml
 在configuration>节点内加入配置:
```xml
<property>
    <name>hadoop.tmp.dir</name>
    <value>/root/hadoop/tmp</value>
    <description>Abase for other temporary directories.</description>
</property>
<property>
    <name>fs.default.name</name>
    <value>hdfs://server1:9000</value>
</property>
```
##### 修改 hadoop-env.sh文件
修改``./hadoop-2.8.1/etc/hadoop/hadoop-env.sh``文件  
将``export JAVA_HOME=${JAVA_HOME}``  
**修改为：**  
``export JAVA_HOME=/usr/java/jdk1.8.0_144``  
 **说明：修改为自己的JDK路径和版本号**

##### 修改hdfs-site.xml
修改``./hadoop-2.8.1/etc/hadoop/hdfs-site.xml``文件，
在<configuration>节点内加入配置:

```xml
<property>
   <name>dfs.name.dir</name>
   <value>/root/hadoop/dfs/name</value>
   <description>Path on the local filesystem where theNameNode stores the namespace and transactions logs persistently.</description>
</property>
<property>
   <name>dfs.data.dir</name>
   <value>/root/hadoop/dfs/data</value>
   <description>Comma separated list of paths on the localfilesystem of a DataNode where it should store its blocks.</description>
</property>
<property>
   <name>dfs.replication</name>
   <value>2</value>
</property>

<property>
      <name>dfs.permissions</name>
      <value>false</value>
      <description>need not permissions</description>
</property>
```
**说明：dfs.permissions配置为false后，可以允许不要检查权限就生成dfs上的文件，方便倒是方便了，但是你需要防止误删除，请将它设置为true，或者直接将该property节点删除，因为默认就是true。** 
##### 新建并且修改mapred-site.xml  
在该版本中，有一个名为mapred-site.xml.template的文件，复制该文件，然后改名为mapred-site.xml，命令是：
```bash
[root@server1 hadoop]# cp mapred-site.xml.template mapred-site.xml
```
修改这个新建的mapred-site.xml文件，在<configuration>节点内加入配置:
```xml
 <property>
    <name>mapred.job.tracker</name>
    <value>server1:49001</value>
</property>
<property>
      <name>mapred.local.dir</name>
       <value>/root/hadoop/var</value>
</property>

<property>
       <name>mapreduce.framework.name</name>
       <value>yarn</value>
</property>
```
##### 修改slaves文件  
 修改``./hadoop-2.8.1/etc/hadoop/slaves``文件，将里面的localhost删除，添加如下内容：

```bash
server2
server3
```
##### 修改yarn-site.xml文件  
修改``./hadoop-2.8.1/etc/hadoop/yarn-site.xml`` 文件，
在<configuration>节点内加入配置(注意了，内存根据机器配置越大越好):

```xml
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>server1</value>
   </property>
   <property>
        <description>The address of the applications manager interface in the RM.</description>
        <name>yarn.resourcemanager.address</name>
        <value>${yarn.resourcemanager.hostname}:8032</value>
   </property>
   <property>
        <description>The address of the scheduler interface.</description>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>${yarn.resourcemanager.hostname}:8030</value>
   </property>
   <property>
        <description>The http address of the RM web application.</description>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>${yarn.resourcemanager.hostname}:8088</value>
   </property>
   <property>
        <description>The https adddress of the RM web application.</description>
        <name>yarn.resourcemanager.webapp.https.address</name>
        <value>${yarn.resourcemanager.hostname}:8090</value>
   </property>
   <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>${yarn.resourcemanager.hostname}:8031</value>
   </property>
   <property>
        <description>The address of the RM admin interface.</description>
        <name>yarn.resourcemanager.admin.address</name>
        <value>${yarn.resourcemanager.hostname}:8033</value>
   </property>
   <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
   </property>
   <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>1024</value>
        <discription>每个节点可用内存,单位MB,默认8182MB</discription>
   </property>
   <property>
        <name>yarn.nodemanager.vmem-pmem-ratio</name>
        <value>2.1</value>
   </property>
   <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>1024</value>
</property>

```
**说明：``yarn.nodemanager.vmem-check-enabled`` 这个的意思是忽略虚拟内存的检查，如果你是安装在虚拟机上，这个配置很有用，配上去之后后续操作不容易出问题。如果是实体机上，并且内存够多，可以将这个配置去掉。**

## 启动Hadoop
### 在namenode上执行初始化
 因为server1是namenode，server2和server3都是datanode，所以只需要对server1进行初始化操作，也就是对hdfs进行格式化。  
进入到server1这台机器的/opt/hadoop-2.8.1/bin目录，执行初始化命令：``./hadoop namenode -format``  ，格式化一个新的分布式文件系统。
如下
```bash
[root@server1 bin]# cd /opt/hadoop-2.8.1/bin/                      
[root@server1 bin]# ./hadoop namenode -format                      
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.

17/10/19 15:09:05 INFO namenode.NameNode: STARTUP_MSG: 
/************************************************************
STARTUP_MSG: Starting NameNode
STARTUP_MSG:   user = root
STARTUP_MSG:   host = server1/192.168.19.185
STARTUP_MSG:   args = [-format]
STARTUP_MSG:   version = 2.8.1
STARTUP_MSG:   classpath = /opt/hadoop-2.8.1/etc/hadoop:/opt/hadoop-2.8.1/share/
```
执行完成，不报错，说明启动成功。  
![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwwgsmqnrj20io0oydj2.jpg)

格式化成功后，可以在看到在``/root/hadoop/dfs/name/``目录多了一个current目录，而且该目录内有一系列文件。
![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwwh9d2gqj20p907kjrk.jpg)

### 在namenode上执行启动命令
因为server1是namenode，server2和server3都是datanode，所以只需要再server1上执行启动命令即可。  
进入到hserver1这台机器的``/opt/hadoop-2.8.1/sbin``目录，也就是执行命令：  
``cd /opt/hadoop/hadoop-2.8.0/sbin``  
执行初始化脚本，也就是执行命令：
``./start-all.sh``  
第一次执行上面的启动命令，会需要我们进行交互操作，在问答界面上输入yes回车

```bash
[root@server1 hadoop-2.8.1]# sbin/start-all.sh 
This script is Deprecated. Instead use start-dfs.sh and start-yarn.sh
Starting namenodes on [server1]
server1: namenode running as process 3609. Stop it first.
server3: starting datanode, logging to /opt/hadoop-2.8.1/logs/hadoop-root-datanode-server3.out
server2: datanode running as process 17888. Stop it first.
server3: [Fatal Error] yarn-site.xml:16:1: Content is not allowed in prolog.
Starting secondary namenodes [0.0.0.0]
0.0.0.0: secondarynamenode running as process 3795. Stop it first.
starting yarn daemons
resourcemanager running as process 3942. Stop it first.
server3: starting nodemanager, logging to /opt/hadoop-2.8.1/logs/yarn-root-nodemanager-server3.out
server2: nodemanager running as process 18038. Stop it first.
server3: [Fatal Error] yarn-site.xml:16:1: Content is not allowed in prolog.
[root@server1 hadoop-2.8.1]# sbin/start-all.sh 
This script is Deprecated. Instead use start-dfs.sh and start-yarn.sh
Starting namenodes on [server1]
server1: namenode running as process 3609. Stop it first.
server3: starting datanode, logging to /opt/hadoop-2.8.1/logs/hadoop-root-datanode-server3.out
server2: datanode running as process 17888. Stop it first.
Starting secondary namenodes [0.0.0.0]
0.0.0.0: secondarynamenode running as process 3795. Stop it first.
starting yarn daemons
resourcemanager running as process 3942. Stop it first.
server2: nodemanager running as process 18038. Stop it first.
server3: starting no
```
没报错，说明执行成功，之前我的server3 上的一个xml 配置错了，很明了的说出了错误的位置。  

## 测试hadoop

启动后，需要测试能使用，才能说明配置正确

首先需要关闭防火墙。

```
[root@server1 ~]# systemctl stop firewalld.service
```

我们的namanode机器是server1，IP是192.168.19.185，直接在谷歌浏览器上输入到端口 50070，自动跳转到了overview页面 （dfshealth.html）   
![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwwhjt03pj213h0g575d.jpg)

继续；  
测试 8088 端口 ：  
自动跳转到了cluster页面
![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwwhv1qa8j21go0fq0vk.jpg)

在namenode机器上执行``jps``
```bash
[root@server1 hadoop-2.8.1]# jps
12469 ResourceManager
12119 NameNode
12313 SecondaryNameNode
12730 Jps
```

在datanode机器上执行``jps``
```bash
[root@server3 hadoop-2.8.1]# jps
10776 NodeManager
11114 Jps
10635 DataNode
```
这只能证明它们启动成功，还要看它们之间互相通信。
![hadoop](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwwi35at0j21bn0g3q3z.jpg)
出现datanode 机器，通信成功。

配置完成。

**参考博客：** <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/pucao_cug/article/details/71698903">http://blog.csdn.net/pucao_cug/article/details/71698903</a>
