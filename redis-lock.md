---
title: 使用Redis完成分布式锁
date: 2017-11-20 22:08:03
tags: [java,redis]
categories: 学习笔记
---

### 实现原理
>分布式的CAP理论告诉我们“任何一个分布式系统都无法同时满足一致性（Consistency）、可用性（Availability）和分区容错性（Partition tolerance），最多只能同时满足两项。”所以，很多系统在设计之初就要对这三者做出取舍。在互联网领域的绝大多数的场景中，都需要牺牲强一致性来换取系统的高可用性，系统往往只需要保证“最终一致性”，只要这个最终时间是在用户可以接受的范围内即可。

为了保证数据的最终一致性，需要很多的技术方案来支持，比如分布式事务、分布式锁等。
#### 使用Redis实现锁的原因
1. Redis有很高的性能；
2. Redis命令对此支持较好，实现起来比较方便。

<!--more-->

#### 主要利用到的命令
##### SETNX
>SETNX key val
当且仅当key不存在时，set一个key为val的字符串，返回1；若key存在，则什么都不做，返回0。

##### expire
expire key timeout
为key设置一个超时时间，单位为second，超过这个时间锁会自动释放，避免死锁。

##### delete
delete key
删除key

#### 实现思想
* 获取锁的时候，使用setnx加锁，并使用expire命令为锁添加一个超时时间，超过该时间则自动释放锁，保证key一致，通过此在释放锁的时候进行判断。
* 获取锁的时候还设置一个获取的超时时间，若超过这个时间则放弃获取锁。
* 释放锁的时候，当前时间小于超时时间，则执行delete进行锁释放。


### 代码结构
```java
package com.devframe.util;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import redis.clients.jedis.Jedis;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

/**
 * <b>redis分布式锁的实现</b></br>
 * 还有一些失败机制没处理，以后在使用测试阶段，完善。
 *
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/20 9:22</pre>
 */
public class RedisLock implements Lock {
    private final static Logger logger = LoggerFactory.getLogger(RedisLock.class);
    /**
     * redis连接
     */
    private final Jedis jedis;
    /**
     * 锁定资源名，锁key，保证唯一。
     */
    private final String lockName;
    /**
     * 资源上锁的最长时间，超时自动解锁单位秒，</br>
     * 建议设置成死的，如果设置不当容易影响效率，严重造成死锁。
     */
    private final int expireTime = Integer.valueOf(PropertyUtil.get("redisLock.expireTime"));
    /**
     * 线程获取不到锁，休眠的时间，单位ms
     * 避免系统资源浪费
     */
    private final long sleepTime = Long.valueOf(PropertyUtil.get("redisLock.sleepTime"));
    /**
     * 当前锁超时的时间戳，单位毫秒
     */
    private long expireTimeOut = 0;
    /**
     * 获取锁状态，锁中断状态</br>
     * 值为false的时候中断获取锁</br>
     */
    private boolean interrupted = true;


    /**
     * 构造方法
     *
     * @param jedis    redis连接
     * @param lockName 上锁key，唯一标识
     */
    public RedisLock(Jedis jedis, String lockName) {
        if (lockName == null) {
            throw new NullPointerException("lockName is required");
        }
        this.jedis = jedis;
        // 重命名的前缀，可以不加，也可以自定义，保证唯一即可。
        this.lockName = "lock" + lockName;
    }

    /**
     * 获取锁。如果锁已被其他线程获取，则进行等待，直到拿到锁为止。
     */
    @Override
    public void lock() {
        while (true) {
            this.lockCheck();
            long id = jedis.setnx(lockName, lockName);
            if (id == 0L) {
                try {
                    /**
                     * 没有获取到锁则进行等待睡眠时间，再去重新获取锁</br>
                     * 这里使用随机时间可能会好一点,可以防止饥饿进程的出现,即,当同时到达多个进程,
                     * 只会有一个进程获得锁,其他的都用同样的频率进行尝试,后面有来了一些进行,
                     * 也以同样的频率申请锁,这将可能导致前面来的锁得不到满足.
                     * 使用随机的等待时间可以一定程度上保证公平性
                     */
                    Thread.sleep(this.sleepTime);
                } catch (InterruptedException e) {
                    logger.error("Thread is interrupted", e);
                }
            } else {
                expireTimeOut = System.currentTimeMillis() + expireTimeOut * 1000 + 1;
                //设置redis中key的过期时间
                jedis.expire(this.lockName, expireTime);
                break;
            }
        }
    }

    /**
     * 中断锁获取
     *
     * @throws InterruptedException 中断异常
     */
    @Override
    public void lockInterruptibly() throws InterruptedException {
        this.interrupted = false;
    }

    /**
     * 它表示用来尝试获取锁，会立即返回，如果获取成功，则返回true，如果获取失败（即锁已被其他线程获取），则返回false，</br>
     * 也就说这个方法无论如何都会立即返回。在拿不到锁时不会一直在那等待。
     *
     * @return boolean
     */
    @Override
    public boolean tryLock() {
        this.lockCheck();
        //尝试获取锁
        long id = jedis.setnx(lockName, lockName);
        //返回结果为0 则已经存在key，已经存在锁。
        if (id == 0L) {
            return false;
        } else {
            expireTimeOut = System.currentTimeMillis() + expireTimeOut * 1000 + 1;
            //设置redis中key的过期时间
            jedis.expire(this.lockName, expireTime);
            return true;
        }
    }

    /**
     * 它表示用来尝试获取锁，如果获取成功，则返回true，如果获取失败（即锁已被其他线程获取），则返回false，</br>
     * 这个方法在拿不到锁时会等待一定的时间，在时间期限之内如果还拿不到锁，就返回false。</br>
     * 如果如果一开始拿到锁或者在等待期间内拿到了锁，则返回true。</br>
     *
     * @param time 等待时间
     * @param unit 时间单位
     * @return boolean
     * @throws InterruptedException 中断异常
     */
    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        if (time == 0) {
            return false;
        }
        if (unit == null) {
            throw new NullPointerException("TimeUnit is required.");
        }
        long now = System.currentTimeMillis();
        long timeOutAt = now + calcSeconds(time, unit);
        while (true) {
            this.lockCheck();
            long id = jedis.setnx(this.lockName, this.lockName);
            // id = 0 表示加锁失败
            if (id == 0) {
                // 获取锁超时
                if (System.currentTimeMillis() > timeOutAt) {
                    return false;
                }
                // 休眠一段时间，线程再继续获取锁。
                Thread.sleep(this.sleepTime);
            } else {
                //获取锁成功，设置锁过期时间戳
                expireTimeOut = System.currentTimeMillis() + expireTimeOut * 1000 + 1;
                jedis.expireAt(this.lockName, expireTimeOut);
                return true;
            }
        }
    }

    /**
     * <b>释放锁<b/>
     * 当前时间小于过期时间，则锁未超时，删除锁，</br>
     * 过了超时时间，redis已经删除了该key。
     */
    @Override
    public void unlock() {
        if (System.currentTimeMillis() < expireTimeOut) {
            jedis.del(lockName);
        }
    }

    @Override
    public Condition newCondition() {
        //TODO 涉及到 Condition 例外一个重要内容，以后再实现这个方法
        throw new UnsupportedOperationException("did not supported.");
    }

    /**
     * 检查当前线程资源redis连接和锁的状态
     */
    private void lockCheck() {
        if (jedis == null) {
            throw new NullPointerException("Jedis is required.");
        }
        if (!interrupted) {
            throw new RuntimeException("Thread is interrupted.");
        }
    }

    /**
     * TimeUnit单位时间转换成毫秒
     *
     * @param time 时间
     * @param unit 时间单位
     * @return long
     */
    private long calcSeconds(long time, TimeUnit unit) {
        if (unit == TimeUnit.DAYS) {
            return time * 24 * 60 * 60 * 1000;
        }
        if (unit == TimeUnit.HOURS) {
            return time * 60 * 60 * 1000;
        }
        if (unit == TimeUnit.MINUTES) {
            return time * 60 * 1000;
        }
        if (unit == TimeUnit.SECONDS) {
            return time * 1000;
        }
        if (unit == TimeUnit.MILLISECONDS) {
            return time;
        } else {
            //后面的不实现了，基本上用不到。
            throw new UnsupportedOperationException("cannot be resolved.");
        }
    }
}

```

