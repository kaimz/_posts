---
title: 学习 Spring Cloud：（一）Eureka  服务发现与注册
date: 2018-04-06 22:11:03
tags: [Spring Cloud]
categories: 学习笔记
---

### Spring Cloud简介

> Spring Cloud是一个基于Spring Boot实现的云应用开发工具，它为基于JVM的云应用开发中涉及的配置管理、服务发现、断路器、智能路由、微代理、控制总线、全局锁、决策竞选、分布式会话和集群状态管理等操作提供了一种简单的开发方式。
>
> Spring Cloud包含了多个子项目（针对分布式系统中涉及的多个不同开源产品），比如：Spring Cloud Config、Spring Cloud Netflix、Spring Cloud0 CloudFoundry、Spring Cloud AWS、Spring Cloud Security、Spring Cloud Commons、Spring Cloud Zookeeper、Spring Cloud CLI等项目。



<!--more-->

### 服务治理

我使用的是Spring Cloud Eureka来实现服务治理。使用版本都是2.0版本：

- Spring Boot 2.0
- Spring Cloud Finchley.M9

> Spring Cloud Eureka是Spring Cloud Netflix项目下的服务治理模块。而Spring Cloud Netflix项目是Spring Cloud的子项目之一，主要内容是对Netflix公司一系列开源产品的包装，它为Spring Boot应用提供了自配置的Netflix OSS整合。通过一些简单的注解，开发者就可以快速的在应用中配置一下常用模块并构建庞大的分布式系统。它主要提供的模块包括：服务发现（Eureka），断路器（Hystrix），智能路由（Zuul），客户端负载均衡（Ribbon）等。

#### 创建服务注册中心

1. 使用 IDEA 创建一个 `Spring initializr` 工程，勾选 `Cloud Discovery`下面的 `Eureka Server`，直接创建应用，我是用 maven 构建的，加入的依赖如下：

   ```xml
   <dependency>
   	<groupId>org.springframework.cloud</groupId>
   	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
   </dependency>
   ```

   需要注意的是，相比较 1.X 版本中，netflix 公司的开源项目，都加了个前缀。


1. 在程序启动入口加上注解 `@EnableEurekaServer` ，通过这样的注解配置，提供一个服务注册中心给其他服务进行对话。

2. 在配置文件中加入：

   ```yaml
   # 默认情况下 eureka 会将自己注册一个客户端，服务端不需要，所以我们将它关闭。
   eureka:
     client:
       fetch-registry: false
       register-with-eureka: false # 不注册自己本身
   ```

3. 我们尝试访问这个应用，我设置的端口是 `1001`，所以访问 `http://localhost:1001`，访问到 Eureka 的管理注册服务的页面，可以查看各个服务的状态。

#### 创建服务提供者

1. 创建一个服务提供者工程 `eureka-producer`，现在我们加入的是客户端的依赖就可以了：

   ```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
   </dependency>
   ```

2. 然后再程序启动类加上注解 `@EnableDiscoveryClient` ，该注解能激活Eureka中的DiscoveryClient实现，这样才能实现Controller中对服务信息的输出。

3. 配置文件中加上需要注册到的服务中心的地址（指定应用程序的名称，后期进行服务路由和负载均衡时候方便调用）：

   ```yaml
   spring:
     application:
       name: eureka-producer
   eureka:
     client:
       service-url:
         defaultZone: http://k.wuwii.com:1001/eureka/
   ```

4. 启动应用，
   
   ```
   2018-04-04 23:53:47.436  INFO 1364 --- [  restartedMain] com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1522857227434 with initial instances count: 4
   2018-04-04 23:53:47.440  INFO 1364 --- [  restartedMain] o.s.c.n.e.s.EurekaServiceRegistry        : Registering application eureka-producer with eureka with status UP
   2018-04-04 23:53:47.441  INFO 1364 --- [  restartedMain] com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1522857227440, current=UP, previous=STARTING]
   2018-04-04 23:53:47.442  INFO 1364 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_EUREKA-PRODUCER/localhost:eureka-producer:1002: registering service...
   2018-04-04 23:53:47.494  INFO 1364 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_EUREKA-PRODUCER/localhost:eureka-producer:1002 - registration status: 204
   ```

   上面显示我的注册服务有四个实例，我之前已经注册过了，现在服务发现 `localhost:eureka-producer:1002`，注册到服务中心成功返回状态吗204。

   我们再进入到`http://localhost:1001`，可以查看到 `eureka-producer` 已经注册到我们的服务中心，

   ![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/spring-cloud/eureka-producer-1.png)

#### 复杂配置
   ```yaml
      client:
		#healthcheck:
		  #enabled: true
		registry-fetch-interval-seconds: 10000 #间隔多久去拉取服务注册信息，默认为30秒
		service-url:
		  defaultZone: http://localhost:9000/eureka/
	  instance:
		prefer-ip-address: true
		ip-address: 127.0.0.1
		instance-id: ${spring.application.name}-${eureka.instance.ip-address}-${server.port}-${random.value}
		lease-renewal-interval-in-seconds: 10000   #租期更新时间间隔（默认30秒）
		lease-expiration-duration-in-seconds: 30000 # 租期到期时间（默认90秒）
		#需要增加下面配置，告诉注册中心访问路径变化
		status-page-url-path: /actuator/info
		health-check-url-path: /actuator/health
		home-page-url-path: /
   ```



[代码](https://github.com/kaimz/learn-spring-cloud/tree/master/eureka)
