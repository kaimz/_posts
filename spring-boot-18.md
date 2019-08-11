---
title: 学习Spring Boot：（十八）session共享
date: 2018-02-27 22:39:03
tags: [Spring Boot]
categories: 学习笔记
---

### 前言
前面我们将 Redis 集成到工程中来了，现在需要用它来做点实事了。这次为了解决分布式系统中的 session 共享的问题，将 session 托管到 Redis。

<!--more-->

### 正文
#### 引入依赖
除了上篇文章中引入 `spring-boot-starter-data-redis`，还需要 `spring-session` 依赖：
```xml
		<dependency>
			<groupId>org.springframework.session</groupId>
			<artifactId>spring-session</artifactId>
		</dependency>
```

#### 配置
在系统的配置文件中加入：
```yaml
spring:
  session:
    store-type: redis
```

并且可以发现 `store-type` 有几种值可以设置，都是可以作为 session 共享的媒介。
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/springboot-session/1.png)

可以看到，Spring Session 支持使用Redis、Mongo、JDBC、Hazelcast来存储Session，

这样就完成了。

#### 测试
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/springboot-session/2.png)



可以看出，session 已经被 shiro 接管了。

**spring-session 实现的思路是**：设计一个Filter，利用 `HttpServletRequestWrapper`，实现自己的 `getSession()方法`，接管创建和管理Session数据的工作。
