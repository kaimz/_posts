---
title: Java 中的并发工具类
date: 2018-05-20 10:08:03
tags: [java,并发编程]
categories: 学习笔记
---

`java.util.concurrent` 下提供了一些辅助类来帮助我们在并发编程的设计。

学习了 AQS 后再了解这些工具类，就非常简单了。

*jdk 1.8*

<!--more-->

### 等待多线程完成的CountDownLatch

在 `concurrent` 包下面提供了 `CountDownLatch` 类，它提供了计数器的功能，能够实现让一个线程等待其他线程执行完毕才能进入运行状态。

#### 源码分析

1. 首先看下最关键的地方它的自定义同步器的实现，非常简单：

   ```java
       private static final class Sync extends AbstractQueuedSynchronizer {
           private static final long serialVersionUID = 4982264981922014374L;
   		// 1. 初始化 state 资源变量
           Sync(int count) {
               setState(count);
           }
   
           int getCount() {
               return getState();
           }
   		// 尝试获取贡献模式下的资源，
           // 定义返回值小于 0 的时候获取资源失败
           protected int tryAcquireShared(int acquires) {
               return (getState() == 0) ? 1 : -1;
           }
   
           protected boolean tryReleaseShared(int releases) {
               // Decrement count; signal when transition to zero
               // 自旋。
               for (;;) {
                   int c = getState();
                   if (c == 0)
                       return false;
                   int nextc = c - 1; // 每次释放资源，硬编码减一个资源
                   if (compareAndSetState(c, nextc))
                       return nextc == 0; // 知道为 0 的时候才释放成功，也就是所有线程必须都执行释放操作说明才释放成功。
                
               }
           }
       }
   ```

   在这里查看构造器的源码得知，`CountDownLatch` 内部使用的是 内部类`Sync` 继承了 `AQS` ，将我们传入进来的 `count` 数值当作 AQS state。感觉这个是不是和可重入锁实现是一样的，只不过开始指定了线程获取的锁的次数。

   在上面我也发现了几个特点，第一次看这个代码其实还是不好理解，因为它相对前面的 AQS 和 TwinsLock 就是一个反着设计的代码：

   1. 首先获取资源的时候，线程全部都是先进入等待队列，而且在这一步骤中，不改变 state 资源的数量；
   2. 释放资源的时候，每次固定减少一个资源，直到资源为 0 的时候才表示释放资源成功，所以加入我们有 5 个资源，但是只有四个线程执行，如果只释放四次（总共执行 countDown 四次），就永远也释放不成功，await 一直在阻塞。
   3. 经过上面的分析，发现了 state 的资源数量每次进行 `countDown` 都去减少一个，没有方法去增加数量，所以它是不可逆的，它的计数器是不可以重复使用的。

2. 看下 await 的实现，发现它最终实现的是 `doAcquireSharedInterruptibly` ：

   ```java
       // 仔细看这个代码，和前面的共享模式中的 doAcquireShared 方法基本一摸一样，只不过是当它遇到线程中断信号的时候，立刻抛出中断异常，仔细想想也是的，比如，自己在这里等别人吃饭，不想等了，也懒得管别人做什么了，剩下的吃饭的事情也没必要继续下去了。
   	private void doAcquireSharedInterruptibly(int arg)
           throws InterruptedException {
           final Node node = addWaiter(Node.SHARED);
           boolean failed = true;
           try {
               for (;;) {
                   final Node p = node.predecessor();
                   if (p == head) {
                       int r = tryAcquireShared(arg);
                       if (r >= 0) {
                           setHeadAndPropagate(node, r);
                           p.next = null; // help GC
                           failed = false;
                           return;
                       }
                   }
                   if (shouldParkAfterFailedAcquire(p, node) &&
                       parkAndCheckInterrupt())
                       throw new InterruptedException();
               }
           } finally {
               if (failed)
                   cancelAcquire(node);
           }
       }
   		// 需要注意的是它重写了尝试获取资源的方法，当资源全部消耗完，才能够让你去获取资源，现在才豁然开朗，await 阻塞的线程就是这么被唤醒的。
           protected int tryAcquireShared(int acquires) {
               return (getState() == 0) ? 1 : -1;
           }
   ```

#### 使用场景

CountDownLatch允许一个或多个线程等待其他线程完成操作。

比如经典问题：

> 有Thread1、Thread2、Thread3、Thread4四条线程分别统计C、D、E、F四个盘的大小，所有线程都统计完毕交给Thread5线程去做汇总，应当如何实现？

