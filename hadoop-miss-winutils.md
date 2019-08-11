---
title: Windows下调试Hadoop集群报错Failed to find winutils.exe
date: 2017-10-27 11:48:03
tags: [java]
categories: 学习笔记
---

### 问题
在windows 环境使用``Java``下调试远程虚拟机中的Hadoop集群报错，问题很奇怪，说是少了 ``winutils.exe`` 文件，而且少了``HADOOP_HOME`` 的环境变量；我是部署在虚拟机CentOS 7 上的集群，难道Windows 上使用 它的Hadoop还需要自己安装环境，事实上，是真的。。

<!--more-->

```console
10:17:34,377 DEBUG Shell:675 - Failed to find winutils.exe
java.io.FileNotFoundException: java.io.FileNotFoundException: HADOOP_HOME and hadoop.home.dir are unset. -see https://wiki.apache.org/hadoop/WindowsProblems
	at org.apache.hadoop.util.Shell.fileNotFoundException(Shell.java:528)
	at org.apache.hadoop.util.Shell.getHadoopHomeDir(Shell.java:549)
	at org.apache.hadoop.util.Shell.getQualifiedBin(Shell.java:572)
	at org.apache.hadoop.util.Shell.<clinit>(Shell.java:669)
	at org.apache.hadoop.util.StringUtils.<clinit>(StringUtils.java:79)
	at org.apache.hadoop.fs.FileSystem$Cache$Key.<init>(FileSystem.java:2972)
	at org.apache.hadoop.fs.FileSystem$Cache$Key.<init>(FileSystem.java:2967)
	at org.apache.hadoop.fs.FileSystem$Cache.get(FileSystem.java:2829)
	at org.apache.hadoop.fs.FileSystem.get(FileSystem.java:389)
	at com.devframe.util.HdfsUtils.mkdir(HdfsUtils.java:43)
	at com.devframe.util.HdfsUtilsTest.testMkdir(HdfsUtilsTest.java:32)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:50)
	at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
	at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:47)
	at org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:17)
	at org.junit.internal.runners.statements.RunBefores.evaluate(RunBefores.java:26)
	at org.junit.internal.runners.statements.RunAfters.evaluate(RunAfters.java:27)
	at org.junit.runners.ParentRunner.runLeaf(ParentRunner.java:325)
	at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:78)
	at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:57)
	at org.junit.runners.ParentRunner$3.run(ParentRunner.java:290)
	at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:71)
	at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:288)
	at org.junit.runners.ParentRunner.access$000(ParentRunner.java:58)
	at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:268)
	at org.junit.runners.ParentRunner.run(ParentRunner.java:363)
	at org.junit.runner.JUnitCore.run(JUnitCore.java:137)
	at com.intellij.junit4.JUnit4IdeaTestRunner.startRunnerWithArgs(JUnit4IdeaTestRunner.java:68)
	at com.intellij.rt.execution.junit.IdeaTestRunner$Repeater.startRunnerWithArgs(IdeaTestRunner.java:47)
	at com.intellij.rt.execution.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:242)
	at com.intellij.rt.execution.junit.JUnitStarter.main(JUnitStarter.java:70)
Caused by: java.io.FileNotFoundException: HADOOP_HOME and hadoop.home.dir are unset.
	at org.apache.hadoop.util.Shell.checkHadoopHomeInner(Shell.java:448)
	at org.apache.hadoop.util.Shell.checkHadoopHome(Shell.java:419)
	at org.apache.hadoop.util.Shell.<clinit>(Shell.java:496)
	... 31 more

```
### 寻找问题

百度了下，找到了问题：

