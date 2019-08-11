---
title: 事务的特性和@Transactional注解的使用
date: 2017-11-30 16:08:03
tags: [java]
categories: 学习笔记
---

### @Transactional如何工作

实现了EntityManager接口的持久化上下文代理并不是声明式事务管理的唯一部分，事实上包含三个组成部分：
1. 事务的切面
2. 事务管理器
3. EntityManager Proxy本身

<!--more-->

#### 事务切面
事务的切面是一个“around（环绕）”切面，在注解的业务方法前后都可以被调用。实现切面的具体类是``TransactionInterceptor``。

事务的切面的主要职责：
在’before’时，切面提供一个调用点，来决定被调用业务方法应该在正在进行事务的范围内运行，还是开始一个新的独立事务。
在’after’时，切面需要确定事务被提交，回滚或者继续运行。
在’before’时，事务切面自身不包含任何决策逻辑，是否开始新事务的决策委派给事务管理器完成。

* 新的Entity Manager是否应该被创建？
* 是否应该开始新的事务？

**这些需要事务切面’before’逻辑被调用时决定。事务管理器的决策基于以下两点：**
1. 事务是否正在进行；
2. 事务方法的propagation属性（比如REQUIRES_NEW总要开始新事务）。

#### 事务管理器
如果事务管理器确定要创建新事务，那么将：

创建一个新的entity manager
entity manager绑定到当前线程
从数据库连接池中获取连接
将连接绑定到当前线程
使用ThreadLocal变量将entity manager和数据库连接都绑定到当前线程。

事务运行时他们存储在线程中，当它们不再被使用时，事务管理器决定是否将他们清除。

程序的任何部分如果需要当前的entity manager和数据库连接都可以从线程中获取。

#### EntityManager proxy
当业务方法调用entityManager.persist()时，这不是由entity manager直接调用的。
而是业务方法调用代理，代理从线程获取当前的entity manager事务管理器将entity manager绑定到线程。


### spring 中配置JPA事务
在spring 的配置文件中配置jpa的事务：
```xml
<!-- Jpa 事务管理器 -->
	<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
		<property name="entityManagerFactory" ref="entityManagerFactory" />
	</bean>
	<!-- 打开事务注解 -->
	<tx:annotation-driven transaction-manager="transactionManager"/>
```

当然可以使用aop配置事务：
```xml
<!-- Jpa 事务管理器 -->
	<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
		<property name="entityManagerFactory" ref="entityManagerFactory" />
	</bean>
<!-- 声明式事务配置 -->
	<tx:advice id="txAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<tx:method name="get*" propagation="NOT_SUPPORTED" read-only="true" />
			<tx:method name="count*" propagation="NOT_SUPPORTED" read-only="true" />
			<tx:method name="find*" propagation="NOT_SUPPORTED" read-only="true" />
			<tx:method name="query*" propagation="NOT_SUPPORTED" read-only="true" />
			<tx:method name="*" propagation="REQUIRED" rollback-for="Exception" />
		</tx:attributes>
	</tx:advice>
	<!-- 只对业务逻辑层实施事务-->
	<aop:config>
		<aop:pointcut id="txPointcut" expression="execution(* com.devframe.service.impl.*.*(..))" />
		<aop:advisor advice-ref="txAdvice" pointcut-ref="txPointcut"/>
	</aop:config>
```

### spring中事务的几个特性
**补充下，数据库中的事务的四大特性：**
* ``原子性（Atomicity）``：原子性是指事务包含的所有操作要么全部成功，要么全部失败回滚，这和前面两篇博客介绍事务的功能是一样的概念，因此事务的操作如果成功就必须要完全应用到数据库，如果操作失败则不能对数据库有任何影响。
* ``一致性（Consistency）``：一致性是指事务必须使数据库从一个一致性状态变换到另一个一致性状态，也就是说一个事务执行之前和执行之后都必须处于一致性状态。
* ``隔离性（Isolation）``：隔离性是当多个用户并发访问数据库时，比如操作同一张表时，数据库为每一个用户开启的事务，不能被其他事务的操作所干扰，多个并发事务之间要相互隔离。
* ``持久性（Durability）``：持久性是指一个事务一旦被提交了，那么对数据库中的数据的改变就是永久性的，即便是在数据库系统遇到故障的情况下也不会丢失提交事务的操作。

