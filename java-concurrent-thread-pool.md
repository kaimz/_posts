---
title: Java并发编程中线程池源码分析及使用
date: 2017-10-31 16:38:03
tags: [java,并发编程]
categories: 学习笔记
---

当Java处理高并发的时候，线程数量特别的多的时候，而且每个线程都是执行很短的时间就结束了，频繁创建线程和销毁线程需要占用很多系统的资源和时间，会降低系统的工作效率。


参考<a rel="external nofollow" target="_blank" href="http://www.cnblogs.com/dolphin0520/p/3932921.html">http://www.cnblogs.com/dolphin0520/p/3932921.html</a>

由于原文作者使用的API 是1.6 版本的，参考他的文章，做了一些修改成 jdk 1.8版本的方法，涉及到的内容比较多，可能有少许错误。

**API : jdk1.8.0_144**

<!--more-->

### ThreadPoolExecutor类
Java中线程池主要是并发包``java.util.concurrent`` 中 ``ThreadPoolExecutor``这个类实现的。
#### 构造函数

我们直接调用它的时候，使用的是它的构造函数，它有四个构造函数：
```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    //省略前面的代码
    
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
    
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }
    
   public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }
 
   public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
    
    //省略后面的代码
}
```

``ThreadPoolExecutor``继承了``AbstractExecutorService``抽象类，并提供了四个构造器，事实上，前面三个构造器都是调用的第四个构造器进行的初始化工作。所以主要研究下第四个构造器的方法。

首先了解下构造器中参数的意思：

* ``corePoolSize``: 核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了prestartAllCoreThreads()或者prestartCoreThread()方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中；
* ``maximumPoolSize``: 线程池最大线程数，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程；
* ``keepAliveTime``:字面意思就是心跳时间，就是这个线程池中的线程数量大于``corePoolSize``的时候开始计时，设置空闲线程最多能存活多长时间。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0，它的单位是参数``TimeUnit unit``；
* ``unit``: 参数``keepAliveTime``的时间单位，有7种取值，在TimeUnit类中有7种静态属性：
```java
TimeUnit.DAYS; //天
TimeUnit.HOURS; //小时
TimeUnit.MINUTES; //分钟
TimeUnit.SECONDS; //秒
TimeUnit.MILLISECONDS; //毫秒
TimeUnit.MICROSECONDS; //微妙
TimeUnit.NANOSECONDS; //纳秒
```
* ``workQueue``：一个阻塞队列``BlockingQueue``，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择，以后再详细学习``BlockingQueue``阻塞队列使用：
```java
ArrayBlockingQueue; //　基于数组的阻塞队列实现
LinkedBlockingQueue; // 基于链表的阻塞队列
SynchronousQueue; //一种无缓冲的等待队列
DelayQueue； // 队列中插入数据的操作（生产者）永远不会被阻塞，而只有获取数据的操作（消费者）才会被阻塞。
PriorityBlockingQueue // 基于优先级的阻塞队列
```
* ``threadFactory``: 线程工厂，主要用来创建线程；
* ``handler``: 表示当拒绝处理任务时的策略，有以下四种取值：
```java
ThreadPoolExecutor.AbortPolicy //丢弃任务并抛出RejectedExecutionException异常。 
ThreadPoolExecutor.DiscardPolicy //也是丢弃任务，但是不抛出异常。 
ThreadPoolExecutor.DiscardOldestPolicy //丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy //由调用线程处理该任务
```

#### ThreadPoolExecutor方法
首先``ThreadPoolExecutor``类自己拥有很多方法，用来获取线程池的相关属性。

```

```


``ThreadPoolExecutor``继承了``AbstractExecutorService``这个抽象类，

```java
public abstract class AbstractExecutorService implements ExecutorService{
 
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) { };
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) { };
    public Future<?> submit(Runnable task) {};
    public <T> Future<T> submit(Runnable task, T result) { };
    public <T> Future<T> submit(Callable<T> task) { };
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                            boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
    };
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
    };
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                           long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
    };
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
    };
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
    };
}
```

``AbstractExecutorService``实现了接口 ``ExecutorService``中所有的方法。

