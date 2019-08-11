---
title: Java 并发编程同步器 AQS
date: 2018-05-18 11:11:03
tags: [java,并发编程]
categories: 学习笔记
---

### 分析 AQS（队列同步器）

`AbstractQueuedSynchronizer` （AQS），是用来构建所或者其他同步组件的基础框架，它使用一个 int 成员变量来表示同步状态，通过内置的 FIFO 队列来完成资源获取线程的队列工作。

*源码版本  Jdk 1.8*

<!--more-->

#### 怎么实现队列同步器

同步器主要的使用方式是继承，子类实现它的部分方法来管理同步状态变量就可以了。

简单的说，同步器，使用一个状态 `state:int` 表示它的状态变化，如果有其他的锁需要使用 AQS ，需要操作这个状态变量，AQS 直接提供了三个方法供修改状态变量：

```java
    // 获取当前同步资源状态
	protected final long getState() {
        return state;
    }
	// 设置当前同步状态
    protected final void setState(long newState) {
        state = newState;
    }
	// CAS 操作 设置当前状态，该方法保证状态设置的原子性。
    protected final boolean compareAndSetState(long expect, long update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapLong(this, stateOffset, expect, update);
    }
```

同步器是一个 CLH 队列（FIFO），队列中的元素Node就是保存着线程引用和线程状态的容器，每个线程对同步器的访问，都可以看做是队列中的一个节点。

```java
static final class Node {
    int waitStatus;
    Node prev;
    Node next;
    Node nextWaiter;
    Thread thread;
}
```

| 属性名称        | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| int waitStatus  | 表示节点的状态。其中包含的状态有： <br />`CANCELLED`，值为1，表示当前的线程被取消；<br />`SIGNAL`，值为-1，表示当前节点的后继节点包含的线程需要运行，也就是unpark；<br />`CONDITION`，值为-2，表示当前节点在等待condition，也就是在condition队列中；<br />`PROPAGATE`，值为-3，表示当前场景下后续的acquireShared能够得以执行；<br />值为0，表示当前节点在sync队列中，等待着获取锁。 |
| Node prev       | **前驱节点**，比如当前节点被取消，那就需要前驱节点和后继节点来完成连接。 |
| Node next       | **后继节点**。                                               |
| Node nextWaiter | 存储condition队列中的后继节点。                              |
| Thread thread   | 入队列时的当前线程。                                         |

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/java-aqs/1.png)

当前占有资源的节点就是头节点。

AQS 定义两种资源共享方式：

* `Exclusive`：独占模式，又称排他模式，只能有一个线程占用资源，如 ReentrantLock；
* `Share`：共享模式，多个线程可以一起执行，同时占用资源。

同步器对外部使用者提供五个方法，让锁使用资源的方法，主需要实现其中的部分方法，实现对共享资源的获取和释放就可以了。

- `isHeldExclusively()`：该线程是否正在独占资源。只有用到condition才需要去实现它。
- `tryAcquire(int)`：独占方式。尝试获取资源，成功则返回true，失败则返回false。
- `tryRelease(int)`：独占方式。尝试释放资源，成功则返回true，失败则返回false。
- `tryAcquireShared(int)`：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- `tryReleaseShared(int)`：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

#### 独占模式

##### acquire

