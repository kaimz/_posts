---
title: 学习Spring Boot：（十一） 自定义装配参数
date: 2018-02-23 10:08:12
tags: [Spring Boot]
categories: 学习笔记
---

### 前言
SpringMVC 中 `Controller` 中方法的参数非常灵活，得益于它的强大自动装配，这次将根据上次遗留下的问题，将研究下装配参数。

<!--more-->

### 正文

SpringMVC中使用了两个接口来处理参数：
* `HandlerMethodArgumentResolver` 处理方法请求参数；
* `HandlerMethodReturnValueHandler` 处理方法的返回参数。

这里我们将使用 `HandlerMethodArgumentResolver` 来处理方法上的参数。





未完成
大家可以参考[使用HandlerMethodArgumentResolver接口自定义Spring MVC的参数接受规则](https://blog.csdn.net/he90227/article/details/51537273)