```java
public interface ExecutorService extends Executor {
  
    void shutdown();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
  
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```


``ExecutorService`` 接口继承了 ``Executor``接口。

```java
public interface Executor {
    void execute(Runnable command);
}
```
可以看出类``ThreadPoolExecutor``拥有了多少方法。

平时开发中主要使用方法：
```
execute() // 线程池启动一个线程
submit() // 线程池启动一个线程，有返回值
shutdown()  //执行完毕所有等待中的线程，再关闭线程池
shutdownNow() // 直接关闭，不等待
```

* execute()方法实际上是Executor中声明的方法，在ThreadPoolExecutor进行了具体的实现，这个方法是ThreadPoolExecutor的核心方法，通过这个方法可以向线程池提交一个任务，交由线程池去执行。

* submit()方法是在ExecutorService中声明的方法，在AbstractExecutorService就已经有了具体的实现，在ThreadPoolExecutor中并没有对其进行重写，这个方法也是用来向线程池提交任务的，但是它和execute()方法不同，它能够返回任务执行的结果，去看submit()方法的实现，会发现它实际上还是调用的execute()方法，只不过它利用了Future来获取任务执行结果。

* shutdown()和shutdownNow()是用来关闭线程池的。

### 线程池的实现

#### 线程池的状态

```java
* The runState provides the main lifecycle control, taking on values:
     *
     *   RUNNING:  Accept new tasks and process queued tasks
     *   SHUTDOWN: Don't accept new tasks, but process queued tasks
     *   STOP:     Don't accept new tasks, don't process queued tasks,
     *             and interrupt in-progress tasks
     *   TIDYING:  All tasks have terminated, workerCount is zero,
     *             the thread transitioning to state TIDYING
     *             will run the terminated() hook method
     *   TERMINATED: terminated() has completed
     *
     * The numerical order among these values matters, to allow
     * ordered comparisons. The runState monotonically increases over
     * time, but need not hit each state. The transitions are:
     *
     * RUNNING -> SHUTDOWN
     *    On invocation of shutdown(), perhaps implicitly in finalize()
     * (RUNNING or SHUTDOWN) -> STOP
     *    On invocation of shutdownNow()
     * SHUTDOWN -> TIDYING
     *    When both queue and pool are empty
     * STOP -> TIDYING
     *    When pool is empty
     * TIDYING -> TERMINATED
     *    When the terminated() hook method has completed
     *
     * Threads waiting in awaitTermination() will return when the
     * state reaches TERMINATED.
```
根据上面的代码文档，，可以清楚的了解到线程池的各种状态，以及在这种状态中能做的事情，状态之间的转变。

如果调用了shutdown()方法，则线程池处于SHUTDOWN状态，此时线程池不能够接受新的任务，它会等待所有任务执行完毕；

如果调用了shutdownNow()方法，则线程池处于STOP状态，此时线程池不能接受新的任务，并且会去尝试终止正在执行的任务；

当线程池处于SHUTDOWN或STOP状态，并且所有工作线程已经销毁，任务缓存队列已经清空或执行结束后，线程池被设置为TERMINATED状态。


```java

    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3; //29
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;   //536870911 目前最大线程容量

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS; // 111 00000000000000000000000000000
    private static final int SHUTDOWN   =  0 << COUNT_BITS; // 000 00000000000000000000000000000
    private static final int STOP       =  1 << COUNT_BITS; // 001 00000000000000000000000000000
    private static final int TIDYING    =  2 << COUNT_BITS; // 010 00000000000000000000000000000 
    private static final int TERMINATED =  3 << COUNT_BITS; // 100 00000000000000000000000000000

    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; } //最高3位， 状态
    private static int workerCountOf(int c)  { return c & CAPACITY; } //后29位 ，工作数量
    private static int ctlOf(int rs, int wc) { return rs | wc; }

```
ctl作为ThreadPoolExecutor的核心状态控制字段，包含来两个信息：
* 工作线程总数  ``workerCount``
* 线程池状态 ``RUNNING``、 ``SHUTDOWN``、 ``STOP``、 ``TIDYING``、 ``TERMINATED``。