```java
    // 独占式获取同步状态，实现该方法需要查询当前状态并判断同步状态是否符合预期，然后再进行 CAS 设置同步状态。
	public final void acquire(long arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            // 上面判断如果为 true 的话，表示该等待的线程被中断过，但是等待过程中不能响应中断消息，获取资源后再自我中断，并且释放
            selfInterrupt();
    }

	// 尝试去获取资源状态，如果获取成功，返回 true，这里没有去实现方法，需要锁中自己去实现。
	// 这里没有直接使用抽象方法，因为考虑到独占模式只实现 tryAcquire-tryRelease 这两个方法；而贡献模式只用实现tryAcquireShared-tryReleaseShared 这两个方法，避免都要去实现。
    protected boolean tryAcquire(long arg) {
        throw new UnsupportedOperationException();
    }
	
	// 在上一步中获取线程资源失败后，我们后面要做的就是让该线程加入到等待队列中。
	// 为当前线程创建一个节点，同步器的模式为参数 mode 模式进入等待队列的队尾，并返回构造好的节点。
	// @param mode 指定模式：EXCLUSIVE（独占）和SHARED（共享）
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            // 设置尾节点的时候需要线程安全，需要基于 CAS 操作设置尾节点，只有设置成功后当前节点才能正式与之前的尾节点进行联系。
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 如果队尾为 null 的时候，不能快速的入队操作，将使用 enq
        enq(node);
        return node;
    }

	// 将一个 node 节点 加入队尾，返回上一个尾节点。
    private Node enq(final Node node) {
        // CAS自旋volatile变量
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                // 队尾为 null 的话，证明该队列是空的，需要进行初始化；
                // 初始化一个队列，头节点为空的，尾节点指向头节点。
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                // 直到拿到锁，
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
	// 前面的步骤都是：当线程获取资源失败后，怎么样将线程放入到等待队列的队尾。
	
	// 这一步就是当前线程放入到等待队列后，需要等待其他线程使用完资源释放，自己去获取资源。
	// 如果获取到资源就返回
	// 如果 node 节点的线程线程中断过，就返回 true
    final boolean acquireQueued(final Node node, long arg) {
        boolean failed = true; // 最终是否成功获得资源
        try {
            boolean interrupted = false; // 标记线程是否被中断
            // 自旋锁
            for (;;) {
                final Node p = node.predecessor(); // 获得指定节点的前驱节点 prev
                if (p == head && tryAcquire(arg)) { // 如果该节点的前驱节点是 head 节点，则去尝试获取资源，
                    // 头节点拿到资源
                    setHead(node); // 拿到资源将该 Head 指向该节点，并且将线程的引用置 Null
                    // 由于只有一个线程能够获取到资源，因此设置头节点的时候，不需要 AQS 操作，直接设置即可。
                    p.next = null; // help GC ，拿到资源后，将原头节点从队列中完全拿出来，让系统回收资源
                    failed = false; // 标记已经成功拿到资源
                    return interrupted; 
                }
                // 没有拿到资源，进入等待状态，并且检查是否被中断过。
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true; // 线程中断过
            }
        } finally {
            if (failed)
                cancelAcquire(node); // 出现异常，但没有成功获取获取到资源，取消该线程
        }
    }

	// 主要是去检查前面的线程是否是在等待准备运行，避免已经放弃了的线程节点，去寻找一个安全点（等待状态 waitStatus = 0）
	// 前节点状态是SIGNAL时，当前线程需要阻塞，等待被它唤醒；
	// 前节点状态是CANCELLED时，通过循环将当前节点之前所有取消状态的节点移出队列；
	// 前节点状态是其他状态时，需要设置前节点为SIGNAL。
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;//拿到前驱的状态
        if (ws == Node.SIGNAL)
            //如果已经告诉前驱拿完号后通知自己一下，那就可以安心休息了
            return true;
        if (ws > 0) {
            /*
             * 如果前驱放弃了，那就一直往前找，直到找到最近一个正常等待的状态，并排在它的后边。
             * 注意：那些放弃的结点，由于被自己“加塞”到它们前边，它们相当于形成一个无引用链，稍后就会被保安大叔赶走了(GC回收)！
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
             //如果前驱正常，那就把前驱的状态设置成SIGNAL，告诉它拿完号后通知自己一下。有可能失败，人家说不定刚刚释放完呢！
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
	
    //  上一步执行完毕 如果返回 true，表示自己要安心休息了，就开始执行这个步骤
	// 返回 true ，表示当前线程中断过。
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this); // LockSupport.park 底层实现，让该线程进入等待状态
        return Thread.interrupted(); // 返回检查当前线程是否被中断过
    }
```

总结 `acquire` 流程：

1. 使用 `tryAcquire()`尝试直接去获取资源，如果成功则直接返回；
2. 如果上一步没有成功，`tryAcquire()`尝试直接去获取资源，如果成功则直接返回；
3. `acquireQueued()` 让线程进入等待队列中自旋，当轮到自己去获取资源的时候，采取尝试获取资源，如果被中断过，则返回 `true`，如果返回 `false` 则直接返回；
4. 如果上一步返回 true，表示线程被中断过，但是在等待过程中是不响应的，获取到资源的时候，才去将本线程进行中断。

##### release

```java
    public final boolean release(long arg) {
        if (tryRelease(arg)) { // 尝试去释放节点
            Node h = head; // 找到头节点
            if (h != null && h.waitStatus != 0) // 判断该线程状态，主要是查看后面有没有节点需要它来唤醒的。
                unparkSuccessor(h); // 如果有线程需要它去唤醒，就去唤醒等待队列中的下一个线程
            return true;
        }
        return false;
    }    
	// 注意在独占模式下，这个方法是线程安全的，直接 setState(0) 释放资源即可。
	// 释放完所有的资源（state = 0 ） ，返回  true。
	protected boolean tryRelease(long arg) {
        throw new UnsupportedOperationException();
    }
	// 唤醒等待队列中的最前面节点的状态未取消的线程。
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        // 如果当前等待线程的标志没有取消，则将线程节点的状态置 0，
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0); // 升级为等待状态

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        // 开始去唤醒下一个节点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) { // 判断下个线程为空或者已经取消
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev) // 当下个节点是无效节点的时候，然后从尾节点开始遍历寻找有效节点，作为下个节点
                // 所以这里有个疑问，为什么要从尾节点开始遍历？
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread); // 唤醒下个线程，也只是唤醒一个有效线程状态
        // 既然参与竞争，它是等待队列中排在最前面的等待队列，经过前面的 shouldParkAfterFailedAcquire 调整，一定是 Head 的后继节点，下次自旋的时候，拿到资源的条件成立。
    }
```

