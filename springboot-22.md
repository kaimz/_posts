---
title: 学习Spring Boot：（二十二）使用 AOP
date: 2018-03-05 22:08:08
tags: [Spring Boot]
categories: 学习笔记
---

### 前言
>AOP [^1]，意为：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。基于AOP实现的功能不会破坏原来程序逻辑，因此它可以很好的对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。

AOP 专门用于处理系统中分布于各个模块（不同方法）中的交叉关注点的问题，在 Java EE 应用中，常常通过 AOP 来处理一些具有横切性质的系统级服务，如事务管理、安全检查、缓存、对象池管理等，AOP 已经成为一种非常常用的解决方案。

<!--more-->
[^1]: Aspect Oriented Programming的缩写

### 正文
#### Spring Boot 中使用
1. 在 `pom.xml` 中加入 aop 依赖：
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
```
2. 当我们需要在非接口上面进行切面操作的时候，就需要 `CGLIB`来实现 AOP，在系统配置文件中加入设置：
```yaml
spring: 
  aop:
    proxy-target-class: true
```
默认为 `false`。
#### 切点表达式
列出常用的几个表达式：
1. `execution()` 满足execution中描述的方法签名。  `execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern)throws-pattern?)`  
* `modifier-pattern`：表示方法的修饰符;
* `ret-type-pattern`：表示方法的返回值
* `declaring-type-pattern`：表示方法所在的类的路径
* `name-pattern`：表示方法名
* `param-pattern`：表示方法的参数
* `throws-pattern`：表示方法抛出的异常
* 其中后面跟着`?`的是可选项。

2. `this()`是用来限定方法所属的类，为接口则限定所有的实现类，为类的话，限定这单个类。
3. `@annotation`表示具有某个标注的方法。
4. `args` 表示方法的参数属于一个特定的类，`@args` 表示参数有特定的标注注解。
5. `within` 包或者类型满足within中描述的包或者类型的类的所有非私有方法，`@within` 类型拥有@target描述中给出的annotation,其中@target和@within的区别在于@within要求的annotation的级别为CLASS，而@target为RUNTIME
    . `target` 业务实例对象（非代理实例）的类型满足target 中的描述的类型，@target	类型拥有@target描述中给出的annotation
6. `bean()` 表示所有匹配的 bean，例如 ,bean("*Service")，匹配所有 `Service` 结尾的类。可以使用 `!bean()` 表示不匹配。

**注意事项:**
* 在各个pattern中，可以使用"*"来表示匹配所有。
* 在`param-pattern`中，可以指定具体的参数类型，多个参数间用`,`隔开，各个也可以用 `*` 来表示匹配任意类型的参数，如`(String)`表示匹配一个`String`参数的方法；`(*,String)`表示匹配有两个参数的方法，第一个参数可以是任意类型，而第二个参数是String类型。
* 可以用`(..)`表示零个或多个任意的方法参数。
* 使用`&&`符号表示与关系，使用`||`表示或关系、使用`!`表示非关系。在XML文件中使用`and`、`or`和`not`这三个符号。

AspectJ提供了五种定义通知的标注：

* `@Before`：前置通知，在调用目标方法之前执行通知定义的任务
* `@After`：后置通知，在目标方法执行结束后，无论执行结果如何都执行通知定义的任务
* `@AfterReturning`：后置通知，在目标方法执行结束后，如果执行成功，则执行通知定义的任务
* `@AfterThrowing`：异常通知，如果目标方法执行过程中抛出异常，则执行通知定义的任务
* `@Around`：环绕通知，在目标方法执行前和执行后，都需要执行通知定义的任务

**通过标注定义通知只需要两个步骤：**
1. 将以上五种标注之一添加到切面的方法中
2. 在标注中设置切点的定义。

#### 创建环绕通知
环绕通知相比其它四种通知有其特殊之处。环绕通知本质上是将前置通知、后置通知和异常通知整合成一个单独的通知。

用`@Around`标注的方法，该方法必须有一个`ProceedingJoinPoint`类型的参数，

在方法体中，需要通过这个参数，以`joinPoint.proceed();`的形式调用目标方法。注意在环绕通知中必须进行该调用，否则目标方法本身的执行就会被跳过。

计算方法的执行时间：
```
    @Around("logPointCut()") //切点
    public Object around(ProceedingJoinPoint point) throws Throwable {
        long beginTime = System.currentTimeMillis();
        //执行方法
        Object result = point.proceed();
        //执行时长(毫秒)
        long time = System.currentTimeMillis() - beginTime;

        return result;
    }
```
#### 处理通知中参数
获取参数的方式则需要使用关键词是`args`。
```java
    @Pointcut("bean(sysUserServiceImpl) && args(userEntity,..)")
    public void userPointCut(SysUserEntity userEntity) {

    }

    @Before("userPointCut(userEntity)")
    public void validateUser(SysUserEntity userEntity) {
        // to handler args
    }
```
这里有个非常严格的一点就是，`args(userEntity,..)`，表示目标方法，可能有多个参数，但是包括 `userEntity`，这里 `userEntity` 必须参数名相同，不同就编织了。

**`args()`中参数的名称必须跟切点方法的签名中`public void validateUser(SysUserEntity userEntity)`的参数名称相同。如果使用切点函数定义，其中的参数名称也必须与通知方法签名中的参数完全相同**
#### AfterReturning增强处理
```java
    @AfterReturning(pointcut = "logPointCut()", returning = "rtv")
    public void logAfter(Object rtv) {
        System.out.println(Objects.toString(rtv));
    }
```
使用 `@AfterReturning` 注解时，指定了一个`returning`属性，假设该属性值为`rvt`，这表明允许在Advice方法（logAfter()方法）中定义名为rvt的形参，程序可通过rvt形参来访问目标方法的返回值。

注意：
**虽然`AfterReturning`增强处理可以访问到方法的返回值，但它不可以改变目标方法的返回值。**
#### AOP切面的优先级
有时候，我们对一个方法会有多个切面的问题，这个时候还会涉及到切面的执行顺序的问题。

我们可以定义每个切面的优先级， Spring 中提供注解 `@Order(i)` ，当 `i` 的值越小，优先级越高。

### 总结

Spring AOP 属于动态代理。

Spring 的 AOP 代理由 Spring 的 IoC 容器负责生成、管理，其依赖关系也由 IoC 容器负责管理。因此，AOP 代理可以直接使用容器中的其他 Bean 实例作为目标，这种关系可由 IoC 容器的依赖注入提供。

总得来说使用 AOP 的时候，需要做好三件事情，

1. 由于 AOP 一般作用于业务代码上，首先写好业务代码；
2. AOP 使用阶段，定义切入点，一个切入点可以横切多个业务组件；
3. 切入点增强处理，为业务点织入增强处理的动作。

### 参考文章
* [Spring AOP中定义切点（PointCut）和通知（Advice）](https://www.tianmaying.com/tutorial/spring-aop-point-advice)
* [Spring Boot中使用AOP统一处理Web请求日志](http://blog.didispace.com/springbootaoplog/)