COUNT_BITS 是32减去3 就是29，下面的线程池状态就是－1 到 3 分别向左移动29位。

 如此，int的右侧29位，代表着线程数量，总数可以达到2的29次，29位后的3位代表线程池的状态
这样，线程池增加一个线程，只需吧ctl加1即可，而我们也发现实际这个线程池的最高线程数量是2的29次减1。并不是先前我们现象的2的32次减1。这个作者在注释中也提到了，说如果后续需要增大这个值， 可以吧ctl定义成AtomicLong。

#### 任务的执行excute
##### 属性变量
了解``ThreadPoolExecutor``类中其他的一些比较重要成员变量：
```java

private final BlockingQueue<Runnable> workQueue;              //任务缓存队列，用来存放等待执行的任务
private final ReentrantLock mainLock = new ReentrantLock();   //线程池的主要状态锁，对线程池状态（比如线程池大小
                                                              //、runState等）的改变都要使用这个锁
private final HashSet<Worker> workers = new HashSet<Worker>();  //用来存放工作集
 
private volatile long  keepAliveTime;    //线程存货时间   
private volatile boolean allowCoreThreadTimeOut;   //是否允许为核心线程设置存活时间
private volatile int   corePoolSize;     //核心池的大小（即线程池中的线程数目大于这个参数时，提交的任务会被放进任务缓存队列）
private volatile int   maximumPoolSize;   //线程池最大能容忍的线程数
 
private volatile int   poolSize;       //线程池中当前的线程数
 
private volatile RejectedExecutionHandler handler; //任务拒绝策略
 
private volatile ThreadFactory threadFactory;   //线程工厂，用来创建线程
 
private int largestPoolSize;   //用来记录线程池中曾经出现过的最大线程数
 
private long completedTaskCount;   //用来记录已经执行完毕的任务个数


 /**
 * Wait condition to support awaitTermination
 */
private final Condition termination = mainLock.newCondition(); //线程等待时的关闭的条件

/* The context to be used when executing the finalizer, or null. */
private final AccessControlContext acc; // 执行任务完成后使用的内容，或者为null
```
* largestPoolSize只是一个用来起记录作用的变量，用来记录线程池中曾经有过的最大线程数目，跟线程池的容量没有任何关系。
* 线程池线程一般正常工作的时候最大线程数为corePoolSize，当任务数量大于corePoolSize的时候，任务就进入等待的队列中，不继续增加线程；当等待队列也放满的时候，不能再往里面装任务的时候，这个时候就需要重新开辟新的线程，来工作了，并且数量要小于``maximumPoolSize``；如果大于maximumPoolSize，就调用handler方法。

##### 执行任务 execute
使用``AbstractExecuorService``中的submit 方法，可以执行新的进程，当然submit，最终执行的是execute方法，在``ThreadPoolExecutor``类中实现了excute方法；

重点研究exexute 方法的实现，这个有点难，网上介绍1.6里面的源码中execute方法已经和我这个1.8版本有很大出入了，大致上应该没有偏离：
```java

/**
     * Executes the given task sometime in the future.  The task
     * may execute in a new thread or in an existing pooled thread.
     *
     * If the task cannot be submitted for execution, either because this
     * executor has been shutdown or because its capacity has been reached,
     * the task is handled by the current {@code RejectedExecutionHandler}.
     *
     * @param command the task to execute
     * @throws RejectedExecutionException at discretion of
     *         {@code RejectedExecutionHandler}, if the task
     *         cannot be accepted for execution
     * @throws NullPointerException if {@code command} is null
     */
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```
执行流程就是：
1. 判断提交的任务command是否为null，若是null，则抛出空指针异常；
2. 第二步 ct1.get()；用这个``workerCountOf( ct1.get())`` 计算线程池已经使用多少线程；
3. 当使用的线程数小于核心线程数（corePoolSize），进入addWorker 方法中，这里就是开始进程的地方，进入到最重要的地方，为了这一步不要跳得太远，还是接着看execute方法，后面再看addWorker方法；
4. 当使用的线程数不小于核心线程数（corePoolSize），新来得任务就要进入等待执行的状态；  
    `` if (isRunning(c) && workQueue.offer(command)) `` 检查线程是否在running 状态和任务是否能够成功进入等待``排队`` ；   
