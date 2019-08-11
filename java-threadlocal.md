---
title: Java并发编程中ThreadLocal
date: 2017-11-25 1:08:03
tags: [java,并发编程]
categories: 学习笔记
---

---

ThreadLocal，很多地方叫做线程本地变量，也有些地方叫做线程本地存储。ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。

但是要注意，虽然ThreadLocal能够解决上面说的问题，但是由于在每个线程中都创建了副本，所以要考虑它对资源的消耗，比如内存的占用会比不使用ThreadLocal要大。
<!--more-->
### 实现原理
#### 拥有方法
![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwxoarlbuj20f60g5ab2.jpg)
下面看下几个怎么设计实现ThreadLocal的方法：
#### get
```java
/**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
1. 获取当前线程；
2. 然后通过getMap 获取Map；
3. 获取到Map的键值对；
4. 传入``this`` 当前ThreadLocal获取当前的键值对；
5. 根据获取到的entry 返回值，为null 的话调用``setInitialValue``方法；
#### getMap
```java
/**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
返回线程中的threadLocals变量，继续看threadLocals的实现；
#### threadLocals
```java
/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```
它是ThreadLocal中的静态内部类ThreadLocalMap：
#### ThreadLocalMap
##### Entry
```java
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```
ThreadLocalMap的Entry继承了WeakReference，并且使用ThreadLocal作为键值。
#### setInitialValue
```java
/**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```
如果map不为空，就设置键值对，为空，再创建Map，看一下createMap的实现;
#### createMap
```java
/**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
这样为当前线程创建副本变量就完毕了。
#### 怎么创建副本变量

首先，在每个线程Thread内部有一个ThreadLocal.ThreadLocalMap类型的成员变量threadLocals，这个threadLocals就是用来存储实际的变量副本的，键值为当前ThreadLocal变量，value为变量副本（即T类型的变量）。

初始时，在Thread里面，threadLocals为空，当通过ThreadLocal变量调用get()方法或者set()方法，就会对Thread类中的threadLocals进行初始化，并且以当前ThreadLocal变量为键值，以ThreadLocal要保存的副本变量为value，存到threadLocals。

然后在当前线程里面，如果要使用副本变量，就可以通过get方法在threadLocals里面查找。
### 使用场景
1. 各种连接池获取连接（如，数据库连接，redis连接）；
2. session管理。

### 学习代码
```java
package com.wuwii.test;

import org.junit.Test;

/**
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/24 20:32</pre>
 */
public class TestThreadLocal {

    /**
     * 声明一个ThreadLocal变量
     */
    private ThreadLocal<String> local1 = new ThreadLocal<>();

    /**
     * 为ThreadLocal赋值
     */
    private void setValue() {
        local1.set(Thread.currentThread().getName());
    }


    @Test
    public void test1() {
         Thread thread1 = new Thread(() -> {
             /*
              * 每次调用get方法前，必须要set，不然会抛出NPE
              */
            setValue();
            System.out.printf("线程一的localValue为: %s%n", local1.get());
        });
        new Thread(() -> {
            //先给线程二的threadLocal赋值，然后运行线程一，最后打印线程二的threadLocal
            setValue();
            try {
                /*
                 * 在线程二中添加运行线程一，证明了每个线程保存的ThreadLocal的副本变量是不同的
                 */
                thread1.start();
                thread1.join();
                // 运行完线程一，再输出线程二
                System.out.printf("线程二的localValue为: %s%n", local1.get());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}

```

输出结果：
```
线程一的localValue为: Thread-0
线程二的localValue为: Thread-1
```
可以看出，线程二并没有被影响。

**参考博客：<a rel="external nofollow" target="_blank" href="http://www.cnblogs.com/dolphin0520/p/3920407.html">Java并发编程：深入剖析ThreadLocal</a>**
