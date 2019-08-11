---
title: Spring 中使用定时任务
date: 2018-03-15 23:12:03
tags: [Spring]
categories: 学习笔记
---

### 前言
需要定时用于系统监控，或者做一些定时自动做一些操作，使用起来还是蛮简单的，就是配置参数需要记录下。
<!--more-->
Spring 中自己内置的定时任务
### 正文
#### Spring 中使用
在 Spring 中使用定时任务有两种方式：
* 注解方式；
* 配置文件。

##### 注解方式
注解方式方式配置起来非常方便，但是，如果要修改定时任务的时间计划，就需要修改源代码，
1. 在 `application.xml` 文件中添加开启定时任务的注解：
```xml
 <task:annotation-driven />
```
2. 如果使用的使用的 IDEA 可以自动添加配置文件的 `空间命名`、`xml 标签规范`和命名空间相应的验证文件，在 Eclipse 中将视图从 `source` 切换到 `Namespaces` 勾选自动添加相应命名空间。
```xml
xmlns:task="http://www.springframework.org/schema/task"
xsi:schemaLocation="http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task.xsd"
```
3. 新建定时任务计划，使用注解方式：
```java
@Component
public class ScheduleJob {
    /**
     * 每 2s 执行一次，
     * 函数不能有返回值，不然会报错
     */
    @Scheduled(fixedDelay = 2000)
    public void operateScheduleJob() {
        System.out.println(">>>>>>>> 执行定时任务！");
    }
}
```

##### 使用配置文件
1. 新建定时任务计划类，不适用任何注解。
```java
public class ScheduleJob {
    /**
     * 函数不能有返回值，不然会报错
     */
    public void operateScheduleJob() {
        System.out.println(">>>>>>>> 执行定时任务！");
    }
}
```
2. 在 `application.xml` 中上面的给出命名空间校验地址，后在配置文件中注册相应的实体类和方法
```xml
	<bean id="task" class="xxx.xxx.schedule.ScheduleJob"/>
	<task:scheduled-tasks>
		<!-- 这里表示的是每天0点执行一次 -->
		<task:scheduled ref="task" method="operateScheduleJob" cron="0 0 0 * * ?" />
	</task:scheduled-tasks>
```
#### Spring Boot 中使用
在程序启动类上面加上注解 `EnableScheduling`，表示开启定时任务功能，然后就能在程序中和上面 `Spring` 一样使用注解的方式定义定时任务。

#### 关于 @Schedule 注解
**它有多种定时规则表达式：**
* `cron`：指定cron表达式
* `zone`：默认使用服务器默认时区。可以设置为`java.util.TimeZone`中的zoneId
* `fixedDelay`：从上一个任务完成开始到下一个任务开始的间隔，单位毫秒
* `fixedDelayString`：同上，时间值是String类型
* `fixedRate`：从上一个任务开始到下一个任务开始的间隔，单位毫秒
* `fixedRateString`：同上，时间值是String类型
* `initialDelay`：任务首次执行延迟的时间，单位毫秒
* `initialDelayString`：同上，时间值是String类型
#### cron 表达式
Cron表达式是一个字符串，字符串以5或6个空格隔开，分为6或7个域，每一个域代表一个含义，Cron有如下两种语法格式：
* Seconds Minutes Hours DayofMonth Month DayofWeek Year
* Seconds Minutes Hours DayofMonth Month DayofWeek

**每一个域可出现的字符如下：**
* `Seconds`: 可出现 `, - * /` 四个字符，有效范围为`0-59`的整数
* `Minutes`: 可出现 `, - * /` 四个字符，有效范围为`0-59`的整数
* `Hours`: 可出现 `, - * /` 四个字符，有效范围为`0-23`的整数
* `DayofMonth`: 可出现 `, - * / ? L W C` 八个字符，有效范围为`0-31`的整数
* `Month`: 可出现 `, - * /`四个字符，有效范围为1-12的整数或`JAN-DEC`
* `DayofWeek`: 可出现 `, - * / ? L C #` 四个字符，有效范围为`1-7`的整数或`SUN-SAT`两个范围。1表示星期天，2表示星期一， 依次类推
* `Year`: 可出现 `, - * /` 四个字符，有效范围为`1970-2099`年

**每一个域都使用数字，但还可以出现如下特殊字符，它们的含义是：**

* `*`：表示匹配该域的任意值，假如在`Minutes`域使用`*`, 即表示每分钟都会触发事件。
* `?`：只能用在`DayofMonth`和`DayofWeek`两个域。它也匹配域的任意值，但实际不会。因为DayofMonth和 DayofWeek会相互影响。例如想在每月的20日触发调度，不管20日到底是星期几，则只能使用如下写法： 13 13 15 20 * ?, 其中最后一位只能用？，而不能使用，如果使用表示不管星期几都会触发，实际上并不是这样。
* `-`：表示范围，例如在Minutes域使用5-20，表示从5分到20分钟每分钟触发一次。
* `/`：表示起始时间开始触发，然后每隔固定时间触发一次，例如在Minutes域使用5/20,则意味着5分钟触发一次，而25，45等分别触发一次。
* `,`：表示列出枚举值值。例如：在Minutes域使用5,20，则意味着在5和20分每分钟触发一次。
* `L`：表示最后，只能出现在DayofWeek和DayofMonth域，如果在DayofWeek域使用5L,意味着在最后的一个星期四触发。
* `W`：表示有效工作日(周一到周五),只能出现在DayofMonth域，系统将在离指定日期的最近的有效工作日触发事件。例如：在 DayofMonth使用5W，如果5日是星期六，则将在最近的工作日：星期五，即4日触发。如果5日是星期天，则在6日(周一)触发；如果5日在星期一 到星期五中的一天，则就在5日触发。另外一点，W的最近寻找不会跨过月份。
* `LW`：这两个字符可以连用，表示在某个月最后一个工作日，即最后一个星期五。
* `#`：用于确定每个月第几个星期几，只能出现在DayofMonth域。例如在`4#2`，表示某月的第二个星期三。

### Reference
* [Spring Task定时任务的配置和使用](https://www.jianshu.com/p/25c601f43552)](https://github.com/kaimz/learning-code/tree/master/jpa-muti-datasource)
