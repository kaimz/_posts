---
title: 学习Spring Boot：（九）统一异常处理
date: 2018-02-23 10:08:10
tags: [Spring Boot]
categories: 学习笔记
---

### 前言
开发的时候，每个controller的接口都需要进行捕捉异常的处理，以前有的是用切面做的，但是SpringMVC中就自带了`@ControllerAdvice` ，用来定义统一异常处理类，在 SpringBoot 中额外增加了 `@RestControllerAdvice`。

<!--more-->

### 使用
#### 创建全局异常处理类
通过使用 `@ControllerAdvice` 或者 `@RestControllerAdvice` 定义统一的异常处理类。

在方法的注解上加上 `@ExceptionHandler` 用来指定这个方法用来处理哪种异常类型，然后处理完异常，将相关的结果返回。

```java
@RestControllerAdvice
public class ExceptionHandler {
    /**
     * logger
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(ExceptionHandler.class);

    /**
     * 处理系统自定义的异常
     *
     * @param e 异常
     * @return 状态码和错误信息
     */
    @org.springframework.web.bind.annotation.ExceptionHandler(KCException.class)
    public ResponseEntity<String> handleKCException(KCException e) {
        LOGGER.error(e.getMessage(), e);
        return ResponseEntity.status(e.getCode()).body(e.getMessage());
    }

    @org.springframework.web.bind.annotation.ExceptionHandler(DuplicateKeyException.class)
    public ResponseEntity<String> handleDuplicateKeyException(DuplicateKeyException e) {
        LOGGER.error(e.getMessage(), e);
        return ResponseEntity.status(HttpStatus.CONFLICT).body("数据库中已存在该记录");
    }

    @org.springframework.web.bind.annotation.ExceptionHandler(AuthorizationException.class)
    public ResponseEntity<String> handleAuthorizationException(AuthorizationException e) {
        LOGGER.error(e.getMessage(), e);
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body("没有权限，请联系管理员授权");
    }

    /**
     * 处理异常
     *
     * @param e 异常
     * @return 状态码
     */
    @org.springframework.web.bind.annotation.ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleException(Exception e) {
        LOGGER.error(e.getMessage(), e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
    }
}


```

#### 测试
我在 controller 写了一个新增的方法，由于我的数据库中设置了 username 字段**唯一索引**，所以相同的值添加第二次的时候，肯定会抛出上面方法中的第二个异常 `DuplicateKeyException` ：
```java
@PostMapping()
    public ResponseEntity insert(@RequestBody SysUserEntity user) {
        sysUserService.save(user);
        return ResponseEntity.status(CREATED).build();
    }
```
第一次新增的时候：
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/contoller-advice/1.png)

第二次新增的时候返回异常信息：
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/contoller-advice/2.png)