{% blockquote  Hadoop Wiki ——  https://wiki.apache.org/hadoop/WindowsProblems %}
**Problems running Hadoop on Windows**

Hadoop requires native libraries on Windows to work properly -that includes to access the file:// filesystem, where Hadoop uses some Windows APIs to implement posix-like file access permissions.

This is implemented in HADOOP.DLL and WINUTILS.EXE.

In particular, %HADOOP_HOME%\BIN\WINUTILS.EXE must be locatable.

If it is not, Hadoop or an application built on top of Hadoop will fail.

**How to fix a missing WINUTILS.EXE**

You can fix this problem in two ways

* Install a full native windows Hadoop version. The ASF does not currently (September 2015) release such a version; releases are available externally.
* Or: get the WINUTILS.EXE binary from a Hadoop redistribution. There is a repository of this for some Hadoop versions  <a rel="external nofollow" target="_blank" href="https://github.com/steveloughran/winutils">on github</a>.

Then

* Set the environment variable %HADOOP_HOME% to point to the directory above the BIN dir containing WINUTILS.EXE.
* Or: run the Java process with the system property hadoop.home.dir set to the home directory.

{% endblockquote %}

上面的意思是说Hadoop使用一些Windows api来实现文件访问。

必要 hadoop.DLL和WINUTILS.EXE，这两个文件。

还需要配置 ``% HADOOP_HOME %``的环境变量，来定位 ``WINUTILS.EXE``;

解决办法就是去上面它给的GitHub上 下载对应版本的 文件，将 adoop.DLL和WINUTILS.EXE 文件拷到本地 （Windows）的``Hadoop`` 文件夹下的``bin``文件夹中。

### 解决问题
#### 在Windows 上配置本地Hadoop 环境
##### 本地安装Hadoop
 将对应版本的 Hadoop 压缩包，拷一份到Windows 电脑的D盘中解压，我的是Hadoop2.8.1 版本的，将``hadoop-2.8.1.tar.gz`` 解压完就是这样的：

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwx6opyhnj20hx098jrv.jpg)

然后将自己从上面引用地址 GitHub 中 下载对应版本的文件，将  hadoop.DLL和WINUTILS.EXE 拷贝到 bin 目录中。

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwx7fllvaj20hr09wdgl.jpg)

修改 ``/etc/hadoop/hadoop-env.cmd`` 文件中
```
set JAVA_HOME=%JAVA_HOME%
```
为（修改成自己机器配置的JDK位置）
```
set JAVA_HOME=C:\Program Files\Java\jdk1.8.0_144
```

**需要注意的是我这个配置还有个小问题，并不能成功使用Hadoop 命令。这个将在文章最后面讲出原因。**

查看 /etc/hadoop/core-site.xml 中``fs.default.name``是不是的属性值是不是和服务器中一致。不一致需要改成一致。
```xml
<configuration>
<property>
        <name>hadoop.tmp.dir</name>
        <value>/root/hadoop/tmp</value>
        <description>Abase for other temporary directories.</description>
   </property>
   <property>
        <name>fs.default.name</name>
        <value>hdfs://server1:9000</value>
   </property>
</configuration>
```


##### 配置Hadoop环境变量

新增环境变量 ``HADOOP_HOME`` ，变量值为 ``D:\hadoop-2.8.1``

环境变量``Path`` 中新增 ``%HADOOP_HOME%\bin``

##### 配置本地Hosts
需要在C:\Windows\System32\drivers\etc\hosts 文件配置 ip，例如：使用 HDFS 的时候我们机器的配置文件中的地址是：``hdfs://server1:9000`` ，但是本地电脑没配置Hosts 的话，找不到 server1 的机器。

新增我的三台机器的集群信息
```
192.168.19.185 server1
192.168.19.184 server2
192.168.19.199 server3
```

这样下来，再次本地（Windows）调试虚拟机中Hadoop 集群就不会出现开头的问题了。

### 最后说下中途说的那个问题
我在 ``/etc/hadoop/hadoop-env.cmd`` 文件中 修改成这样的：
```bash
set JAVA_HOME=C:\Program Files\Java\jdk1.8.0_144
```
但是Windows 下的 ``CMD`` 或者``PowerShell`` 并不能成功使用Hadoop 命令，会报错：
```bash
PS C:\Users\server> hadoop version
系统找不到指定的路径。
Error: JAVA_HOME is incorrectly set.
       Please update D:\hadoop-2.8.1\etc\hadoop\hadoop-env.cmd
'-Xmx512m' 不是内部或外部命令，也不是可运行的程序
或批处理文件。
```

报错，我们设置的``JAVA_HOME`` 位置并不正确。

这个问题很奇怪，因为我的这个JDK 位置用过很多次了，可以肯定没问题。

在网上找到了问题所在，不过还是需要自己改。。

>if your java environment path contains space, such as "C:\Program Files\java\xxxxx" , the word 《Program Files》 contains a space, so CMD can't identificate
this is the right answer

``Program Files``，就是这个我们安装软件默认的路径，有空格，CMD 不能识别它，导致我的位置失效了。所以设置路径的时候不能有空格。