这里为什么从开始尾节点遍历，[参考文章](https://my.oschina.net/xianggao/blog/532709)

因为在CLH队列中的结点随时有可能被中断，`被中断的结点的waitStatus设置为CANCEL,而且它会被踢出CLH队列`，如何个踢出法，就是它的前趋结点的next并不会指向它，而是指向下一个非CANCEL的结点,而它自己的next指针指向它自己（将自己踢出，并让 GC 回收）。一旦这种情况发生，如何从头向尾方向寻找继任结点会出现问题，`因为一个CANCEL结点的next为自己，那么就找不到正确的继任接点`。

**总结下 release：**

需要独占模式中自定义的同步器子类去实现，用来释放资源，释放相应的资源，将 state 减少相应的数量即可，如果完全释放了资源，唤醒等待队列中有效的线程来获取资源。

1. 处理当前节点：非CANCELLED状态重置为0；
2. 寻找下个节点：如果是CANCELLED状态，说明节点中途溜了，从队列尾开始寻找排在最前还在等着的节点
3. 唤醒：利用LockSupport.unpark唤醒下个节点里的线程。

##### 总结

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/java-aqs/2.jpg)

1. 获取资源的时候，队列同步器中每个节点都是一个线程在进行自旋，如果该节点的前驱节点是头节点，它就可以去获取资源，退出自旋的时候，将本线程的节点设置成头节点；
2. 释放资源的时候，将本线程的等待状态改成 0 （等待状态），然后让下一个小于 0 的有效节点的节点状态改成 0（等待状态），然后资源状态。
3. 只有当前节点的前一个节点为 `SIGNAL` 时，才能当前节点才能被挂起。对线程的挂起及唤醒操作是通过使用 LockSupport 类的 `park/unpark` 实现的。

##### doAcquireNanos

该方法提供了具备有超时功能的获取状态的调用，如果在指定的`nanosTimeout`内没有获取到状态，那么返回false，反之返回true。可以将该方法看做acquireInterruptibly的升级版，也就是在判断是否被中断的基础上增加了超时控制。
针对超时控制这部分的实现，主要需要计算出睡眠的delta，也就是间隔值。间隔可以表示为 nanosTimeout  等于原有`nanosTimeout – now（当前时间）+ lastTime（睡眠之前记录的时间）`。如果nanosTimeout大于0，那么还需要使当前线程睡眠，反之则返回false。

