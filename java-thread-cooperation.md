---
title: Java并发编程：线程间协作的两种方式：wait、notify、notifyAll和Condition
date: 2017-12-6 17:28:03
tags: [java,并发编程]
categories: 学习笔记
---

原文链接 <a rel="external nofollow" target="_blank" href="http://www.cnblogs.com/dolphin0520/p/3920385.html">Java并发编程：线程间协作的两种方式：wait、notify、notifyAll和Condition</a>

在前面我们将了很多关于同步的问题，然而在现实中，需要线程之间的协作。比如说最经典的生产者-消费者模型：当队列满时，生产者需要等待队列有空间才能继续往里面放入商品，而在等待的期间内，生产者必须释放对临界资源（即队列）的占用权。因为生产者如果不释放对临界资源的占用权，那么消费者就无法消费队列中的商品，就不会让队列有空间，那么生产者就会一直无限等待下去。因此，一般情况下，当队列满时，会让生产者交出对临界资源的占用权，并进入挂起状态。然后等待消费者消费了商品，然后消费者通知生产者队列有空间了。同样地，当队列空时，消费者也必须等待，等待生产者通知它队列中有商品了。这种互相通信的过程就是线程间的协作。

<!--more-->

今天我们就来探讨一下Java中线程协作的最常见的两种方式：利用Object.wait()、Object.notify()和使用Condition。

### wait()、notify()和notifyAll()
wait()、notify()和notifyAll()是Object类中的方法：
```java
/**
 * Wakes up a single thread that is waiting on this object's
 * monitor. If any threads are waiting on this object, one of them
 * is chosen to be awakened. The choice is arbitrary and occurs at
 * the discretion of the implementation. A thread waits on an object's
 * monitor by calling one of the wait methods
 */
public final native void notify();
 
/**
 * Wakes up all threads that are waiting on this object's monitor. A
 * thread waits on an object's monitor by calling one of the
 * wait methods.
 */
public final native void notifyAll();
 
/**
 * Causes the current thread to wait until either another thread invokes the
 * {@link java.lang.Object#notify()} method or the
 * {@link java.lang.Object#notifyAll()} method for this object, or a
 * specified amount of time has elapsed.
 * <p>
 * The current thread must own this object's monitor.
 */
public final native void wait(long timeout) throws InterruptedException;
```

从这三个方法的文字描述可以知道以下几点信息：

　　1. wait()、notify()和notifyAll()方法是本地方法，并且为final方法，无法被重写。

　　2. 调用某个对象的wait()方法能让当前线程阻塞，并且当前线程必须拥有此对象的monitor（即锁）

　　3. 调用某个对象的notify()方法能够唤醒一个正在等待这个对象的monitor的线程，如果有多个线程都在等待这个对象的monitor，则只能唤醒其中一个线程；

　　4. 调用notifyAll()方法能够唤醒所有正在等待这个对象的monitor的线程；

　　有朋友可能会有疑问：为何这三个不是Thread类声明中的方法，而是Object类中声明的方法（当然由于Thread类继承了Object类，所以Thread也可以调用者三个方法）？其实这个问题很简单，由于每个对象都拥有monitor（即锁），所以让当前线程等待某个对象的锁，当然应该通过这个对象来操作了。而不是用当前线程来操作，因为当前线程可能会等待多个线程的锁，如果通过线程来操作，就非常复杂了。

　　上面已经提到，如果调用某个对象的wait()方法，当前线程必须拥有这个对象的monitor（即锁），因此调用wait()方法必须在同步块或者同步方法中进行（synchronized块或者synchronized方法）。

　　调用某个对象的wait()方法，相当于让当前线程交出此对象的monitor，然后进入等待状态，等待后续再次获得此对象的锁（Thread类中的sleep方法使当前线程暂停执行一段时间，从而让其他线程有机会继续执行，但它并不释放对象锁）；

　　notify()方法能够唤醒一个正在等待该对象的monitor的线程，当有多个线程都在等待该对象的monitor的话，则只能唤醒其中一个线程，具体唤醒哪个线程则不得而知。

　　同样地，调用某个对象的notify()方法，当前线程也必须拥有这个对象的monitor，因此调用notify()方法必须在同步块或者同步方法中进行（synchronized块或者synchronized方法）。

　　nofityAll()方法能够唤醒所有正在等待该对象的monitor的线程，这一点与notify()方法是不同的。

　　这里要注意一点：notify()和notifyAll()方法只是唤醒等待该对象的monitor的线程，并不决定哪个线程能够获取到monitor。

　　举个简单的例子：假如有三个线程Thread1、Thread2和Thread3都在等待对象objectA的monitor，此时Thread4拥有对象objectA的monitor，当在Thread4中调用objectA.notify()方法之后，Thread1、Thread2和Thread3只有一个能被唤醒。注意，被唤醒不等于立刻就获取了objectA的monitor。假若在Thread4中调用objectA.notifyAll()方法，则Thread1、Thread2和Thread3三个线程都会被唤醒，至于哪个线程接下来能够获取到objectA的monitor就具体依赖于操作系统的调度了。

　　上面尤其要注意一点，一个线程被唤醒不代表立即获取了对象的monitor，只有等调用完notify()或者notifyAll()并退出synchronized块，释放对象锁后，其余线程才可获得锁执行。

