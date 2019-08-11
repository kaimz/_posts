---
title: Java中Log4j的使用及配置详情
date: 2017-10-16 22:47:03
tags: [java]
categories: 学习笔记
---

>``Log4j``是Apache的一个开源项目，通过使用Log4j，我们可以控制日志信息输送的目的地是控制台、文件、GUI组件，甚至是套接口服务器、NT的事件记录器、UNIX Syslog守护进程等；我们也可以控制每一条日志的输出格式；通过定义每一条日志信息的级别，我们能够更加细致地控制日志的生成过程。最令人感兴趣的就是，这些可以通过一个配置文件来灵活地进行配置，而不需要修改应用的代码。

项目中日志功能十分强大，可以实时监控你的代码的运行情况，并且就像书页一样清晰可见。
<!--more-->
#### 环境
首先在pom.xml 配置好相关依赖，我这里只使用Log4j，当然还可以使用slf4j 可以管理，
```xml
<log4j.version>1.2.16</log4j.version>
        <dependency>
			<groupId>log4j</groupId>
			<artifactId>log4j</artifactId>
			<version>${log4j.version}</version>
		</dependency>
```

在web.xml 监听 log4j.properties
```xml
<!-- 启动Log4j -->
	<context-param>
		<param-name>log4jConfigLocation</param-name>
		<param-value>classpath:log4j.properties</param-value>
	</context-param>
	<listener>
		<listener-class>org.springframework.web.util.Log4jConfigListener</listener-class>
	</listener>
```
#### 配置log4j.properties 配置文件

```properties
log4j.rootLogger=DEBUG, stdout , R  
log4j.appender.stdout=org.apache.log4j.ConsoleAppender  
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout  
log4j.appender.stdout.layout.ConversionPattern=[QC] %p [%t] %C.%M(%L) | %m%n  
log4j.appender.R=org.apache.log4j.DailyRollingFileAppender  
log4j.appender.R.File=d://log//FTASWorkFlow.log  
log4j.appender.R.layout=org.apache.log4j.PatternLayout  
log4j.appender.R.layout.ConversionPattern=%d-[TS] %p %t %c - %m%n</span>  
```
##### 说明
``rootLogger``也可以写作``rootCategory``,   
rootLogger value的含义
第一个逗号前表示log的级别：``FATAL``,``ERROR``,``WARN``,``INFO``,``DEBUG``,级别依次降低，开发的时候一般选作DEBUG，上线前期可以INFO或者DEBUG，版本稳定了可以WARN或者ERROR。稳定以后可以每天将日志发送到你的邮箱（至于怎么发，看最下面的Appender），这样就不需要每天去看检查上线的项目有没有异常。

第一个逗号后面的表示你定义的``appender``，比如我们这里定义了stdout和R，这个名字可以随便定，和下面的对应就好了。这里的stdout代表控制台输出，上线的时候别忘记关掉，直接在rootLogger里去掉stdout就好了。

##### Log4j提供的appender有以下几种：
```
org.apache.log4j.ConsoleAppender（控制台）
org.apache.log4j.FileAppender（文件）
org.apache.log4j.DailyRollingFileAppender（每天产生一个日志文件）
org.apache.log4j.RollingFileAppender（文件大小到达指定尺寸的时候产生新文件）
org.apache.log4j.WriterAppender（将日志信息以流格式发送到任意指定的地方）
ConsoleAppender和DailyRollingFileAppender以及RollingFileAppender用的比较多，后面两个用哪个看需求。
```
##### log4j提供以下4种布局样式：
不同的Appender有不同的属性，但是Appender都会有一个属性layout，layout又有一个属性PatternLayout 
```
org.apache.log4j.HTMLLayout（以HTML表格形式布局）
org.apache.log4j.PatternLayout（可以灵活地指定布局模式，就是可以自定义输出样式），
org.apache.log4j.SimpleLayout（包含日志信息的级别和信息字符串），
org.apache.log4j.TTCCLayout（包含日志产生的时间、线程、类别等等信息）
```
##### 再看一下PatternLayout的值代表的什么意思
``%d`` 输出日志时间点的日期或时间，紧跟一对花括号进行自定义格式  
``%t`` 输出产生该日志事件的线程名  
``%c`` 输出所属的类目，通常就是所在类的全名  
``%l``  输出行号  
``%m`` 输出代码中指定的消息  
``%n`` 输出一个回车换行符，Windows平台为 ``\r\n``，Unix平台为 ``\n``，也就是一跳消息占用一行，所以``%m%n``基本都是一起用  
``%p`` 输出优先级，即DEBUG，INFO，WARN，ERROR，FATAL  
   我们经常会看到[%-5p]这样的用法，就是对%p进行格式化，占用几个字符空间，因为INFO，DEBUG他们有的占用4个有的占用5个，日志看起来不对其，进行一个格式化而已。  