```java
    // 独占式超时获取资源
	// 在指定时间段内获取资源，成功返回 true 
	private boolean doAcquireNanos(long arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout; // 计算最终超时事件
        final Node node = addWaiter(Node.EXCLUSIVE); // 独占模式加入到等待队列中
        boolean failed = true;
        // 自旋
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                // 没有获取到资源开始工作……
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L) // 超时
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold) // > 1000ns ，超时时间在 1000 ns 内 ，线程自旋中，否则线程进入阻塞状态。
                    // 设置线程还应该睡眠多长时间，避免等待时间过长期间的不断重试。
                    LockSupport.parkNanos(this, nanosTimeout);
                // 中断信号直接中断
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/java-aqs/3.jpg)

#### 共享模式

##### acquireShared

```java
    // 共享模式获取资源，获取成功则返回，获取失败进入同步等待队列中，整个过程忽略线程中断。
	public final void acquireShared(long arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
	// 需要自定义同步器去实现，
	// 负值代表获取失败；
	// 0 代表获取成功，刚好获取剩下的一个资源，没有资源再分配后面的线程；。
	// 正数表示获取成功，返回剩余的资源数量。
    protected long tryAcquireShared(long arg) {
        throw new UnsupportedOperationException();
    }
	// 获取资源失败后，进入等待队列，自旋，直到获取资源才返回。
    private void doAcquireShared(long arg) {
        final Node node = addWaiter(Node.SHARED); // 设置共享模式
        boolean failed = true; // 设置是否获取资源成功标志
        try {
            boolean interrupted = false; // 等待过程中，中断标志
            // 自旋
            for (;;) {
                final Node p = node.predecessor(); //获得前驱节点
                // 如果前驱节点是头节点，则取尝试获取资源
                if (p == head) {
                    long r = tryAcquireShared(arg);
                    if (r >= 0) { // 如果获取资源成功后，就设置头节点为自己（node）
                        setHeadAndPropagate(node, r); 
                        p.next = null; // help GC
                        if (interrupted) // 如果等待过程中出现线程中断就自己中断线程
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                // 判断状态，找到安全点，进入等待状态，等待被唤醒，如果被中断，则补上中断标志
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed) // 出现异常，取消该线程
                cancelAcquire(node);
        }
    }
	// 唤醒符合条件的其他线程
    private void setHeadAndPropagate(Node node, long propagate) {
        Node h = head; // Record old head for check below
        setHead(node); // 将头节点指向 node 节点
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        // 还有资源剩下，则继续唤醒后继节点的线程
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

1. `tryAcquireShared()`尝试获取资源，成功则直接返回；
2. 失败则通过`doAcquireShared()` 进入等待队列 `park()`，直到被 `unpark()` / `interrupt()`并成功获取到资源才返回。整个等待过程也是忽略中断的。

##### releaseShared

```java
    
	public final boolean releaseShared(long arg) {
        if (tryReleaseShared(arg)) { // 尝试释放资源
            doReleaseShared(); // 释放资源成功，唤醒后继节点取竞争资源
            return true;
        }
        return false;
    }
	// 根据作者写这个方法的注释，表达的意思是释放一个或者以上的资源就返回 true，
	// 但是实际情况下，ReentrantReadWriteLock读锁的tryReleaseShared()只有在完全释放掉资源（state=0）才返回true，所以这个可以根据自己的实际情况进行更改的。
    protected boolean tryReleaseShared(long arg) {
        throw new UnsupportedOperationException();
    }

    // 唤醒后继 
    private void doReleaseShared() {
        // 自旋
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) { // 判断后继节点是否需要运行
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) // 将头节点设置成等待状态
                        continue;            // loop to recheck cases
                    unparkSuccessor(h); // 唤醒后继
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) // 如果头节点为等待状态了，则设置节点状态需要往后面的节点传播
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break; 
        }
    }
```



#### Mutex（独占锁）

具体是参考《Java 并发编程艺术》一书第五章：

```java
public class Mutex implements Lock {
    // 自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        /**
         * 判断是否是占有状态
         */
        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        // 尝试获取资源，立即返回。成功则返回true
        @Override
        public boolean tryAcquire(int acquires) {
            assert acquires == 1; // 这里限定只能为1个量
            if (compareAndSetState(0, 1)) {//state为0才设置为1，不可重入！
                setExclusiveOwnerThread(Thread.currentThread());//设置为当前线程独占资源
                return true;
            }
            return false;
        }

        // 尝试释放资源，立即返回。成功则为true，否则false。
        @Override
        protected boolean tryRelease(int releases) {
            assert releases == 1; // 限定为1个量
            if (getState() == 0)
                throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);//释放资源，放弃占有状态
            return true;
        }

        Condition newCondition() {
            return new ConditionObject();
        }
    }

    // 操作全部依赖于AQS自定义的同步器
    private final Sync sync = new Sync();

    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    /**
     * 尝试获取资源，要求立即返回。成功则为true，失败则为false。
     */
    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    /**
     * 释放资源
     */
    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }

    /**
     * 锁是否占有状态
     */
    public boolean isLocked() {
        return sync.isHeldExclusively();
    }
}
```

1. 定义一个静态内部类继承类同步器并实现了独占模式的操作方法；
2. 获取资源 tryAcquire 中，CAS 获取资源，获取成功返回 true；
3. 释放资源 tryRelealse，将资源设置为0；

测试一下（每过一秒打印一个结果）：

```java
public class MutexTest {
    @Test
    public void test() throws InterruptedException {
        Mutex mutex = new Mutex();
        for (int i = 0; i < 5; i++) {
            final int j = i;
            new Thread(() -> {
                try {
                    mutex.lock();
                    TimeUnit.SECONDS.sleep(1);
                    System.out.println(j);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    mutex.unlock();
                }
            }).start();

            TimeUnit.SECONDS.sleep(1);
            System.out.println("------------");
        }
    }
}
```

打印结果：

```java
0
------------
------------
1
------------
2
------------
3
4
------------
```

#### TwinsLock（共享锁）

具体是参考《Java 并发编程艺术》一书第五章：

```java
public class TwinsLock implements Lock {
    /**
     * 自定义的同步器，能够有两个线程同时获取资源
     */
    private final Sync sync = new Sync(2);

    private static final class Sync extends AbstractQueuedSynchronizer {
        Sync(int count) {
            if (count <= 0) {
                throw new IllegalArgumentException("count must large than zero.");
            }
            setState(count);
        }

        @Override
        public int tryAcquireShared(int reduceCount) {
            for (; ; ) {
                int current = getState();
                int newCount = current - reduceCount;
                if (newCount < 0 || compareAndSetState(current, newCount)) {
                    return newCount;
                }
            }
        }

        @Override
        public boolean tryReleaseShared(int returnCount) {
            for (; ; ) {
                int current = getState();
                int newCount = current + returnCount;
                if (compareAndSetState(current, newCount)) { // 将释放的资源返回
                    return true;
                }
            }
        }

        Condition newCondtion() {
            return new ConditionObject();
        }
    }

    @Override
    public void lock() {
        sync.acquireShared(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquireShared(1) >= 0;
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.releaseShared(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondtion();
    }
}
```

TwinsLock 包含了一个自定义的同步器 sync，该同步器以共享方式获取同步状态。当 消耗资源`tryAcquireShared(int reduceCount)` 大于 或者等于 0 的时候，表示当前线程获取锁成功。

验证上面锁的正确性，就是要验证是否同一时刻有两个线程同时进行打印任务。

```java
public class TwinsLockTest {
    @Test
    public void test1() throws InterruptedException {
        final Lock lock = new TwinsLock();

        class Worker extends Thread {
            @Override
            public void run() {
                try {
                    lock.lock();
                    TimeUnit.SECONDS.sleep(1);
                    System.out.println(Thread.currentThread().getName());
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }

        for (int i = 0; i < 10; i++) {
            new Worker().start();
        }

        for (int i = 0; i < 10; i++) {
            TimeUnit.SECONDS.sleep(1);
            System.out.println();
        }
    }
}
```

打印结果：

```java

Thread-5
Thread-2

Thread-1
Thread-6


Thread-0
Thread-9



Thread-4
Thread-3

Thread-7
Thread-8
```



#### 总结

在这个 AQS 同步器中我们时刻需要更改注意两个方面的问题：

- 一是要去维护同步队列，更改同步器的资源状态变量，通过 `Unsafe` 提供原子操作 CAS；

- 二是底层还要去根据同步器状态变量去实现线程等待，线程唤醒的，它是通过 `LockSupport`  的 `park/unpark` 操作。
  当然使用者不需要注意这些问题，代码已经把这些方法都已经封装好了，只要实现资源变量的变化的几个方法就可以了。

- 如果是要使用独占模式，只需要实现 

  ```java
  protected boolean tryAcquire(int arg)
  protected boolean tryRelease(int arg)
  ```

- 如果只是要使用共享模式，需要实现

  ```java
  protected int tryAcquireShared(int arg)
  protected boolean tryReleaseShared(int arg)
  ```

#### 补充：CLH 队列锁

就是源码分析中那个同步队列，节点单位就是内部类 Node，获取锁的时候跟队列的头有关，释放锁主要删除头节点和从尾节点唤醒。它虽然保留了自旋操作，但是真实情况下，是阻塞了线程（LockSupport）。

#### 补充：CAS

这部分参考文章 [认识非阻塞的同步机制CAS](https://www.jianshu.com/p/e2179c74a2e4?utm_campaign=maleskine&utm_content=note&utm_medium=pc_all_hots&utm_source=recommendation)

> CAS 操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作。无论V值是否等于A值，都将返回V的原值。

当多个线程尝试使用CAS同时更新一个变量，最终只有一个线程会成功，其他线程都会失败。但和使用锁不同，失败的线程不会被阻塞，而是被告之本次更新操作失败了，可以再试一次。此时，线程可以根据实际情况，继续重试或者跳过操作，大大减少因为阻塞而损失的性能。所以，CAS是一种乐观的操作，它希望每次都能成功地执行更新操作。

**AQS 中的 CAS 由 `Unsafe` 提供。**

#### 补充：自旋锁

这部分参考文章 [CAS和自旋锁(spin lock)](http://www.cnblogs.com/thomaschen750215/p/4122068.html)

>  由于在多处理器系统环境中有些资源因为其有限性，有时需要互斥访问（mutual exclusion），这时会引入锁的机制，只有获取了锁的进程才能获取资源访问。即是每次只能有且只有一个进程能获取锁，才能进入自己的临界区，同一时间不能两个或两个以上进程进入临界区，当退出临界区时释放锁。设计互斥算法时总是会面临一种情况，即没有获得锁的进程怎么办？通常有2种处理方式。一种是没有获得锁的调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，这就是自旋锁，他不用将线城阻塞起来（NON-BLOCKING)；另一种是没有获得锁的进程就阻塞(BLOCKING)自己，请求OS调度另一个线程上处理器，这就是互斥锁。

跟互斥锁一样，一个执行单元要想访问被自旋锁保护的共享资源，必须先得到锁，在访问完共享资源后，必须释放锁。如果在获取自旋锁时，没有任何执行单元保持该锁，那么将立即得到锁；如果在获取自旋锁时锁已经有保持者，那么获取锁操作将自旋在那里，直到该自旋锁的保持者释放了锁。由此我们可以看出，自旋锁是一种比较低级的保护数据结构或代码片段的原始方式，这种锁可能存在两个问题：

- **递归死锁**：试图递归地获得自旋锁必然会引起死锁：递归程序的持有实例在第二个实例循环，以试图获得相同自旋锁时，不会释放此自旋锁。在递归程序中使用自旋锁应遵守下列策略：递归程序决不能在持有自旋锁时调用它自己，也决不能在递归调用时试图获得相同的自旋锁。此外如果一个进程已经将资源锁定，那么，即使其它申请这个资源的进程不停地疯狂“自旋”,也无法获得资源，从而进入死循环。
- **过多占用cpu资源**。如果不加限制，由于申请者一直在循环等待，因此自旋锁在锁定的时候,如果不成功,不会睡眠,会持续的尝试,单cpu的时候自旋锁会让其它process动不了. 因此，一般自旋锁实现会有一个参数限定最多持续尝试次数. 超出后, 自旋锁放弃当前time slice. 等下一次机会

​     由此可见，自旋锁比较适用于锁使用者保持锁时间比较短的情况。正是由于自旋锁使用者一般保持锁时间非常短，因此选择自旋而不是睡眠是非常必要的，自旋锁的效率远高于互斥锁。

#### 补充：LockSupport

具体内容请参考 [LockSupport（park/unpark）源码分析](https://www.jianshu.com/p/e3afe8ab8364)

`LockSupport.park() `和 `LockSupport.unpark(Thread thread)`调用的是 `Unsafe`中本地方法：

```java
    //park
    public native void park(boolean isAbsolute, long time);
    
    //unpack
    public native void unpark(Object var1);
```

`park` 函数是将当前调用Thread阻塞，而 `unpark` 函数则是将指定线程唤醒。

### ReentrantLock(独占锁)

https://www.jianshu.com/p/fe027772e156

其实 ReentantLock 的实现和上面的例子 `Mutex` 的差不多，不过它另外实现了可重入和公平锁两个部分。

#### 公平锁

```java
protected final boolean tryAcquire(int acquires) {
   final Thread current = Thread.currentThread();
   int c = getState();
   if (c == 0) {
       if (!hasQueuedPredecessors() &&
           compareAndSetState(0, acquires)) {
           setExclusiveOwnerThread(current);
           return true;
       }
   }
   else if (current == getExclusiveOwnerThread()) {
       int nextc = c + acquires;
       if (nextc < 0)
           throw new Error("Maximum lock count exceeded");
       setState(nextc);
       return true;
   }
   return false;
}
```

相比非公平锁，它多了一个方法`hasQueuedPredecessors` 判断队列是否有排在前面的线程在等待锁，没有的话调用`compareAndSetState` 使用 CAS 的方式修改state，然后设置本线程为独占锁，并且它是可重入锁。

### ReentrantReadWriteLock（读写锁）

首先来分析下读写锁的几个重要的特点：

1. 读写状态的设计；
2. 写锁的获取与释放；
3. 读锁的获取与释放；
4. 锁降级的实现。

#### 构造方法

```java
    public ReentrantReadWriteLock() {
        this(false);
    }	
	// 提供公平锁，默认是非公平锁    
	public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```

#### 读写状态设计

在读写锁中最重要的就是Sync类，它继承自AQS，还记得吗，AQS使用一个int型来保存状态，状态在这里就代表锁，它提供了获取和修改状态的方法。可是，这里要实现读锁和写锁，只有一个状态怎么办？Doug Lea是这么做的，它把状态的高16位用作读锁，低16位用作写锁，所以无论是读锁还是写锁最多只能被持有65535次。所以在判断读锁和写锁的时候，需要进行位运算：

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/java-aqs/4.jpg)

1. 由于读写锁共享状态，所以状态不为0，只能说明是有锁，可能是读锁，也可能是写锁；
2. 读锁是高16为表示的，所以读锁加1，就是状态的高16位加1，低16位不变，所以要加的不是1，而是2^16，减一同样是这样。
3. 写锁用低16位表示，要获得写锁的次数，要用状态&2^16-1，结果的高16位全为0，低16位就是写锁被持有的次数。



那它是怎么确定读写的各自的状态的了，是通过位运算符，假设当前同步状态值为 S，写状态等于 `S & 0x0000FFFF `（只有 低 16 位），读状态等于 `S  >>> 16` （无符号补位右移 16 位）；这个时候写状态增加 1 时，等于 `S + 1`，当读状态增加 1，等于 `S + (1 << 16)`，也就是 ` S + 0x00010000`。

根据状态的划分可以得出一个结论： S 不等于 0 时，就是写状态计算公式 `S & 0x0000FFFF == 0 `，则读状态 `S  >>> 16 > 0` 这个时候，读锁获取到了。

#### 同步器的设计

```java
   //ReentrantReadWriteLock的同步器
	// 分别用子类来实现公平和非公平策略  
   abstract static class Sync extends AbstractQueuedSynchronizer {  
       private static final long serialVersionUID = 6317671515068378041L;  
  
       //高16位表示持有读锁的计数 
       static final int SHARED_SHIFT   = 16;  
       //由于读锁用高位部分，读锁个数加1，其实是状态值加 2^16  
       static final int SHARED_UNIT    = (1 << SHARED_SHIFT); 
       // 所以读锁或者写锁分别最多线程数为 2^16 = 65535
       static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;  
       // 低16位表示写锁计数，
       // 写锁的掩码，用于状态的低16位有效值 
       static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;  
  
       // 获取读锁（共享锁）的数量
       static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }  
       // 获取写锁（独占锁）的重入次数
       static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }  
  
       /** 
        * 每个线程特定的 read 持有计数。存放在ThreadLocal，不需要是线程安全的。
        */  
       static final class HoldCounter {  
           int count = 0;  
           //使用id而不是引用是为了避免保留垃圾。注意这是个常量。  
           final long tid = Thread.currentThread().getId();  
       }  
  
       /** 
        * 采用继承是为了重写 initialValue 方法，这样就不用进行这样的处理： 
        * 如果ThreadLocal没有当前线程的计数，则new一个，再放进ThreadLocal里。 
        * 可以直接调用 get。 
        * */  
       static final class ThreadLocalHoldCounter  
           extends ThreadLocal<HoldCounter> {  
           public HoldCounter initialValue() {  
               return new HoldCounter();  
           }  
       }  
       /** 
        * 当前线程持有的可重入读锁的数量，仅在构造方法和readObject(反序列化) 
        * 时被初始化，当持有锁的数量为0时，移除此对象。 
        * 它存储了当前线程的 HoldCounter ，而HoldCounter中的count变量就是用来记录线程获得的写锁的个数。
        */  
       private transient ThreadLocalHoldCounter readHolds;  
  
       /** 
        * 最近一个成功获取读锁的线程的计数。这省却了ThreadLocal查找， 
        * 通常情况下，下一个释放线程是最后一个获取线程。这不是 volatile 的， 
        * 因为它仅用于试探的，线程进行缓存也是可以的 
        * （因为判断是否是当前线程是通过线程id来比较的）。 
        */  
       private transient HoldCounter cachedHoldCounter;  
      
       /**
       	* firstReader是第一个获得读锁的线程； 
        * firstReaderHoldCount是firstReader的重入计数； 
        * 更准确的说，firstReader是最后一个把共享计数从0改为1，并且还没有释放锁。 
        * 如果没有这样的线程，firstReader为null; 
        * firstReader不会导致垃圾堆积，因为在tryReleaseShared中将它置空了，除非 
        * 线程异常终止，没有释放读锁。 
        *  
        * 跟踪无竞争的读锁计数时，代价很低 
        */  
       private transient Thread firstReader = null;  
       private transient int firstReaderHoldCount;  
  
       Sync() {  
           readHolds = new ThreadLocalHoldCounter();  
           setState(getState()); // ensures visibility of readHolds  
       }  
       // 下面两个抽象方法用来实现读锁或者写锁是否需要阻塞。
		abstract boolean writerShouldBlock();//写锁是否需要阻塞
		abstract boolean readerShouldBlock();//读锁是否需要阻塞
       
    // 下面是非公平锁的实现
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        // 持有写锁可重入，不需要阻塞。
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
        // 根据下一个节点是不是写锁（独占锁）确定它是否阻塞
        final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
            return apparentlyFirstQueuedIsExclusive();
        }
    }
    //  AQS 提供的方法，判断下一个节点是独占锁
    final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
   // 公平锁的实现，只要当前线程前面还有线程需要获取锁，都要进行阻塞。    
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }
    // AQS 提供的方法，和 ReentrantLock 里面同步锁实现一样的，判断该线程前面有没有线程在等待获取锁   
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

#### 写锁的获取和释放

首先，写锁是一个可重入的排他锁，如果当前线程获取到了写锁，则增加写状态，如果读锁或者写锁已经被获取了，它则进入等待状态（写锁要确保写锁的操作对读锁可见）。

写锁和对外提供的方法和 ReentrantLock 一样的，这里主要去分析下它是怎么获取和释放资源的。

```java
        @ReservedStackAccess
        protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively()) // 检查当前线程是不是独占模式
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases; // 计算释放后的资源数量
            boolean free = exclusiveCount(nextc) == 0; 
            if (free)
                setExclusiveOwnerThread(null); // 如果写锁都执行完了，释放写锁。
            setState(nextc); // 写入释放完资源的数量
            return free;
        }

        @ReservedStackAccess
        protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c); // 获取写锁的数量
            if (c != 0) { // 如果 state 不为 0 表示锁已经分配出去了
                // (Note: if c != 0 and w == 0 then shared count != 0)
                if (w == 0 || current != getExclusiveOwnerThread()) //如果其他线程获取了写锁则获取失败
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires); // 经过上面的检查后，表示写锁可以重入，并返回 true
                return true;
            }
            // 对于要首次获取写锁，如果允许获取写锁， CAS 操作 获取独占锁
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

获取锁：
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/java-aqs/5.png)

