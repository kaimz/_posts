---
title: 学习Spring Boot：（四）应用日志
date: 2018-02-23 10:08:05
tags: [Spring Boot]
categories: 学习笔记

---

## 前言
应用日志是一个系统非常重要的一部分，后来不管是开发还是线上，日志都起到至关重要的作用。这次使用的是 `Logback` 日志框架。

##  正文
Spring Boot在所有内部日志中使用Commons Logging，但是默认配置也提供了对常用日志的支持，如：Java Util Logging，Log4J, Log4J2和Logback。每种Logger都可以通过配置使用控制台或者文件输出日志内容。

### 默认日志Logback
* **SLF4J**——Simple Logging Facade For Java，它是一个针对于各类Java日志框架的统一Facade抽象。Java日志框架众多——常用的有java.util.logging, log4j, logback，commons-logging, Spring框架使用的是Jakarta Commons Logging API (JCL)。而SLF4J定义了统一的日志抽象接口，而真正的日志实现则是在运行时决定的——它提供了各类日志框架的binding。

* **Logback**是log4j框架的作者开发的新一代日志框架，它效率更高、能够适应诸多的运行环境，同时天然支持SLF4J。

默认情况下，Spring Boot会用Logback来记录日志，并用INFO级别输出到控制台。在运行应用程序和其他例子时，你应该已经看到很多INFO级别的日志了。
![springboot4-1.png](https://i.loli.net/2018/01/21/5a6475f94024e.png)

从上图可以看到，日志输出内容元素具体如下：
* 时间日期：精确到毫秒
* 日志级别：ERROR, WARN, INFO, DEBUG or TRACE
* 进程ID
* 分隔符：`---` 标识实际日志的开始
* 线程名：方括号括起来（可能会截断控制台输出）
* Logger名：通常使用源代码的类名
* 日志内容

### 添加日志依赖
假如maven依赖中添加了spring-boot-starter-logging：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
</dependency>
```
那么，我们的Spring Boot应用将自动使用logback作为应用日志框架，Spring Boot启动的时候，由org.springframework.boot.logging.Logging-Application-Listener根据情况初始化并使用。

**但是呢，实际开发中我们不需要直接添加该依赖，你会发现spring-boot-starter其中包含了 spring-boot-starter-logging，该依赖内容就是 Spring Boot 默认的日志框架 logback。**

也就是我们不需要额外配置。

### 默认配置属性支持
Spring Boot为我们提供了很多默认的日志配置，所以，只要将spring-boot-starter-logging作为依赖加入到当前应用的classpath，则“开箱即用”。
下面介绍几种在application.properties就可以配置的日志相关属性。

#### 控制台输出
日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL，如果设置为WARN，则低于WARN的信息都不会输出。
Spring Boot中默认配置ERROR、WARN和INFO级别的日志输出到控制台。您还可以通过启动您的应用程序–debug标志来启用“调试”模式（开发的时候推荐开启）,以下两种方式皆可：

在运行命令后加入`--debug`标志，如：``$ java -jar springTest.jar --debug``
在`application.properties`中配置`debug=true`，该属性置为true的时候，核心Logger（包含嵌入式容器、hibernate、spring）会输出更多内容，但是你自己应用的日志并不会输出为DEBUG级别。
##### 文件输出
默认情况下，Spring Boot将日志输出到控制台，不会写到日志文件。如果要编写除控制台输出之外的日志文件，则需在application.properties中设置logging.file或logging.path属性。

* **logging.file**，设置文件，可以是绝对路径，也可以是相对路径。如：logging.file=my.log
* **logging.path**，设置目录，会在该目录下创建spring.log文件，并写入日志内容，如：logging.path=/var/log
  如果只配置 logging.file，会在项目的当前路径下生成一个 xxx.log 日志文件。
  如果只配置 logging.path，在 /var/log文件夹生成一个日志文件为 spring.log
>注：二者不能同时使用，如若同时使用，则只有logging.file生效

**默认情况下，日志文件的大小达到10MB时会切分一次，产生新的日志文件，默认级别为：ERROR、WARN、INFO**

#### 级别控制
所有支持的日志记录系统都可以在Spring环境中设置记录级别（例如在application.properties中）
格式为：’logging.level.* = LEVEL’

* **logging.level**：日志级别控制前缀，*为包名或Logger名
* **LEVEL**：选项TRACE, DEBUG, INFO, WARN, ERROR, FATAL, OFF
  举例：

* logging.level.com.dudu=DEBUG：`com.dudu`包下所有class以DEBUG级别输出
* logging.level.root=WARN：root日志以WARN级别输出

#### 自定义日志配置
由于日志服务一般都在ApplicationContext创建前就初始化了，它并不是必须通过Spring的配置文件控制。因此通过系统属性和传统的Spring Boot外部配置文件依然可以很好的支持日志控制和管理。

根据不同的日志系统，你可以按如下规则组织配置文件名，就能被正确加载：
* Logback：logback-spring.xml, logback-spring.groovy, logback.xml, logback.groovy
* Log4j：log4j-spring.properties,log4j-spring.xml, log4j.properties, log4j.xml
* Log4j2：log4j2-spring.xml, log4j2.xml
* JDK (Java Util Logging)：logging.properties

**Spring Boot官方推荐优先使用带有-spring的文件名作为你的日志配置（如使用logback-spring.xml，而不是logback.xml），命名为logback-spring.xml的日志配置文件，spring boot可以为它添加一些spring boot特有的配置项（下面会提到）。**

上面是默认的命名规则，并且放在`src/main/resources`下面即可。

## logback.xml

### 基本配置

如果你即想完全掌控日志配置，但又不想用`logback.xml`作为`Logback`配置的名字，可以在`application.properties`配置文件里面通过`logging.config`属性指定自定义的名字：
```
logging.config=classpath:logging-config.xml

```
虽然一般并不需要改变配置文件的名字，但是如果你想针对不同运行时Profile使用不同的日
志配置，这个功能会很有用。

下面我们来看看一个普通的logback-spring.xml例子
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration  scan="true" scanPeriod="60 seconds" debug="false">
    <contextName>logback</contextName>
    <property name="log.path" value="/Users/tengjun/Documents/log" />
    <!--输出到控制台-->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
       <!-- <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>-->
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %-5level %logger{36}.%M:%L - %msg%n</pattern>
        </encoder>
    </appender>

    <!--输出到文件-->
    <appender name="file" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/logback.%d{yyyy-MM-dd}.log</fileNamePattern>
        </rollingPolicy>
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %-5level %logger{36}.%M:%L - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="console" />
        <appender-ref ref="file" />
    </root>

    <!-- logback为java中的包 -->
    <logger name="com.dudu.controller"/>
    <!--logback.LogbackDemo：类的全路径 -->
    <logger name="com.dudu.controller.LearnController" level="WARN" additivity="false">
        <appender-ref ref="console"/>
    </logger>
</configuration>
```

### 属性分析
#### 根节点<configuration>
**根节点`<configuration>`包含的属性**:
* scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true。
* scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。
* debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。

**根节点<configuration>的子节点：
<configuration>下面一共有2个属性，3个子节点，分别是**：
* 设置上下文名称`<contextName>`
  每个logger都关联到logger上下文，默认上下文名称为“default”。但可以使用设置成其他名字，用于区分不同应用程序的记录。一旦设置，不能修改,可以通过%contextName来打印日志上下文名称。
```xml
<contextName>logback</contextName>
```
* 设置变量<property>
  用来定义变量值的标签， 有两个属性，name和value；其中name的值是变量的名称，value的值时变量定义的值。通过定义的值会被插入到logger上下文中。定义变量后，可以使“${}”来使用变量。

#### 子节点一`<appender>`

**appender用来格式化日志输出节点，有俩个属性`name`和`class`，class用来指定哪种输出策略，常用就是控制台输出策略和文件输出策略。**

##### 控制台输出ConsoleAppender：
```xml
<!--输出到控制台-->
<appender name="console" class="ch.qos.logback.core.ConsoleAppender">
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>ERROR</level>
    </filter>
    <encoder>
        <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %-5level %logger{36}.%M:%L - %msg%n</pattern>
    </encoder>
</appender>
```
`<encoder>`表示对日志进行编码：
* %d{HH: mm:ss.SSS}——日志输出时间
* %thread——输出日志的进程名字，这在Web应用以及异步任务处理中很有用
* %-5level——日志级别，并且使用5个字符靠左对齐
* %logger{36}——日志输出者的名字
* %msg——日志消息
* %n——平台的换行符

ThresholdFilter为系统定义的拦截器，例如我们用ThresholdFilter来过滤掉ERROR级别以下的日志不输出到文件中。如果不用记得注释掉，不然你控制台会发现没日志~
##### 输出到文件RollingFileAppender
另一种常见的日志输出到文件，随着应用的运行时间越来越长，日志也会增长的越来越多，将他们输出到同一个文件并非一个好办法。`RollingFileAppender`用于切分文件日志：
```xml
<!--输出到文件-->
<appender name="file" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
		<fileNamePattern>${log.path}/logback.%d{yyyy-MM-dd}.log</fileNamePattern>
        <maxHistory>30</maxHistory>
		<totalSizeCap>1GB</totalSizeCap>
    </rollingPolicy>
    <encoder>
        <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %-5level %logger{36}.%M:%L - %msg%n</pattern>
    </encoder>
</appender>
```

其中重要的是rollingPolicy的定义，上例中<fileNamePattern>${log.path}/logback.%d{yyyy-MM-dd}.log</fileNamePattern>定义了日志的切分方式——把每一天的日志归档到一个文件中，<maxHistory>30</maxHistory>表示只保留最近30天的日志，以防止日志填满整个磁盘空间。同理，可以使用%d{yyyy-MM-dd_HH-mm}来定义精确到分的日志切分方式。<totalSizeCap>1GB</totalSizeCap>用来指定日志文件的上限大小，例如设置为1GB的话，那么到了这个值，就会删除旧的日志。

补:如果你想把日志直接放到当前项目下，把${log.path}/去掉即可。

logback 每天生成和大小生成冲突的问题可以看这个解答：[传送门](http://blog.csdn.net/wujianmin577/article/details/68922545)

#### 子节点二<root>
root节点是必选节点，用来指定最基础的日志输出级别，只有一个level属性。

level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，不能设置为INHERITED或者同义词NULL。
默认是DEBUG。
可以包含零个或多个元素，标识这个appender将会添加到这个logger。

```xml
<root level="debug">
    <appender-ref ref="console" />
    <appender-ref ref="file" />
</root>
```
#### 子节点三<logger>
<logger>用来设置某一个包或者具体的某一个类的日志打印级别、以及指定<appender>。<logger>仅有一个name属性，一个可选的level和一个可选的addtivity属性。

* name:用来指定受此logger约束的某一个包或者具体的某一个类。
* level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，还有一个特俗值INHERITED或者同义词NULL，代表强制执行上级的级别。如果未设置此属性，那么当前logger将会继承上级的级别。
* addtivity:是否向上级logger传递打印信息。默认是true。

注：使用mybatis的时候，sql语句是debug下才会打印，而这里我们只配置了info，所以想要查看sql语句的话，有以下两种操作：
* 第一种把<root level="info">改成<root level="DEBUG">这样就会打印sql，不过这样日志那边会出现很多其他消息。
* 第二种就是单独给dao下目录配置debug模式，代码如下，这样配置sql语句会打印，其他还是正常info级别：
```xml
<logger name="com.dudu.dao" level="DEBUG" additivity="false">
    <appender-ref ref="console" />
</logger>
```
## 多环境日志输出
据不同环境（prod:生产环境，test:测试环境，dev:开发环境）来定义不同的日志输出，在 logback-spring.xml中使用 springProfile 节点来定义，方法如下：
>文件名称不是logback.xml，想使用spring扩展profile支持，要以logback-spring.xml命名:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml" />
    <logger name="org.springframework.web" level="INFO"/>
    <logger name="org.springboot.sample" level="TRACE" />

    <!-- 开发、测试环境，多个用逗号隔开 -->
    <springProfile name="dev,test">
        <logger name="org.springframework.web" level="INFO"/>
        <logger name="org.springboot.sample" level="INFO" />
        <logger name="com.wuwii" level="DEBUG" additivity="false" />
    </springProfile>

    <!-- 生产环境 -->
    <springProfile name="pro">
        <logger name="org.springframework.web" level="ERROR"/>
        <logger name="org.springboot.sample" level="ERROR" />
        <logger name="com.wuwii" level="ERROR" additivity="false" />
    </springProfile>

</configuration>
```
可以启动服务的时候指定 profile （如不指定使用默认），如指定prod 的方式为：
java -jar xxx.jar –spring.profiles.active=prod

## 格式化颜色
注意到Spring Boot 输出日志到控制台按日志级别有不同的颜色，logback官方也给出了解释：
<a rel="external nofollow" target="_blank" href="https://logback.qos.ch/manual/layouts.html#coloring">https://logback.qos.ch/manual/layouts.html#coloring</a>

我配置了一个这样的 ：
```xml
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %highlight(%-5level) %cyan(%logger{36}).%M:%L - %msg%n</pattern>
        </encoder>
```
