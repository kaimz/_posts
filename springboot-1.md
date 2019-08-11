---
title: 学习Spring Boot：（一）入门
date: 2018-2-23 10:08:01
tags: [Spring Boot]
categories: 学习笔记
---

## 微服务
现在微服务越来越火了，Spring Boot热度蹭蹭直升，自学下。
>微服务其实是服务化思路的一种最佳实践方向，遵循SOA（面向服务的架构）的思路，各个企业在服务化治理上面的道路已经走得很远了，整个软件交付链上各个环节的基础设施逐渐成熟了，微服务就诞生了。

>微服务给我们也带来了很多挑战，服务“微”化之后，一个显著的特征就是服务的数量增多了。如果将软件开发和交付也作为一种生产模式的看待，那么数量众多的微服务实际上就类似于传统生产线上的产品，而在传统生产模式下，为了能够高效地生产大量产品，通常采用的就是标准化生产。

<!--more-->

## 学习
>Spring Boot只是简化了spring 全家桶的配置，它使用“习惯优于配置”（Convention Over Configuration 项目中存在大量的配置，此外还内置了一个习惯性的配置，让你无需手动进行配置）的理念让你的项目快速运行起来。使用Spring Boot很容易创建一个独立运行（运行jar,内嵌Servlet容器）、准生产级别的基于Spring框架的项目，使用Spring Boot你可以不用或者只需要很少的Spring配置。

## 核心
* 自动配置：针对很多Spring应用程序常见的应用功能，Spring Boot能自动提供相关配置。
* 起步依赖：告诉Spring Boot需要什么功能，它就能引入需要的库。
* 命令行界面：这是Spring Boot的可选特性，借此你只需写代码就能完成完整的应用程序，无需传统项目构建。
* Actuator：让你能够深入运行中的Spring Boot应用程序，一探究竟。

## 入门：搭建一个Spring Boot Web
### 初始化
我是使用的IDEA，它已经集成了Spring Boot。
选择file - 新建一个项目，选择Spring Initializr
注意我选择的jdk  是1.8 ，推荐使用1.8 ，好像低版本的1.5 、1.6有限制，还有就是现在最新版本1.5.9的Spring Boot还不支持jdk9。 
![](https://ws1.sinaimg.cn/large/ece1c17dgy1fniu4p558oj20vr0p13zo.jpg)

next -》 next
选择Spring Boot 版本，选择需要的模块，我们开始学习就使用默认的Web模块。
![](https://ws1.sinaimg.cn/large/ece1c17dgy1fniu5731p7j20vr0p1t9v.jpg)

### 结构

初始化完成后，会生成几个文件，项目结构：
![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fnnhm0jydbj20da0cv0t6.jpg)

* pom.xml：Maven构建说明文件。
* *Application：带有main()方法的类，用于启动应用程序。
* *ApplicationTests：一个空的Junit测试类，它加载了一个使用Spring Boot字典配置功能的Spring应用程序上下文。
* application.properties：一个空的properties文件，用于配置项目的相关属性。
* static存放相关静态文件；
* template 存放模板渲染文件。

#### pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.wuwii</groupId>
	<artifactId>learn-spring-boot</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>learn-spring-boot</name>
	<description>Demo project for Spring Boot</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>


</project>

```

##### 使用的是Spring Boot父级依赖
，spring-boot-starter-parent是一个特殊的starter,它用来提供相关的Maven默认依赖，使用它之后，常用的包依赖可以省去version标签。

##### 起步依赖 spring-boot-starter-xx
Spring Boot提供了很多”开箱即用“的依赖模块，都是以spring-boot-starter-xx作为命名的。举个例子来说明一下这个起步依赖的好处，比如组装台式机和品牌机，自己组装的话需要自己去选择不同的零件，最后还要组装起来，期间有可能会遇到零件不匹配的问题。耗时又消力，而品牌机就好一点，买来就能直接用的，后续想换零件也是可以的。相比较之下，后者带来的效果更好点（这里就不讨论价格问题哈），起步依赖就像这里的品牌机，自动给你封装好了你想要实现的功能的依赖。就比如我们之前要实现web功能，引入了spring-boot-starter-web这个起步依赖。

起步依赖本质上是一个Maven项目对象模型（Project Object Model，POM），定义了对其他库的传递依赖，这些东西加在一起即支持某项功能。很多起步依赖的命名都暗示了它们提供的某种或者某类功能。

##### Spring Boot Maven插件
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

* 把项目打包成一个可执行的超级JAR（uber-JAR）,包括把应用程序的所有依赖打入JAR文件内，并为JAR添加一个描述文件，其中的内容能让你用java -jar来运行应用程序。
* 搜索public static void main()方法来标记为可运行类。

### 运行
现在添加一个接口，来启动项目运行：
```java
package com.wuwii.learnspringboot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@SpringBootApplication
public class LearnSpringBootApplication {

	@RequestMapping("/")
	public String index() {
		return "Hello World";
	}
	public static void main(String[] args) {
		SpringApplication.run(LearnSpringBootApplication.class, args);
	}
}

```
#### 启动方式
 * @SpringBootApplication是Sprnig Boot项目的核心注解，主要目的是开启自动配置。后续讲解原理的时候再深入介绍。main方法这是一个标准的Java应用的main的方法，主要作用是作为项目启动的入口，直接运行它的main() 函数。
 * 使用命令 mvn spring-boot:run”在命令行启动该应用，IDEA中该命令在如下位置：
    ![](https://ws1.sinaimg.cn/large/ece1c17dgy1fniutggip4j20h50gcwg2.jpg)
* 运行“mvn package”进行打包时，会打包成一个可以直接运行的 JAR 文件，使用“java -jar”命令就可以直接运行。
  ![](https://ws1.sinaimg.cn/large/ece1c17dgy1fnivaegtkuj20fn0dxwev.jpg)

![](https://ws1.sinaimg.cn/large/ece1c17dgy1fnivck4591j20kp0b5my6.jpg)

## 总结
* 了解Spring Boot 的基本结构和相关属性的概念；
* 启动和运行方式。
