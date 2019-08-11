---
title: Java中使用有返回值的线程
date: 2017-11-1 10:18:03
tags: [java,并发编程]
categories: 学习笔记
---

在创建多线程程序的时候，我们常实现Runnable接口，Runnable没有返回值，要想获得返回值，Java5提供了一个新的接口Callable，可以获取线程中的返回值，但是获取线程的返回值的时候，需要注意，我们的方法是异步的，获取返回值的时候，线程任务不一定有返回值，所以，需要判断线程是否结束，才能够去取值。

<!--more-->

### 测试代码

```java
package com.wuwii.test;

import java.util.concurrent.*;

/**
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/10/31 11:17</pre>
 */
public class Test {

    private static final Integer SLEEP_MILLS = 3000;

    private static final Integer RUN_SLEEP_MILLS = 1000;

    private int afterSeconds = SLEEP_MILLS / RUN_SLEEP_MILLS;

    // 线程池（根据机器的核心数）
    private final ExecutorService fixedThreadPool = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    private void testCallable() throws InterruptedException {
        Future<String> future = null;
        try {
            /**
             * 在创建多线程程序的时候，我们常实现Runnable接口，Runnable没有返回值，要想获得返回值，Java5提供了一个新的接口Callable
             *
             * Callable需要实现的是call()方法，而不是run()方法，返回值的类型有Callable的类型参数指定，
             * Callable只能由ExecutorService.submit() 执行，正常结束后将返回一个future对象。
             */
            future = fixedThreadPool.submit(() -> {
                Thread.sleep(SLEEP_MILLS);
                return "The thread returns value.";
            });
        } catch (Exception e) {
            e.printStackTrace();
        }

        if (future == null) return;

        for (;;) {
            /**
             * 获得future对象之前可以使用isDone()方法检测future是否完成，完成后可以调用get()方法获得future的值，
             * 如果直接调用get()方法，get()方法将阻塞到线程结束，很浪费。
             */
            if (future.isDone()) {
                try {
                    System.out.println(future.get());
                    break;
                } catch (InterruptedException | ExecutionException e) {
                    e.printStackTrace();
                }
            } else {
                System.out.println("After " + afterSeconds-- + " seconds,get the future returns value.");
                Thread.sleep(1000);
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        new Test().testCallable();
    }
}


```
运行结果：
```
After 3 seconds,get the future returns value.
After 2 seconds,get the future returns value.
After 1 seconds,get the future returns value.
The thread returns value.
```



### 总结:
1. 需要返回值的线程使用Callable 接口，实现call 方法；
2. 获得future对象之前可以使用isDone()方法检测future是否完成，完成后可以调用get()方法获得future的值，如果直接调用get()方法，get()方法将阻塞到线程结束。