4.1. 进入队列后，重新检查任务，如果线程池状态不是running状态， ，将回滚任务，拒绝执行任务，这样做主要是因为任务如果还在缓存队列等待的过程中，线程池中断了，就回滚任务，为了安全。  
4.2. 如果线程中的线程数为0 了，创建一个空线程。
5. 当使用的线程数不小于核心线程数（corePoolSize）的时候，并且添加进入到缓存队列失败后，就会执行``else if (!addWorker(command, false))reject(command);`` 这段代码，意思就是直接开辟一个新的线程去行这个任务，如果执行失败，拒绝策略进行处理这个任务，当然，如果当前线程池中的线程数目达到``maximumPoolSize``，addWorker方法中也会采取任务拒绝策略进行处理。

##### addWorker 创建线程
下面将是阅读``addWorker``的源码，研究线程池怎么添加一个任务的。
```java
    /**
     * Checks if a new worker can be added with respect to current
     * pool state and the given bound (either core or maximum). If so,
     * the worker count is adjusted accordingly, and, if possible, a
     * new worker is created and started, running firstTask as its
     * first task. This method returns false if the pool is stopped or
     * eligible to shut down. It also returns false if the thread
     * factory fails to create a thread when asked.  If the thread
     * creation fails, either due to the thread factory returning
     * null, or due to an exception (typically OutOfMemoryError in
     * Thread.start()), we roll back cleanly.
     *
     * @param firstTask the task the new thread should run first (or
     * null if none). Workers are created with an initial first task
     * (in method execute()) to bypass queuing when there are fewer
     * than corePoolSize threads (in which case we always start one),
     * or when the queue is full (in which case we must bypass queue).
     * Initially idle threads are usually created via
     * prestartCoreThread or to replace other dying workers.
     *
     * @param core if true use corePoolSize as bound, else
     * maximumPoolSize. (A boolean indicator is used here rather than a
     * value to ensure reads of fresh values after checking other pool
     * state).
     * @return true if successful
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask); 
            final Thread t = w.thread; //创建一个线程
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) { //当任务成功添加到线程池，去执行它，改变标志符号。
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
看代码注释知道了第二个参数``core``的意义，当它为``true``的时候 使用的是线程核心数中的线程，当它为``false`` 的时候，使用的是数量是maximumPoolSize，就是当缓存中的队列也排满的时候。

因此，调用这个 addWorker方法有4种传参的方式：
```
addWorker(command, true);
addWorker(command, false);
addWorker(null, false);
addWorker(null, true);
```
1. 第一个：线程数小于corePoolSize时，放一个需要处理的task进worker set。如果worker set长度超过corePoolSize，就返回false。
2. 第二个：当队列被放满时，就尝试将这个新来的task直接放入worker set，而此时worker set 的长度限制是maximumPoolSize。如果线程池也满了的话就返回false。
3. 第三个：放入一个空的task进set，比较的的长度限制是maximumPoolSize。这样一个task为空的worker在线程执行的时候会判断出后去任务队列里拿任务，这样就相当于世创建了一个新的线程，只是没有马上分配任务。
4.   第四个：这个方法就是放一个null的task进set，而且是在小于corePoolSize时。实际使用中是在 prestartCoreThread() 方法。这个方法用来为线程池先启动一个worker等待在那边，如果此时set中的数量已经达到corePoolSize那就返回false，什么也不干。还有是 ``prestartAllCoreThreads()`` 方法，准备corePoolSize个worker，初始化线程池中的线程。  
默认情况下，创建线程池之后，线程池中是没有线程的，需要提交任务之后才会创建线程。在实际中如果需要线程池创建之后立即创建线程，可以通过以下两个方法办到：
```
prestartCoreThread()：初始化一个核心线程；
prestartAllCoreThreads()：初始化所有核心线程
```

前面代码的意思就是验证线程池的状态是不是在``RUNNING``状态，并且判断，线程数是不是超过了``maximumPoolSize``，如果超过了最大线程数量，直接返回false，就回到execute 方法最后个``if else()``代码块中，拒绝任务。

##### Worker 中主要实现
``Worker`` 这个类很简单，只是继承了一个``Runnable``接口，然后在``run()``方法中去执行我们传入的``firstTask`` 主要是其中的run 方法，它的run方法调用的是``runWorker``：
```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