配置
```
# redis lock
# s
redisLock.expireTime=1
# ms
redisLock.sleepTime=100
```
### 测试
测试就选用最经典的秒杀系统吧，使用分布式锁可以控制资源。

下面模拟500人秒杀100件商品。
```java
package com.devframe.util;

import org.junit.Test;
import redis.clients.jedis.Jedis;

/**
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/20 14:12</pre>
 */
public class RedisLockTest {
    /**
     * 100件物品
     */
    public static int goodsNum = 100;
    /**
     * 500人
     */
    private static int personNum = 500;

    /**
     * 不加锁的情况
     */
    @Test
    public void test() {
        for (int i = 0; i < personNum; i++) {
            new Thread(() -> {
                if (goodsNum > 0) {
                    System.out.println(Thread.currentThread().getName() + "获取了锁");
                    System.out.println("商品剩余：" + --goodsNum);
                }
            }).start();
        }

    }

    /**
     * 加上分布锁
     * @param args
     */
    public static void main(String[] args) {
        for (int i = 0; i < personNum; i++) {
            new Thread(() -> {
                Jedis jedis = RedisUtil.getJedis();
                //初始化锁，key保持一致
                Lock lock = new RedisLock(jedis, "aa");
                try {
                    lock.lock();
                    if (goodsNum > 0) {
                        System.out.println(Thread.currentThread().getName() + "获取了锁");
                        System.out.println("商品剩余：" + --goodsNum);
                    }
                } finally {
                    //释放锁，并且释放redis连接
                    lock.unlock();
                    RedisUtil.returnResource(jedis);
					
                }
            }).start();
        }
    }
}

```
不加锁的部分结果：
```
Thread-100获取了锁
商品剩余：-3
Thread-99获取了锁
商品剩余：5
商品剩余：6
Thread-98获取了锁
商品剩余：-5
商品剩余：7
商品剩余：-4
商品剩余：0
商品剩余：1
Thread-105获取了锁
商品剩余：-6
```
上锁的结果：
```
Thread-8获取了锁
商品剩余：5
Thread-238获取了锁
商品剩余：4
Thread-72获取了锁
商品剩余：3
Thread-137获取了锁
商品剩余：2
Thread-402获取了锁
商品剩余：1
Thread-337获取了锁
商品剩余：0
```

### 总结
1. 并发量大的时候，需要考虑锁时间；
2. 考虑失败情况，上锁了，但是设置超时时间失败（redis崩溃等各种情况），锁一致都没有释放，导致死锁的情况发生，现在需要做的是，把key的value设置成超时的时间，每次上锁失败都去检查一次，超时的就覆盖，可以避免死锁。