这个问题关键就是要知道**四条线程何时执行完。**

下面是我的解决思路：

```java
/**
 * 如有Thread1、Thread2、Thread3、Thread4四条线程分别统计C、D、E、F四个盘的大小，
 * 所有线程都统计完毕交给Thread5线程去做汇总，应当如何实现？
 *
 * Created by KronChan on 2018/5/14 17:00.
 */
public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        // 初始化计数器，设置总量i，调用一次countDown(）方法后i的值会减1。
        // 在一个线程中如果调用了await()方法，这个线程就会进入到等待的状态，当参数i为0的时候这个线程才继续执行。
        final CountDownLatch countDownLatch = new CountDownLatch(4);
        Runnable thread1 = () -> {
            try {
                TimeUnit.SECONDS.sleep(2);
                System.out.println("统计 C 盘大小");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // 统计完成计数器 -1
            countDownLatch.countDown();
        };
        Runnable thread2 = () -> {
            try {
                TimeUnit.SECONDS.sleep(2);
                System.out.println("统计 D 盘大小");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            countDownLatch.countDown();
        };
        Runnable thread3 = () -> {
            try {
                TimeUnit.SECONDS.sleep(2);
                System.out.println("统计 E 盘大小");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            countDownLatch.countDown();
        };
        Runnable thread4 = () -> {
            try {
                TimeUnit.SECONDS.sleep(2);
                System.out.println("统计 F 盘大小");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            countDownLatch.countDown();
        };

        ExecutorService pool = Executors.newFixedThreadPool(4);
        pool.execute(thread1);
        pool.execute(thread2);
        pool.execute(thread3);
        pool.execute(thread4);
        // 等待 i 值为 0 ，等待四条线程执行完毕。
        countDownLatch.await();
        System.out.println("统计完成");
        pool.shutdown();
    }
}
```

### 同步屏障CyclicBarrier

CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

#### 源码分析

##### 属性

```java
private final ReentrantLock lock = new ReentrantLock();
// 线程协作
private final Condition trip = lock.newCondition();
// 必须同时到达barrier的线程个数。
private final int parties;
// parties个线程到达barrier时，会执行的动作，会让到达屏障中的任意一个线程去执行这个动作。
private final Runnable barrierCommand;
// 控制屏障的循环使用，它是可重复使用的，每次使用CyclicBarrier，本次所有线程同属于一代，即同一个Generation
private Generation generation = new Generation();
 // 处在等待状态的线程个数。
private int count;
```

##### 主要的方法

###### 构造函数

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}
// 构造函数主要实现了，设置一组线程的数量，到达屏障时候的临界点，可以设置到达屏障的时候需要处理的动作，后面屏障允许它们通过。
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

###### await

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen;
    }
}

private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    // 独占锁
    lock.lock();
    try {
        // 保存当前的generation
        final Generation g = generation;

        // generation broken，不允许使用，则抛出异常。
        if (g.broken)
            throw new BrokenBarrierException();

        // 如果当前线程被中断，则通过breakBarrier()终止CyclicBarrier，唤醒CyclicBarrier中所有等待线程。
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

       // 等待的计数器减一
       int index = --count;
       // 如果计数器的 count 正好为0， 说明已经有parties个线程到达barrier了。执行预定的Runnable任务后，更新换代，准备下一次使用。
       if (index == 0) {  // tripped
           boolean ranAction = false;
           try {
               // 如果barrierCommand不为null，则执行该动作。
               final Runnable command = barrierCommand;
               if (command != null)
                   command.run();
               ranAction = true;
               // 唤醒所有等待线程，并更新generation，准备下一次使用
               nextGeneration();
               return 0;
           } finally {
               if (!ranAction)
                   breakBarrier();
           }
       }

        // 当前线程一直阻塞，
        // 1. 有parties个线程到达barrier
        // 2. 当前线程被中断
        // 3. 超时
        // 直到上面三者之一发生，就唤醒所有线程继续执行下去
        // 自旋
        for (;;) {
            try {
                // 如果不是超时等待，则调用awati()进行等待；否则，调用awaitNanos()进行等待。
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                // 如果等待过程中，线程被中断，通过breakBarrier()终止CyclicBarrier，唤醒CyclicBarrier中所有等待线
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    Thread.currentThread().interrupt();
                }
            }

            // borken
            if (g.broken)
                throw new BrokenBarrierException();

            // 如果generation已经换代，则返回index。
            if (g != generation)
                return index;

            // 超时，则通过breakBarrier()终止CyclicBarrier，唤醒CyclicBarrier中所有等待线程。
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock(); // 释放独占锁
    }
}
// barrier被broken后，调用breakBarrier方法，将generation.broken设置为true，并使用signalAll通知所有等待的线程。
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}
```

#### 使用场景

CyclicBarrier可以用于多线程计算数据，最后合并计算结果的场景，然后四条线程又可以分别去干自己的事情了。

现在我将上面的统计磁盘的任务 `CountDownLatch` 中改下，统计完统计最终后，每个线程要发出退出信号。

下面是我的实现代码：

```java
public class CyclicBarrierDemo {
    public static void main(String[] args) {
        String[] drivers = {"C", "D", "E", "F"};
        int length = drivers.length;
        ExecutorService pool = Executors.newFixedThreadPool(length);
        // 如果线程都到达barrier状态后，会从四个线程中选择一个线程去执行Runnable。
        CyclicBarrier cyclicBarrier = new CyclicBarrier(length, () -> {
            System.out.printf("%s 线程告诉你，统计完毕，待继续执行%n", Thread.currentThread().getName());
        });
        Stream.of(drivers).forEach((d) -> {
            pool.execute(new StatisticsDemo(d, cyclicBarrier));
        });
        pool.shutdown();
    }

