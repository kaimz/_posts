---
title: Java中创建线程池的常用方法
date: 2017-11-1 17:08:03
tags: [java]
categories: 学习笔记
---

### 创建线程池
学习了Java中线程池的工作流程，现在学习一下怎么使用线程池；前面了解到构造一个线程池参数，最简单的线程池构造函数：
```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```
最少需要设置这么几个参数：
```java
corePoolSize 核心池大小，
maximumPoolSize 最大线程数量，
keepAliveTime 心跳时间
unit 心跳时间单位，什么时候销毁多余的线程
workQueue 最重要的，阻塞队列，存储等待中的任务
```

<!--more-->

在前面创建过线程池：
```java
private static final BlockingQueue queue = new ArrayBlockingQueue(5);

private static final ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS,
            queue);
```
第一步创建一个固定容量的队列来存储等待执行的任务；  
第二步设置核心池数，最大容量数，心跳时间参数。

这个executor线程池说明了，核心池数为5，缓存队列最多存储5个任务，最大线程池数为10，当任务数量大于核心数（5）的时候，监控空闲线程，在心跳时间200 MILLISECONDS后，结束任务，直到线程池中线程数不大于核心数 5。

### 使用Executors来创建线程池
~~如果没有特殊的要求，一般都是推荐用Executors工具类来创建线程池，因为它的参数都给我们配置好了，直接拿来用就好。~~
``Executors``类提供的方法来创建线程池：
```java
Executors.newCachedThreadPool(); //创建一个缓冲池，缓冲池容量大小为Integer.MAX_VALUE
Executors.newSingleThreadExecutor(); //创建容量为1的缓冲池
Executors.newFixedThreadPool(int corePoolSize); //创建固定容量大小的缓冲池，缓存队列大小为Integer.MAX_VALUE
Executors.newScheduledThreadPool(int corePoolSize) //创建一个最大容量为Integer.MAX_VALUE的缓冲池，支持定时及周期性任务执行
Executors.newSingleThreadScheduledExcutor //创建一个单例线程池，定期或延时执行任务。
Executors.newWorkStealingPool //创建持有足够线程的线程池来支持给定的并行级别，并通过使用多个队列，减少竞争，它需要穿一个并行级别的参数，如果不传，则被设定为默认的CPU数量。
```

#### newCachedThreadPool
newCachedThreadPool 创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。缓冲池容量大小为Integer.MAX_VALUE。
```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```


#### newSingleThreadExecutor
创建容量为1的缓冲池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。
```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
#### newFixedThreadPool
创建固定容量大小的缓冲池，缓存队列大小为Integer.MAX_VALUE:
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
定长线程池的大小最好根据系统资源进行设置。如``Runtime.getRuntime().availableProcessors()``。
#### newScheduledThreadPool
创建一个最大容量为Integer.MAX_VALUE的缓冲池，支持定时及周期性任务执行。
```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```
这里主要主要它的定时任务用法；

```java
package com.wuwii.test;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

/**
 * 测试newScheduledThreadPool
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/1 16:25</pre>
 */
public class TestPoolTwo {

    public static void main(String[] args) {
        ScheduledExecutorService scheduleExcutor = Executors.newScheduledThreadPool(5);
        //延迟两秒执行
        scheduleExcutor.schedule(() -> {
            System.out.println("Delay 2 seconds.");
        }, 2, TimeUnit.SECONDS);

        //延迟两秒执行，后面每隔五秒执行
        scheduleExcutor.scheduleAtFixedRate(() -> {
            System.out.println("Delay 2 seconds.");
        }, 2, 5, TimeUnit.SECONDS);
    }
}

```
主要注意的有两点：
1. 使用的是``ScheduledExecutorService`` 这个接口，这个接口也是继承``ExecutorService``，所以也有sumit，execute方法；
2. ``ScheduledExecutorService``接口中有定时，延迟执行任务的方法:``scheduleAtFixedRate``,``schedule``。

#### newSingleThreadScheduledExcutor
创建一个单例线程池，定期或延时执行任务，方法同同上面的``newScheduledThreadPool``：

```java
public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1));
    }
```
#### newWorkStealingPool
创建一个拥有多个任务队列（以便减少连接数）的线程池：
```java
public static ExecutorService newWorkStealingPool() {
        return new ForkJoinPool
            (Runtime.getRuntime().availableProcessors(),
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }

public static ExecutorService newWorkStealingPool(int parallelism) {
        return new ForkJoinPool
            (parallelism,
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }
```
默认不传入线程池大小，默认按机器CPU能力来设置。

它使用的是``ForkJoinPool``多线程中的任务分解机制，将大任务按照预先制定的规则将大任务分解成小任务，多线程并发。这个是java7新加入的线程池，可以使用相对少的线程来处理大量的任务。

### 阿里代码规范补充
编码的时候发现了最新的阿里代码规范工具中，发现了这个提示了，记录，

线程池不允许使用Executors去创建，而是通过ThreadPoolExecutor的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。 说明：Executors各个方法的弊端：
1. newFixedThreadPool和newSingleThreadExecutor:
    主要问题是堆积的请求处理队列可能会耗费非常大的内存，甚至OOM。
2. newCachedThreadPool和newScheduledThreadPool:
    主要问题是线程数最大数是Integer.MAX_VALUE，可能会创建数量非常多的线程，甚至OOM。
3. 创建线程或线程池时请指定有意义的线程名称，方便出错时回溯。创建线程池的时候请使用带ThreadFactory的构造函数，并且提供自定义ThreadFactory实现或者使用第三方实现。         
    Positive example 1：
```java
    //org.apache.commons.lang3.concurrent.BasicThreadFactory
    ScheduledExecutorService executorService = new ScheduledThreadPoolExecutor(1,
        new BasicThreadFactory.Builder().namingPattern("example-schedule-pool-%d").daemon(true).build());
```


Positive example 2：
```java
    ThreadFactory namedThreadFactory = new ThreadFactoryBuilder()
        .setNameFormat("demo-pool-%d").build();

    //Common Thread Pool
    ExecutorService pool = new ThreadPoolExecutor(5, 200,
         0L, TimeUnit.MILLISECONDS,
         new LinkedBlockingQueue<Runnable>(1024), namedThreadFactory, new ThreadPoolExecutor.AbortPolicy());

    pool.execute(()-> System.out.println(Thread.currentThread().getName()));
    pool.shutdown();//gracefully shutdown
```


Positive example 3：
```java
    <bean id="userThreadPool"
        class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
        <property name="corePoolSize" value="10" />
        <property name="maxPoolSize" value="100" />
        <property name="queueCapacity" value="2000" />

    <property name="threadFactory" value= threadFactory />
        <property name="rejectedExecutionHandler">
            <ref local="rejectedExecutionHandler" />
        </property>
    </bean>
    //in code
    userThreadPool.execute(thread);
```