获取锁的过程：

1. 首先获取写锁的数量；
2. 判断写锁是否已经被获取了，如果已经获取了，就要做重入操作，将锁的资源数量加一，然后返回；如果是首次获取，就要进行 CAS 操作获取独占锁；

释放锁的过程：

1. 计算如果释放完资源的数量；
2. 如果剩下的资源数量为 0，则释放写锁；
3. 如果剩下的资源数量不为 0，就将计算完的资源数量写入。

#### 读锁的获取和释放

读锁是一个支持**重入**的**共享锁**，它能够被多个线程同时获取，当写锁的状态为 0 的时候，读锁总是获取成功的，并且增加读状态。这里比较复杂些。

```java
        @ReservedStackAccess
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            // 先处理本地本地计数器
            // 判断当前线程是否为第一个读线程firstReader
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                // 如果当前线程读锁重入次数为 1，再去释放这个线程，firstReader置空 ，否则减去重入次数
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                // 若当前线程不是第一个读线程，那么首先会获取缓存计数器
                // 注：深层次去分析了源码发现 ：readHolds存储了每一个线程的HoldCounter，而HoldCounter中的count变量就是用来记录线程获得的写锁的个数。
                HoldCounter rh = cachedHoldCounter;
                if (rh == null ||
                    rh.tid != LockSupport.getThreadId(current)) // 如果计数器为空或者 tid 不等于当前线程的 tid，则获取缓存计数器，
                    rh = readHolds.get();
                int count = rh.count;
                // 如果当前线程的计数器数量小于或者等于 1 的时候，移除当前线程的计数器，
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0) // 如果小于 1 的时候则抛出异常，
                        throw unmatchedUnlockException();
                }
                // 如果正常获取到了当前线程计数器，则将计数数量减一
                --rh.count;
            }
            // 当其他线程读锁也在释放读锁，AQS 可能失败所以自旋重试
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT; // 高 16 位 减一
                if (compareAndSetState(c, nextc)) // AQS
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }

        private IllegalMonitorStateException unmatchedUnlockException() {
            return new IllegalMonitorStateException(
                "attempt to unlock read lock, not locked by current thread");
        }

        @ReservedStackAccess
        protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current) // 当没有线程持有写锁的时候就可以获取读锁
                return -1; // 获取读锁失败后，线程进入等待队列
            int r = sharedCount(c); // 获取读锁的数量
            if (!readerShouldBlock() && // 通过公平的策略判断，如果读锁不用阻塞
                r < MAX_COUNT &&  // 读锁数量没有超出上限
                compareAndSetState(c, c + SHARED_UNIT)) { // 就去将读锁的资源数量加一，这个时候注意的是，由于读锁在高 16 位上。
                // 经过上一步，已经成功获取到读锁，后面进行相关设置
                if (r == 0) {
                    firstReader = current; // 如果它是第一个获取到读锁的线程，则将 firstReader 指向它，并且计数读锁firstReader重入次数为1
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++; // 重入次数加一
                } else {
                    // 对于 不是 firstReader 读锁计数更新，更新当前线程的缓存
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null ||
                        rh.tid != LockSupport.getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            // 获取读锁失败（阻塞或者数量上限或者 AQS 设置失败），重试，跟 tryAcquireShared 差不多的逻辑。
            return fullTryAcquireShared(current);
        }

```

