---
title: 学习Spring Boot：（三）配置文件
date: 2018-02-23 10:08:03
tags: [Spring Boot]
categories: 学习笔记
---

### 前言
Spring Boot使用``习惯优于配置``（项目中存在大量的配置，此外还内置了一个习惯性的配置，让你无需手动进行配置）的理念让你的项目快速运行起来。

<!--more-->

### 正文
#### 使用配置文件注入属性
Spring Boot 默认的配置文件`src/main/java/resources/application.properties`或者`src/main/java/resources/application.yml`，在这里我们可以配置一些常量。
首先我们使用配置文件给一个类注入相关的属性：
```
com.wuwii.controller.pet.no=${random.uuid}
com.wuwii.controller.pet.name=Tom
```
通过注解`@Value(value=”${config.name}”)`就可以绑定到你想要的属性上面。
```java
@RestController
@RequestMapping("/pet")
public class PetController {
    @Value("${com.wuwii.controller.pet.no}")
    private String no;
    @Value("${com.wuwii.controller.pet.name}")
    private String name;
    @RequestMapping("/d")
    public String detail() {
        return "no: " + no + ", name: " + name;
    }

}
```
启动
![](https://ws1.sinaimg.cn/large/ece1c17dgy1fno1evzf6sj20fv04n0sp.jpg)

一个个绑定数据还是很不方便，可以新建一个Bean，专门用来绑定注入的属性使用注解@ConfigurationProperties(prefix = "prefix")，不过需要注意的是先要引入相关依赖
```java
<dependency>  
     <groupId>org.springframework.boot</groupId>  
     <artifactId>spring-boot-configuration-processor</artifactId>  
     <optional>true</optional>  
</dependency>  
```
通过使用spring-boot-configuration-processor jar， 你可以从被@ConfigurationProperties注解的节点轻松的产生自己的配置元数据文件。

这里我新建一个PetBean用来注入属性。
```java
@ConfigurationProperties(prefix = "com.wuwii.controller.pet")
public class PetBean {
    private String no;
    private String name;

    public String getNo() {
        return no;
    }

    public void setNo(String no) {
        this.no = no;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

```
注意在启动类上加上注解
```java
@EnableConfigurationProperties({PetBean.class})，
```
根据字面意思不难理解，就是开启配置属性。

新建一个controller，注入我们创建的PetBean，
```java
@RestController
@RequestMapping("/v2/pet")
public class PetController1 {
    @Autowired
    private PetBean pet;
    @RequestMapping("/d")
    public String detail() {
        return "no: " + pet.getNo() + ", name: " + pet.getName();
    }
}

```
重启Spring Boot，访问新地址：
![](https://ws1.sinaimg.cn/large/ece1c17dgy1fno25n0o60j20hi043weg.jpg)

#### 使用自定义的配置文件
我们在resouce目录下面创建一个``bean/pet.properties``，加入
```
com.wuwii.name=Tom
com.wuwii.no=123456
```
新建一个PetBean1.java：
`@PropertySource` 这个注解可以指定具体的属性配置文件，优先级比较低。 
```java
@Configuration
@ConfigurationProperties(prefix = "com.wuwii")
@PropertySource("classpath:bean/pet.properties")
public class PetBean1 {
    private String no;
    private String name;

    public String getNo() {
        return no;
    }

    public void setNo(String no) {
        this.no = no;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

```

在controller中加入PetBean1的注入
```java
@Autowired
    private PetBean1 pet1;
    @RequestMapping("/d2")
    public String detail2() {
        return "no: " + pet1.getNo() + ", name: " + pet1.getName();
    }
```
#### 应用配置文件（.properties或.yml）
在配置文件中直接写：
```
server.port=8080
```
`.yml`格式的配置文件如：
```
server:
    port: 8080
```
tips: .yml中冒号后面一定要加一个空格。 
#### 随机数
配置文件中${random} 可以用来生成各种不同类型的随机值，
```
my.secret=${random.value}
my.number=${random.int}
my.bignumber=${random.long}
my.number.less.than.ten=${random.int(10)}
my.number.in.range=${random.int[1024,65536]}
```
#### 属性占位符
```
app.name=MyApp
app.description=${app.name} is a Spring Boot application
```
可以在配置文件中引用前面配置过的属性（优先级前面配置过的这里都能用）。
通过如${app.name:默认名称}方法还可以设置默认值，当找不到引用的属性时，会使用默认的属性。
#### 属性名匹配规则
例如有如下配置对象：
```java
@Component
@ConfigurationProperties(prefix="person")
public class ConnectionSettings {

    private String firstName;

}
```
`firstName`可以使用的属性名如下：
* ``person.firstName``，标准的驼峰式命名
* `person.first-name`，虚线（-）分割方式，推荐在.properties和.yml配置文件中使用
* `PERSON_FIRST_NAME`，大写下划线形式，建议在系统环境变量中使用

#### 属性验证
可以使用`JSR-303`注解进行验证，例如：
```java
@Component
@ConfigurationProperties(prefix="connection")
public class ConnectionSettings {

    @NotNull
    private InetAddress remoteAddress;

    // ... getters and setters

}
```
#### 配置文件的优先级
Spring Boot 支持多种外部配置方式，这些方式优先级如下：
1. 命令行参数
2. 来自`java:comp/env`的JNDI属性
3. Java系统属性（`System.getProperties()`）
4. 操作系统环境变量
5. RandomValuePropertySource配置的`random.*`属性值
6. jar包外部的`application-{profile}.properties`或`application.yml(带spring.profile)`配置文件
7. jar包内部的`application-{profile}.properties`或`application.yml(带spring.profile)`配置文件
8. jar包外部的`application.properties`或`application.yml(不带spring.profile)`配置文件
9. jar包内部的`application.properties`或`application.yml(不带spring.profile)`配置文件
10. `@Configuration`注解类上`的@PropertySource`
11. 通过`SpringApplication.setDefaultProperties`指定的默认属性


同样，这个列表按照优先级排序，也就是说，src/main/resources/config下application.properties覆盖src/main/resources下application.properties中相同的属性，此外，如果你在相同优先级位置同时有application.properties和application.yml，那么application.properties里的属性里面的属性就会覆盖application.yml。

#### Profile-多环境配置
当应用程序需要部署到不同运行环境时，一些配置细节通常会有所不同，最简单的比如日志，生产日志会将日志级别设置为WARN或更高级别，并将日志写入日志文件，而开发的时候需要日志级别为DEBUG，日志输出到控制台即可。
如果按照以前的做法，就是每次发布的时候替换掉配置文件，这样太麻烦了，Spring Boot的Profile就给我们提供了解决方案，命令带上参数就搞定。

这里我们来模拟一下，只是简单的修改端口来测试
在Spring Boot中多环境配置文件名需要满足application-{profile}.properties的格式，其中{profile}对应你的环境标识，比如：

* application-dev.properties：开发环境
* application-prod.properties：生产环境

然后在application.properties中加入
```
spring.profiles.active=dev
```
或application.yml中加入
```yaml
spring:
    # 环境 dev|test|pro
    profiles:
        active: dev
```
或启动命令：
```
java -jar xxx.jar --spring.profiles.active=dev
```
参数用--xxx=xxx的形式传递。意思就是表示在application.properties文件中配置了属性。
可以通过SpringApplication.setAddCommandLineProperties(false)禁用命令行配置。

### 附：Appendix A. Common application properties
<a rel="external nofollow" target="_blank" href="https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html">Appendix A. Common application properties</a>

### 参考文章
* 《SpringBoot揭秘 快速构建微服务体系》
* <a rel="external nofollow" target="_blank" href="http://tengj.top/2017/02/28/springboot2/">Spring Boot干货系列：（二）配置文件解析</a>
* <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/clementad/article/details/51970962">spring boot 读取配置文件（application.yml）中的属性值</a>