注意当没有可执行的任务的时候，执行``getTask()``方法：
```java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) { //判断线程状态和缓存队列中的线程是否为空
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) { 
                //也就是说如果线程池处于STOP状态、或者任务队列已为空或者允许为核心池线程设置空闲存活时间并且线程数大于1时，允许worker退出。
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```
这个时候看到了，它原来去缓存队列中去取任务，来执行。


并且下面代码块做的任务，作者已经给出注释了
```
// Recheck while holding lock.
// Back out on ThreadFactory failure or if
// shut down before lock acquired.
```
很容易理解了这段代码。

怎么样开启线程池，并且添加一个任务就到此结束了。

#### 任务拒绝策略
当线程池的任务缓存队列已满并且线程池中的线程数目达到maximumPoolSize，如果还有任务到来就会采取任务拒绝策略，通常有以下四种策略：
```
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
```

#### 任务缓存队列及排队策略

workQueue，任务缓存队列，用来存放等待执行的任务；  
一个阻塞队列``BlockingQueue``，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择：
```java
ArrayBlockingQueue; //　基于数组的阻塞队列实现，此队列创建时必须指定大小；
LinkedBlockingQueue; // 基于链表的阻塞队列，如果创建时没有指定此队列大小，则默认为Integer.MAX_VALUE；
SynchronousQueue; //一种无缓冲的等待队列，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。
DelayQueue； // 队列中插入数据的操作（生产者）永远不会被阻塞，而只有获取数据的操作（消费者）才会被阻塞。
PriorityBlockingQueue // 基于优先级的阻塞队列
```
#### 线程池关闭
ThreadPoolExecutor提供了两个方法，用于线程池的关闭，分别是shutdown()和shutdownNow()，其中：
* shutdown()：不会立即终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务；
* shutdownNow()：立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务。

#### 创建线程池并且使用

```java
package com.wuwii.test;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/1 11:08</pre>
 */
public class TestPool {
    private static final BlockingQueue queue = new ArrayBlockingQueue(5);

    private static final ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS,
            queue);


    public static void main(String[] args) {
        ThreadPoolExecutor executor = TestPool.executor;
        for (int i = 0; i < 15; i++) {
            MyTask myTask = new MyTask(i);
            executor.execute(myTask);
            System.out.println("线程池中线程数目：" + executor.getPoolSize() + "，缓存队列中等待执行的任务数目：" +
                    executor.getQueue().size() + "，已执行完的任务数目：" + executor.getCompletedTaskCount());
        }
        executor.shutdown();
    }
}

class MyTask implements Runnable {
    private int taskNum;

    public MyTask(int num) {
        this.taskNum = num;
    }

    @Override
    public void run() {
        System.out.println("正在执行task " + taskNum);
        try {
            Thread.currentThread().sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("task " + taskNum + "执行完毕");
    }
}

```
执行结果：
```
正在执行task 0
线程池中线程数目：1，缓存队列中等待执行的任务数目：0，已执行完的任务数目：0
线程池中线程数目：2，缓存队列中等待执行的任务数目：0，已执行完的任务数目：0
线程池中线程数目：3，缓存队列中等待执行的任务数目：0，已执行完的任务数目：0
正在执行task 1
线程池中线程数目：4，缓存队列中等待执行的任务数目：0，已执行完的任务数目：0
正在执行task 2
正在执行task 3
线程池中线程数目：5，缓存队列中等待执行的任务数目：0，已执行完的任务数目：0
正在执行task 4
线程池中线程数目：5，缓存队列中等待执行的任务数目：1，已执行完的任务数目：0
线程池中线程数目：5，缓存队列中等待执行的任务数目：2，已执行完的任务数目：0
线程池中线程数目：5，缓存队列中等待执行的任务数目：3，已执行完的任务数目：0
线程池中线程数目：5，缓存队列中等待执行的任务数目：4，已执行完的任务数目：0
线程池中线程数目：5，缓存队列中等待执行的任务数目：5，已执行完的任务数目：0
线程池中线程数目：6，缓存队列中等待执行的任务数目：5，已执行完的任务数目：0
线程池中线程数目：7，缓存队列中等待执行的任务数目：5，已执行完的任务数目：0
正在执行task 10
线程池中线程数目：8，缓存队列中等待执行的任务数目：5，已执行完的任务数目：0
正在执行task 11
正在执行task 12
线程池中线程数目：9，缓存队列中等待执行的任务数目：5，已执行完的任务数目：0
正在执行task 13
线程池中线程数目：10，缓存队列中等待执行的任务数目：5，已执行完的任务数目：0
正在执行task 14
task 0执行完毕
task 2执行完毕
task 1执行完毕
正在执行task 7
task 3执行完毕
正在执行task 8
正在执行task 6
正在执行task 5
task 4执行完毕
task 10执行完毕
task 11执行完毕
task 14执行完毕
task 12执行完毕
task 13执行完毕
正在执行task 9
task 7执行完毕
task 6执行完毕
task 5执行完毕
task 8执行完毕
task 9执行完毕
```