阅读读锁的时候需要注意`firstReader `，和 `HoldCounter ` 这两个变量的变化就可以了：

1. 如果读锁没有被持有，那么每一个线程的 `HoldCounter `  变量中的 count 变量一定是为 0；
2. 如果当前线程是第一个获取到读锁的线程，设置 ``firstReader `` 为当前线程，并且设置 `firstReadHoldCount  `数量；
3. 那么如果当前线程不是第一个获取读锁的线程，那么获取当前线程的 `HoldCounter `，获取 `count` 的值，判断它等不等于 0 ，如果等于 0 的话，表示当前线程没有获取读锁，那么可以从 `readHolds` 的管理中将它移除，

#### 锁降级

**锁降级是指把持住（当前拥有的）写锁，再获取到读锁，随后释放（先前拥有的）写锁的过程。**

### 总结
主要是分析 AQS 源码的实现，了解到所有的同步类都是实现自定义的同步器 `sync` ，实现独占方法或者共享方法中的获取资源和释放资源方法供自己使用，同步器只要关注资源变量 state 的变化，对使用者非常友好，层次分明，而不需要关注队列和线程的阻塞的情况。

### 参考文章
* 《Java 并发编程的艺术》
* [AbstractQueuedSynchronizer详解](http://luojinping.com/2015/06/19/AbstractQueuedSynchronizer%E8%AF%A6%E8%A7%A3/)
* [Java并发之AQS详解](http://www.cnblogs.com/waterystone/p/4920797.html)
* [认识非阻塞的同步机制CAS](https://www.jianshu.com/p/e2179c74a2e4?utm_campaign=maleskine&utm_content=note&utm_medium=pc_all_hots&utm_source=recommendation)
* [CAS和自旋锁(spin lock)](http://www.cnblogs.com/thomaschen750215/p/4122068.html)
* [LockSupport（park/unpark）源码分析](https://www.jianshu.com/p/e3afe8ab8364)
* [Java并发-ReentrantReadWriteLock源码分析](https://blog.csdn.net/yuhongye111/article/details/39055531)