``%r`` 输出自应用启动到输出该log信息耗费的毫秒数  
``%c`` 输出所属的类目，通常就是所在类的全名  
``%x`` 输出对齐  

##### 再看看appender的其他属性
```
log4j.appender.FILE.File=D:/logs/log4j.log      --------定义输出文件的位置及文件名
log4j.appender.FILE.MaxFileSize=1MB             --------定义每个文件的大小，超过这个大小，则新建一个文件，注意单位 MB 或 KB
log4j.appender.D.Threshold = DEBUG              --------输出DEBUG级别以上的日志
```
##### 输出到邮件
```
log4j.appender.MAIL=org.apache.log4j.net.SMTPAppender（指定输出到邮件）
log4j.appender.MAIL.Threshold=FATAL
log4j.appender.MAIL.BufferSize=10
log4j.appender.MAIL.From=chenyl@hollycrm.com（发件人）
log4j.appender.MAIL.SMTPHost=mail.hollycrm.com（SMTP服务器）
log4j.appender.MAIL.Subject=Log4J Message
log4j.appender.MAIL.To=chenyl@hollycrm.com（收件人）
log4j.appender.MAIL.layout=org.apache.log4j.PatternLayout（布局）
log4j.appender.MAIL.layout.ConversionPattern=[framework] %d - %c -%-4r [%t] %-5p %c %x - %m%n（格式）
 
输出到数据库
log4j.appender.DATABASE=org.apache.log4j.jdbc.JDBCAppender（指定输出到数据库）
log4j.appender.DATABASE.URL=jdbc:mysql://localhost:3306/test（指定数据库URL）
log4j.appender.DATABASE.driver=com.mysql.jdbc.Driver（指定数据库driver）
log4j.appender.DATABASE.user=root（指定数据库用户）
log4j.appender.DATABASE.password=root（指定数据库用户密码）
log4j.appender.DATABASE.sql=INSERT INTO LOG4J (Message) VALUES ('[framework] %d - %c -%-4r [%t] %-5p %c %x - %m%n')（组织SQL语句）
log4j.appender.DATABASE.layout=org.apache.log4j.PatternLayout（布局）
log4j.appender.DATABASE.layout.ConversionPattern=[framework] %d - %c -%-4r [%t] %-5p %c %x - %m%n（格式）
```
##### 我的项目最终配置
```
# Rules reminder:
# DEBUG < INFO < WARN < ERROR < FATAL

### 设置级别和目的地(这里多个目的地) ###
#级别为DEBUG
#目的地为CONSOLE，zhangLog；zhangLog为自定义输出端，可随意命名
log4j.rootLogger = DEBUG,CONSOLE,zhangLog

### 这里的com.wuwii是我项目的包名，也就是在这个包记录日志时，开发阶段是只记录DEBUG及以上级别的日志，正式上线的时候可以改成INFO、ERROR
#### 当然就可以设定特定包打印的级别
log4j.logger.com.wuwii=DEBUG

#Log4j提供的appender有以下几种：
#org.apache.log4j.ConsoleAppender（控制台），
#org.apache.log4j.FileAppender（文件），
#org.apache.log4j.DailyRollingFileAppender（每天产生一个日志文件），
#org.apache.log4j.RollingFileAppender（文件大小到达指定尺寸的时候产生一个新的文件），
#org.apache.log4j.WriterAppender（将日志信息以流格式发送到任意指定的地方）

### 输出到控制台 ###
log4j.appender.CONSOLE = org.apache.log4j.ConsoleAppender
log4j.appender.CONSOLE.Target = System.out
log4j.appender.CONSOLE.layout = org.apache.log4j.PatternLayout
log4j.appender.CONSOLE.layout.ConversionPattern =  %d{ABSOLUTE} %5p %c{1}:%L - %m%n

# My logging configuration...
## 可以设置特定工具的打印日志级别
log4j.logger.org.mybatis.jpetstore=INFO
log4j.logger.com.ibatis=INFO
log4j.logger.com.ibatis.common.jdbc.SimpleDataSource=INFO
log4j.logger.com.ibatis.common.jdbc.ScriptRunner=INFO
log4j.logger.com.ibatis.sqlmap.engine.impl.SqlMapClientDelegate=INFO
log4j.logger.java.sql.Connection = INFO
log4j.logger.java.sql.Statement = INFO
log4j.logger.java.sql.PreparedStatement = INFO
log4j.logger.java.sql.ResultSet = INFO

### 输出到日志文件 ###
#写到文件中，并且追加
log4j.appender.zhangLog = org.apache.log4j.DailyRollingFileAppender
# 设置文件输出位置
#log4j.appender.zhangLog.File =D\:\\debug.log
log4j.appender.zhangLog.File=${catalina.home}/logs/wuwii/debug.log
#log4j.appender.zhangLog.File =/var/debug/debug.log
log4j.appender.zhangLog.Append = true
## 只输出DEBUG级别以上的日志
log4j.appender.zhangLog.Threshold = DEBUG
#'.'yyyy-MM-dd: 设置为每天产生一个新的文件
#1)’.’yyyy-MM: 每月
#2)’.’yyyy-ww: 每周
#3)’.’yyyy-MM-dd: 每天
#4)’.’yyyy-MM-dd-a: 每天两次
#5)’.’yyyy-MM-dd-HH: 每小时
#6)’.’yyyy-MM-dd-HH-mm: 每分钟
log4j.appender.zhangLog.DatePattern = '.'yyyy-MM-dd
#当文件达到2kb时，文件会被备份成"debug.txt.1"，新的"log.txt"继续记录log信息
## 在DailyRollingFileAppender 没这个属性
#log4j.appender.zhangLog.MaxFileSize = 2KB 
#最多建5个文件，当文件个数较多时，后面不再新建文件
## 在DailyRollingFileAppender 没这个属性
#log4j.appender.zhangLog.MaxBackupIndex = 5
log4j.appender.zhangLog.layout = org.apache.log4j.PatternLayout
log4j.appender.zhangLog.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss} [%t:%r] - [%p] [%c{1}:%L] [%M] %m%n
#设置子Logger是否继承父Logger的输出源
#默认情况下子Logger会继承父Logger的appender，也就是说子Logger会在父Logger的appender里输出
log4j.additivity.zhangLog = false
```

