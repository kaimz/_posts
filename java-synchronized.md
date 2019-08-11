---
title: Java多线程中synchronized
date: 2017-11-5 10:08:03
tags: [java,并发编程]
categories: 学习笔记
---

在多线程中，当多个线程同时访问同一个资源对象的时候，由于线程在处理中是不可控的，导致，执行的结果可能出现不可控的错误。

例如：两个线程thread-1和thread-2，同时要数据入库，需要判断数据字段a，不重复，所以当插入数据的时候先去检查数据库中a字段，当我们的两个线程中字段a相同的时候，出现thread1先执行查询，在thread2查询，两个线程同时都会得到a字段没重复，这个时候，数据入库，肯定会有问题的。

有线程安全的问题，这个资源叫做``临界资源``。

<!--more-->

当多个线程同时访问临界资源（一个对象，对象中的属性，一个文件，一个数据库等）时，就可能会产生线程安全问题。

解决办法有两个，一个是让线程同步synchronized， 一个是lock。

#### synchronized关键字
使用 ``synchronized``关键字来修饰一个方法和方法块，当线程访问这个对象的synchronized修饰的方法的时候，会锁住这个方法，其他线程无法访问，等待这个线程执行完毕，其他线程才排队进来依次执行，
```java
public class TestThread {
    public static void main(String[] args) {
        ThreadData threadData = new ThreadData();
        ThreadData threadData1 = new ThreadData();
        new Thread(() -> threadData.data1()).start();
        new Thread(() -> ThreadData.data2()).start();
        new Thread(() -> threadData.data3()).start();
        new Thread(() -> threadData1.data1()).start();
    }
}

class ThreadData {
    public synchronized void data1() {
        System.out.println("begin data1");
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("end data1");
    }

    public synchronized static void data2() {
        System.out.println("data2");
    }

    public synchronized void data3() {
        System.out.println("begin data3");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("end data3");
    }
}
```
打印结果
```
begin data1
data2
begin data1
end data1
begin data3
end data1
end data3
```

首先需要理解线程安全的两个方面：**执行控制**和**内存可见**。

**执行控制**的目的是控制代码执行（顺序）及是否可以并发执行。

**内存可见**控制的是线程执行结果在内存中对其它线程的可见性。根据[Java内存模型](http://blog.csdn.net/suifeng3051/article/details/52611310)的实现，线程在具体执行时，会先拷贝主存数据到线程本地（CPU缓存），操作完成后再把结果从线程本地刷到主存。

`synchronized`关键字解决的是执行控制的问题，它会阻止其它线程获取当前对象的监控锁，这样就使得当前对象中被`synchronized`关键字保护的代码块无法被其它线程访问，也就无法并发执行。更重要的是，`synchronized`还会创建一个**内存屏障**，内存屏障指令保证了所有CPU操作结果都会直接刷到主存中，从而保证了操作的内存可见性，同时也使得先获得这个锁的线程的所有操作，都**happens-before**于随后获得这个锁的线程的操作。

#### 总结

1. 当一个线程正在访问一个对象的synchronized方法，那么其他线程不能访问该对象的其他synchronized方法。
2. 如果一个线程A需要访问对象object1的synchronized方法fun1，另外一个线程B需要访问对象object2的synchronized方法fun1，即使object1和object2是同一类型），也不会产生线程安全问题，因为他们访问的是不同的对象，所以不存在互斥问题。
3. 如果一个线程执行一个对象的非static synchronized方法，另外一个线程需要执行这个对象所属类的static synchronized方法，此时不会发生互斥现象，因为访问static synchronized方法占用的是类锁，而访问非static synchronized方法占用的是对象锁，所以不存在互斥现象。