#### 事务隔离级别
隔离级别是指若干个并发的事务之间的隔离程度。``TransactionDefinition`` 接口中定义了五个表示隔离级别的常量：
* ``TransactionDefinition.ISOLATION_DEFAULT``：这是默认值，表示使用底层数据库的默认隔离级别。对大部分数据库而言，通常这值就是TransactionDefinition.ISOLATION_READ_COMMITTED。
* ``TransactionDefinition.ISOLATION_READ_UNCOMMITTED``：该隔离级别表示一个事务可以读取另一个事务修改但还没有提交的数据。该级别不能防止脏读，不可重复读和幻读，因此很少使用该隔离级别。比如PostgreSQL实际上并没有此级别。
* ``TransactionDefinition.ISOLATION_READ_COMMITTED``：该隔离级别表示一个事务只能读取另一个事务已经提交的数据。该级别可以防止脏读，这也是大多数情况下的推荐值。
* ``TransactionDefinition.ISOLATION_REPEATABLE_READ``：该隔离级别表示一个事务在整个过程中可以多次重复执行某个查询，并且每次返回的记录都相同。该级别可以防止脏读和不可重复读。
* ``TransactionDefinition.ISOLATION_SERIALIZABLE``：所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

**补充下数据库中，如果不考虑事务的隔离性，会发生的几种问题：**
* ``脏读``：脏读是指在一个事务处理过程里读取了另一个未提交的事务中的数据。
当一个事务正在多次修改某个数据，而在这个事务中这多次的修改都还未提交，这时一个并发的事务来访问该数据，就会造成两个事务得到的数据不一致。
* ``不可重复读``：不可重复读是指在对于数据库中的某个数据，一个事务范围内多次查询却返回了不同的数据值，这是由于在查询间隔，被另一个事务修改并提交了。
例如事务T1在读取某一数据，而事务T2立马修改了这个数据并且提交事务给数据库，事务T1再次读取该数据就得到了不同的结果，发送了不可重复读。
不可重复读和脏读的区别是，脏读是某一事务读取了另一个事务未提交的脏数据，而不可重复读则是读取了前一事务提交的数据。
在某些情况下，不可重复读并不是问题，比如我们多次查询某个数据当然以最后查询得到的结果为主。但在另一些情况下就有可能发生问题，例如对于同一个数据A和B依次查询就可能不同，A和B就可能打起来了……
* ``虚读(幻读)``：幻读是事务非独立执行时发生的一种现象。例如事务T1对一个表中所有的行的某个数据项做了从“1”修改为“2”的操作，这时事务T2又对这个表中插入了一行数据项，而这个数据项的数值还是为“1”并且提交给数据库。而操作事务T1的用户如果再查看刚刚修改的数据，会发现还有一行没有修改，其实这行是从事务T2中添加的，就好像产生幻觉一样，这就是发生了幻读。幻读和不可重复读都是读取了另一条已经提交的事务（这点就脏读不同），所不同的是不可重复读查询的都是同一个数据项，而幻读针对的是一批数据整体（比如数据的个数）。

#### 事务传播行为
所谓事务的传播行为是指，如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。在``TransactionDefinition``定义中包括了如下几个表示传播行为的常量：
* ``TransactionDefinition.PROPAGATION_REQUIRED``：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。这是默认值。
* ``TransactionDefinition.PROPAGATION_REQUIRES_NEW``：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
* ``TransactionDefinition.PROPAGATION_SUPPORTS``：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
* ``TransactionDefinition.PROPAGATION_NOT_SUPPORTED``：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
* ``TransactionDefinition.PROPAGATION_NEVER``：以非事务方式运行，如果当前存在事务，则抛出异常。
* ``TransactionDefinition.PROPAGATION_MANDATORY``：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
* ``TransactionDefinition.PROPAGATION_NESTED``：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。

选择默认的，即``PROPAGATION_REQUIRED``，事务具有传播机制，多个事务，对于已经存在的事务，下一个事务会加入当前事务。

#### 事务超时
所谓事务超时，就是指一个事务所允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务。在 TransactionDefinition 中以 int 的值来表示超时时间，其单位是秒。

默认设置为底层事务系统的超时值，如果底层数据库事务系统没有设置超时值，那么就是none，没有超时限制。

#### 事务只读属性
只读事务用于客户代码只读但不修改数据的情形，只读事务用于特定情景下的优化，比如使用Hibernate的时候。 默认为读写事务。