#### 测试
测试的类没有启动 web ，默认的是查找 resources 根目录下的  ``log4j.properties`` ，没有则找不到。
```
package com.wuwii.test;
import org.apache.log4j.Logger;

public class Log4jTest {
    public static Logger logger1 = Logger.getLogger(Log4jTest.class);
    public static void main(String[] args) {
        //logger1
        logger1.trace("他真的很喜欢你 像春雨下得淅淅沥沥，trace");
        logger1.debug("他真的很喜欢你 像夏日聒噪的蝉鸣，debug");
        logger1.info("他真的很想念你 像秋叶落得悄无声息，info");
        logger1.warn("他真的很喜欢你 想冬天的雪沁在心里，warn");
        logger1.error("他真的很喜欢你 像狗本性难移，error");
        logger1.fatal("他真的很喜欢你 所以他可以一直没脸没皮，fatal");
    }
}

```
运行代码后，我们可以看到控制台打印了：

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwwarhai6j210t07otak.jpg)

因为我们设置了输入到控制台了，再去查看我们的打印日志文件的位置，也可以看到报错信息，使用的 是``org.apache.log4j.DailyRollingFileAppender``，并没有 ``maxBackupIndex`` 和 ``maxFileSize`` 属性，所以上面的配置文件也不正确，需要删掉这两行，

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwwber01uj20mh01swee.jpg)

使用的是每天生成一个文件，前一天的备份成``yyyy-MM-dd`` 符合。

打开文件看到

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwwbpurptj20xg04e3zc.jpg)

正确写入，

Log4j的使用及配置就是这样的了。

**参考博客** <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/zhengliusu/article/details/44619023">http://blog.csdn.net/zhengliusu/article/details/44619023</a>