从上面的结果可以看出来，当线程池中线程的数目大于5时，便将任务放入任务缓存队列里面，当任务缓存队列满了之后，便创建新的线程。如果上面程序中，将for循环中改成执行20个任务，就会抛出任务拒绝异常了。

例外创建线程的时候建议使用的时``Executors``类提供的方法来创建线程池：
```
Executors.newCachedThreadPool(); //创建一个缓冲池，缓冲池容量大小为Integer.MAX_VALUE
Executors.newSingleThreadExecutor(); //创建容量为1的缓冲池
Executors.newFixedThreadPool(int corePoolSize); //创建固定容量大小的缓冲池，缓存队列大小为Integer.MAX_VALUE
Executors.newScheduledThreadPool(int corePoolSize) //创建一个最大容量为Integer.MAX_VALUE的缓冲池，支持定时及周期性任务执行
```

### 配置线程池的大小
一般需要根据任务的类型来配置线程池大小：

* 如果是CPU密集型任务，就需要尽量压榨CPU，参考值可以设为 NCPU+1；

* 如果是IO密集型任务，参考值可以设置为2*NCPU。

当然，这只是一个参考值，具体的设置还需要根据实际情况进行调整，比如可以先将线程池大小设置为参考值，再观察任务运行情况和系统负载、资源利用率来进行适当调整。

### 总结
1. 当一个task被安排进来的时候，再确定不是空值后，直接判断在池中已经有工作的线程是否小于corePoolSize，小于则增加一个线程来负责这个task。
2.  如果池中已经工作的线程大于等于corePoolSize，就向队列里存task，而不是继续增加线程。
3. 当workQueue.offer失败时，也就是说task不能再向队列里放的时候，而此时工作线程大于等于corePoolSize，那么新进的task，就要新开一个线程来接待了。
4. 线程池工作机制是这样：  
     a.如果正在运行的线程数小于 ``corePoolSize``，那就马上创建线程并运行这个任务，而不会进行排队。  
     b. 如果正在运行的线程数不小于 ``corePoolSize``，那就把这个任务放入队列。  
     c. 如果队列满了，并且正在运行的线程数小于 ``maximumPoolSize``，那么还是要创建线程并运行这个任务。  
     d.如果队列满了，并且正在运行的线程数不小于 ``maximumPoolSize``，那么线程池就会调用handler里方法。(采用``LinkedBlockingDeque``就不会出现队列满情况)。
5. 使用线程池的时候，需要注意先分配好线程池的大小，大约每个线程占用10M内存，就是空间换时间，如果控制的不好，会存在内存溢出的问题，导致机器宕机。