下面看一个例子就明白了：
```java
public class Test {
    public static Object object = new Object();
    public static void main(String[] args) {
        Thread1 thread1 = new Thread1();
        Thread2 thread2 = new Thread2();
         
        thread1.start();
         
        try {
            Thread.sleep(200);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
         
        thread2.start();
    }
     
    static class Thread1 extends Thread{
        @Override
        public void run() {
            synchronized (object) {
                try {
                    object.wait();
                } catch (InterruptedException e) {
                }
                System.out.println("线程"+Thread.currentThread().getName()+"获取到了锁");
            }
        }
    }
     
    static class Thread2 extends Thread{
        @Override
        public void run() {
            synchronized (object) {
                object.notify();
                System.out.println("线程"+Thread.currentThread().getName()+"调用了object.notify()");
            }
            System.out.println("线程"+Thread.currentThread().getName()+"释放了锁");
        }
    }
}
```
无论运行多少次，运行结果必定是：
```
线程Thread-1调用了object.notify()
线程Thread-1释放了锁
线程Thread-0获取到了锁
```
### Condition
Condition是在java 1.5中才出现的，它用来替代传统的Object的wait()、notify()实现线程间的协作，相比使用Object的wait()、notify()，使用Condition1的await()、signal()这种方式实现线程间协作更加安全和高效。因此通常来说比较推荐使用Condition，在阻塞队列那一篇博文中就讲述到了，阻塞队列实际上是使用了Condition来模拟线程间协作。

* Condition是个接口，基本的方法就是await()和signal()方法；
* Condition依赖于Lock接口，生成一个Condition的基本代码是lock.newCondition()；
* 调用Condition的await()和signal()方法，都必须在lock保护之内，就是说必须在lock.lock()和lock.unlock之间才可以使用。

Conditon中的await()对应Object的wait()；

Condition中的signal()对应Object的notify()；

Condition中的signalAll()对应Object的notifyAll()。
　　
### 生产者-消费者模型的实现

#### 使用Object的wait()和notify()实现
```java
public class Test {
    private int queueSize = 10;
    private PriorityQueue<Integer> queue = new PriorityQueue<Integer>(queueSize);
      
    public static void main(String[] args)  {
        Test test = new Test();
        Producer producer = test.new Producer();
        Consumer consumer = test.new Consumer();
          
        producer.start();
        consumer.start();
    }
      
    class Consumer extends Thread{
          
        @Override
        public void run() {
            consume();
        }
          
        private void consume() {
            while(true){
                synchronized (queue) {
                    while(queue.size() == 0){
                        try {
                            System.out.println("队列空，等待数据");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notify();
                        }
                    }
                    queue.poll();          //每次移走队首元素
                    queue.notify();
                    System.out.println("从队列取走一个元素，队列剩余"+queue.size()+"个元素");
                }
            }
        }
    }
      
    class Producer extends Thread{
          
        @Override
        public void run() {
            produce();
        }
          
        private void produce() {
            while(true){
                synchronized (queue) {
                    while(queue.size() == queueSize){
                        try {
                            System.out.println("队列满，等待有空余空间");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notify();
                        }
                    }
                    queue.offer(1);        //每次插入一个元素
                    queue.notify();
                    System.out.println("向队列取中插入一个元素，队列剩余空间："+(queueSize-queue.size()));
                }
            }
        }
    }
}
```

#### 使用Condition实现
```java
public class Test {
    private int queueSize = 10;
    private PriorityQueue<Integer> queue = new PriorityQueue<Integer>(queueSize);
    private Lock lock = new ReentrantLock();
    private Condition notFull = lock.newCondition();
    private Condition notEmpty = lock.newCondition();
     
    public static void main(String[] args)  {
        Test test = new Test();
        Producer producer = test.new Producer();
        Consumer consumer = test.new Consumer();
          
        producer.start();
        consumer.start();
    }
      
    class Consumer extends Thread{
          
        @Override
        public void run() {
            consume();
        }
          
        private void consume() {
            while(true){
                lock.lock();
                try {
                    while(queue.size() == 0){
                        try {
                            System.out.println("队列空，等待数据");
                            notEmpty.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    queue.poll();                //每次移走队首元素
                    notFull.signal();
                    System.out.println("从队列取走一个元素，队列剩余"+queue.size()+"个元素");
                } finally{
                    lock.unlock();
                }
            }
        }
    }
      
    class Producer extends Thread{
          
        @Override
        public void run() {
            produce();
        }
          
        private void produce() {
            while(true){
                lock.lock();
                try {
                    while(queue.size() == queueSize){
                        try {
                            System.out.println("队列满，等待有空余空间");
                            notFull.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    queue.offer(1);        //每次插入一个元素
                    notEmpty.signal();
                    System.out.println("向队列取中插入一个元素，队列剩余空间："+(queueSize-queue.size()));
                } finally{
                    lock.unlock();
                }
            }
        }
    }
}
```

### 参考资料
* 《Java编程思想》
*  <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/ns_code/article/details/17225469">http://blog.csdn.net/ns_code/article/details/17225469</a>
*   <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/ghsau/article/details/7481142">http://blog.csdn.net/ghsau/article/details/7481142</a>