**只读事务**并不是一个强制选项，它只是一个“暗示”，提示数据库驱动程序和数据库系统，这个事务并不包含更改数据的操作，那么JDBC驱动程序和数据库就有可能根据这种情况对该事务进行一些特定的优化，比方说不安排相应的数据库锁，以减轻事务对数据库的压力，毕竟事务也是要消耗数据库的资源的。 
但是你非要在“只读事务”里面修改数据，也并非不可以，只不过对于数据一致性的保护不像“读写事务”那样保险而已。 
因此，“只读事务”仅仅是一个性能优化的推荐配置而已，并非强制你要这样做不可。



```
1.我认为单条sql执行的时候是没有事务的，在数据库层面已经保证了该sql的原子性，再加事务只会浪费性能。而且spring的事务是面向切面的，你没有做相关配置，它肯定不会多此一举给你加上事务。
2.在执行多条查询语句时加上事务是为了保证数据前后一致性，只要加上Transactional就能解决这个问题。
3.事务由多个级别，级别越高数据库为了提供相关支持肯定也会占用更多资源，而且数据库也考虑到了事务中只做查询的这个问题，所以它提供了read-only这个状态，相应数据做的支持也会少很多，占用的资源就自然少了。
```

#### spring事务回滚规则
指示spring事务管理器回滚一个事务的推荐方法是在当前事务的上下文内抛出异常。spring事务管理器会捕捉任何未处理的异常，然后依据规则决定是否回滚抛出异常的事务。

默认配置下，spring只有在抛出的异常为运行时unchecked异常时才回滚该事务，也就是抛出的异常为RuntimeException的子类(Errors也会导致事务回滚)，而抛出checked异常则不会导致事务回滚。可以明确的配置在抛出那些异常时回滚事务，包括checked异常。也可以明确定义那些异常抛出时不回滚事务。还可以编程性的通过setRollbackOnly()方法来指示一个事务必须回滚，在调用完setRollbackOnly()后你所能执行的唯一操作就是回滚。

### @Transactional注解
#### 属性

属性 | 类型 | 描述
---|---|---
value | String | 可选的限定描述符，指定使用的事务管理器
propagation | enum: Propagation | 可选的事务传播行为设置
isolation | enum: Isolation | 可选的事务隔离级别设置
readOnly | boolean | 读写或只读事务，默认读写
timeout | int (in seconds granularity) | 事务超时时间设置
rollbackFor | Class对象数组，必须继承自Throwable | 导致事务回滚的异常类数组
rollbackForClassName |	类名数组，必须继承自Throwable |	导致事务回滚的异常类名字数组
noRollbackFor |	Class对象数组，必须继承自Throwable |	不会导致事务回滚的异常类数组
noRollbackForClassName |	类名数组，必须继承自Throwable |	不会导致事务回滚的异常类名字数组

#### 使用方法
 @Transactional 可以作用于接口、接口方法、类以及类方法上。当作用于类上时，该类的所有 public 方法将都具有该类型的事务属性，同时，我们也可以在方法级别使用该标注来覆盖类级别的定义。

虽然 @Transactional 注解可以作用于接口、接口方法、类以及类方法上，但是 Spring 建议不要在接口或者接口方法上使用该注解，因为这只有在使用基于接口的代理时它才会生效。另外， @Transactional 注解应该只被应用到 public 方法上，这是由 Spring AOP 的本质决定的。如果你在 protected、private 或者默认可见性的方法上使用 @Transactional 注解，这将被忽略，也不会抛出任何异常。

默认情况下，只有来自外部的方法调用才会被AOP代理捕获，也就是，类内部方法调用本类内部的其他方法并不会引起事务行为。

只要方法内部抛出``rollbackFor``设置的异常，就会回滚。

例如：
在方法上加上
```java
@Transactional(value="transactionManager", rollbackFor = Exception.class)
```
方法内部只要抛出指定的异常或者错误，就全部回滚。

补充，回滚异常是自己定义的异常类，最好按照要求继承``RuntimeException``。
如果非常有必要在事务中捕捉异常，而且需要回滚事务，那么直接再将这个异常抛出就可以了，但是不建议这么使用。




**参考文章：**
* <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/bao19901210/article/details/41724355">spring事物配置，声明式事务管理和基于@Transactional注解的使用</a>
* <a rel="external nofollow" target="_blank" href="https://www.cnblogs.com/wangyonglong/p/5178450.html">JPA和事务管理</a>
* <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/hy6688_/article/details/44763869">Spring事务传播特性的浅析——事务方法嵌套调用的迷茫</a>
* <a rel="external nofollow" target="_blank" href="https://www.cnblogs.com/fjdingsd/p/5273008.html">数据库事务的四大特性以及事务的隔离级别</a>