    static class StatisticsDemo implements Runnable {

        private String driveName;

        private CyclicBarrier cyclicBarrier;

        public StatisticsDemo(String driveName, CyclicBarrier cyclicBarrier) {
            this.driveName = driveName;
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            try {
                TimeUnit.SECONDS.sleep((int) (Math.random() * 10));
                System.out.printf("%s 线程统计 %s 盘大小%n", Thread.currentThread().getName(), driveName);
                cyclicBarrier.await();
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
            System.out.printf("%s 准备退出%n", driveName);
        }
    }
}
```

执行结果：

```java
pool-1-thread-1 线程统计 C 盘大小
pool-1-thread-2 线程统计 D 盘大小
pool-1-thread-3 线程统计 E 盘大小
pool-1-thread-4 线程统计 F 盘大小
pool-1-thread-4 线程告诉你，统计完毕，待继续执行
F 准备退出
E 准备退出
D 准备退出
C 准备退出
```

### 控制并发线程数的Semaphore

> `Semaphore`（信号量）是用来控制同时访问特定资源的线程数量（许可证数），它通过协调各个线程，以保证合理的使用公共资源。

#### 源码分析

##### 构造函数

```java
public Semaphore(int permits) {
     sync = new NonfairSync(permits);
}
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

具有公平锁的特性，`permits` 指定许可数量，就是资源数量 `state`。 

##### 同步器的实现

```java
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }
		// 非公平的方式获取共享锁
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                // 获取资源数量
                int available = getState();
                int remaining = available - acquires; // 本次请求获取锁需要的资源的数量
                if (remaining < 0 ||
                    compareAndSetState(available, remaining)) // 如果资源足够，尝试 CAS 获取锁
                    return remaining;
            }
        }
		// 释放锁
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases; // 释放锁的时候，返还资源
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next)) // CAS 操作，避免其他的线程也在释放资源
                    return true;
            }
        }
		// 减少资源数量
        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }
		// 清空资源，返回历史资源数量
        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

    /**
     * NonFair version
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

    /**
     * Fair version
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                // 同样的公平锁情况下，判断该线程前面有没有线程等待获取锁
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

```

##### 提供其他的方法

* `availablePermits`：获取此信号量中当前可用的许可证数（还能有多少个线程执行）；
* `drainPermits`：立刻使用完所有可用的许可证；
* `reducePermits`：减少相应数量的许可证，是一个 `protected` 方法；
* `isFair`：是否是公平状态；
* `hasQueuedThreads`：等待队列中是否有线程，等待获取许可证；
* `getQueueLength`：等待队列中等待获取许可证的线程数量；
* `getQueuedThreads`：`protected` 方法，获取等待队列中的线程。

#### 使用场景

**`Semaphore`可以用于做流量控制，特别是公用资源有限的应用场景**，比如我们有五台机器，有十名工人，每个工人需要一台机器才能工作，一名工人工作完了就可以休息了，机器让其他没工作过的工人使用。

下面是我的实现代码：

```java
public class SemaphoreDemo {
    public static void main(String[] args) {
        int num = 10;
        Semaphore machines = new Semaphore(5);
        for (int i = 0; i < num; i++) {
            new Thread(new Worker(i, machines)).start();
        }
    }

    static class Worker extends Thread {
        private Semaphore machines;

        private int worker;

        Worker(int worker, Semaphore semaphore) {
            this.worker = worker;
            this.machines = semaphore;
        }
        @Override
        public void run() {
            try {
                machines.acquire();
                System.out.printf("工人 %d 开始使用机器工作了 %n", worker);
                TimeUnit.SECONDS.sleep((int) (Math.random() * 10));
                System.out.printf("工人 %d 干完活了，让出机器了%n", worker);
                machines.release();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

}
```

执行一下结果：

```java
工人 0 开始使用机器工作了 
工人 4 开始使用机器工作了 
工人 3 开始使用机器工作了 
工人 2 开始使用机器工作了 
工人 1 开始使用机器工作了 
工人 1 干完活了，让出机器了
工人 5 开始使用机器工作了 
工人 5 干完活了，让出机器了
工人 6 开始使用机器工作了 
工人 2 干完活了，让出机器了
工人 7 开始使用机器工作了 
工人 4 干完活了，让出机器了
工人 8 开始使用机器工作了 
工人 0 干完活了，让出机器了
工人 9 开始使用机器工作了 
工人 8 干完活了，让出机器了
工人 6 干完活了，让出机器了
工人 3 干完活了，让出机器了
工人 9 干完活了，让出机器了
工人 7 干完活了，让出机器了
```

虽然上面有 10 个工人（线程）一起并发，但是，它同时只有五个工人能够是执行的。

### 线程间交换数据的Exchanger

Exchanger（交换者）是一个用于线程间协作的工具类。Exchanger 用于两个工作线程间的数据交换。

具体上来说，Exchanger类允许在两个线程之间定义同步点。当两个线程都到达同步点时，他们交换数据结构，因此第一个线程的数据进入到第二个线程中，第二个线程的数据进入到第一个线程中，这要就完成了一个“交易”的环节。

#### 源码分析

源码很难看懂，主要还是

[【死磕Java并发】-----J.U.C之并发工具类：Exchanger](https://www.jianshu.com/p/c523826b2c94)

#### 使用场景

**Exchanger 可以用于遗传算法**。遗传算法里需要选出两个人作为交配对象，这时候会交换两人的数据。

下面做一个卖书买书的例子：

```java
public class ExchangerDemo {
    private static final Exchanger<String> EXCHANGER = new Exchanger<>();
    private static final ExecutorService POOLS = Executors.newFixedThreadPool(2);

    public static void main(String[] args) throws InterruptedException {
        POOLS.execute(() -> {
            String bookName = "浮生六记";
            System.out.printf("饭饭要卖一本%s。%n", bookName);
            try {
                String pay = EXCHANGER.exchange(bookName);
                System.out.printf("饭饭卖出一本%s赚了%s￥。%n", bookName, pay);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        TimeUnit.SECONDS.sleep(5);
        System.out.println("》》》》》》》》饭饭先到了交易地点 睡了 5 s，七巧来了");
        System.out.println("》》》》》》》》准备交易");

        POOLS.execute(() -> {
            String pay = "23";
            try {
                String bookName = EXCHANGER.exchange(pay);
                System.out.printf("七巧付了%s￥买了一本%s。%n", pay, bookName);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        POOLS.shutdown();
        for (; ; ) {
            if (POOLS.isTerminated()) {
                System.out.println("交易结束！");
                return;
            }
        }
    }
}
```

执行结果：

```java
饭饭要卖一本浮生六记。
》》》》》》》》饭饭先到了交易地点 睡了 5 s，七巧来了
》》》》》》》》准备交易
七巧付了23￥买了一本浮生六记。
饭饭卖出一本浮生六记赚了23￥。
交易结束！
```

#### 总结

`Exchanger `主要完成的是两个工作线程之间的数据交换，如果有一个线程没有执行 `exchange()`方法，则会一直等待。还可以设置最大等待时间`exchange（V v, TimeUnit unit）`

### CyclicBarrier和CountDownLatch的区别

> CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置。所以CyclicBarrier能处理更为复杂的业务场景。例如，如果计算发生错误，可以重置计数器，并让线程重新执行一次。
>
> CyclicBarrier还提供其他有用的方法，比如`getNumberWaiting`方法可以获得 `CyclicBarrier`阻塞的线程数量。`isBroken()`方法用来了解阻塞的线程是否被中断。

### 参考文章

- 《Java 并发编程的艺术》
