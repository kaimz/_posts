---
title: Java客户端使用Jedis操作Redis
date: 2017-10-13 15:38:03
tags: [java,redis]
categories: 学习笔记
---

### 写在前面
搭建好redis ，这是我们需要在java中操作它，在这里我使用``jedis``  ，这次主要使用redis，存储信息，到时间超时，并且自动删除超时信息，累计数据List，达到一定数量，入库，删除，所以这个时候为了数据安全，删除完，才去写入新数据，需要写一个简单的分布式锁。

<!--more-->
### 编码
#### 准备，导入Jar包
首先在``pox.xml``加入所需要的Jar 包：

```xml
	<jedis.version>2.9.0</jedis.version>

		<dependency>
		    <groupId>redis.clients</groupId>
		    <artifactId>jedis</artifactId>
		    <version>${jedis.version}</version>
		</dependency>
```


#### 编写连接工具类
首先编写工具类去连接redis：
```java

import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

import java.io.Serializable;

/** 
* @ClassName: RedisUtil 
* @Description: redis工具类 
* @author zhangkai 
* @date 2017年9月26日 下午3:20:29 
*  
*/
public class RedisUtil implements Serializable{
    
    private static final long serialVersionUID = -1149678082569464779L;

    //Redis服务器IP
    private static  String addr;
    
    //Redis的端口号
    private static int port;
    
    //访问密码
    private static String auth;
    
    //可用连接实例的最大数目，默认值为8；
    //如果赋值为-1，则表示不限制；如果pool已经分配了maxActive个jedis实例，则此时pool的状态为exhausted(耗尽)。
    private static int maxActive;
    
    //控制一个pool最多有多少个状态为idle(空闲的)的jedis实例，默认值也是8。
    private static int maxIdle;
    
    //等待可用连接的最大时间，单位毫秒，默认值为-1，表示永不超时。如果超过等待时间，则直接抛出JedisConnectionException；
    private static int maxWait;
    
    @SuppressWarnings("unused")
	private static int timeOut;
    
    //在borrow一个jedis实例时，是否提前进行validate操作；如果为true，则得到的jedis实例均是可用的；
    private static boolean testOnBorrow;
    
    public static Jedis jedis;//非切片额客户端连接
    
    public static JedisPool jedisPool;//非切片连接池
    
   // public static ShardedJedis shardedJedis;//切片额客户端连接
    
   // public static ShardedJedisPool shardedJedisPool;//切片连接池
    
    static {
    	addr = PropertyUtil.get("redis.addr");
    	auth = PropertyUtil.get("redis.auth");
    	port = Integer.parseInt(PropertyUtil.get("redis.port"));
    	maxIdle = Integer.parseInt(PropertyUtil.get("redis.maxIdle"));
    	maxActive = Integer.parseInt(PropertyUtil.get("redis.maxActive"));
    	maxWait = Integer.parseInt(PropertyUtil.get("redis.maxWait"));
    	timeOut = Integer.parseInt(PropertyUtil.get("redis.timeOut"));
    	testOnBorrow = PropertyUtil.get("redis.testOnBorrow").equals("true") ? true :false;
    	initialPool();
    }
    
    public RedisUtil(){
    	initialPool(); 
        jedis = getJedis();
    }
    
    /**
     * 初始化非切片池
     */
    private static void initialPool() {
        // 池基本配置 
        JedisPoolConfig config = new JedisPoolConfig();
        config.setMaxTotal(maxActive); 
        config.setMaxIdle(maxIdle); 
        config.setMaxWaitMillis(maxWait); 
        config.setTestOnBorrow(testOnBorrow);
        jedisPool = new JedisPool(config, addr, port);
    }

    /**
     * 获取Jedis实例
     * @return
     */
    public synchronized static Jedis getJedis() {
        try {
            if (jedisPool != null) {
               jedis = jedisPool.getResource();
               jedis.auth(auth);//认证
               return jedis;
            } else {
                return null;
            }
        } catch (Exception e) {
            Log.error(e);
            return null;
        }
    }
    
    /**
     * 释放jedis资源
     * @param jedis
     */
	public static void returnResource(final Jedis jedis) {
        if (jedis != null) {
        	jedis.close();
        }
    }
}

```
配置文件
```code
# Redis Settings
redis.addr=192.168.19.200
redis.port=6379
redis.auth=master

redis.maxIdle=300
redis.maxActive=1024
redis.maxWait=10000
redis.timeOut=10000
redis.testOnBorrow=false
```
#### 使用
连接上redis我们就可以使用jedis操作我们的redis，直接写业务

##### 登陆，保存会话
```java
@Override
    public void login(String ucid) {
        Jedis jedisindex = getJedis();
        String key = "login" + ucid;
        try {
            //设置登陆时常保存到30m，每次操作都会过来重新存下，重新刷新时间;
            jedisindex.expire(key,  1800);
            
            //TODO code 
            
        } catch (Exception e) {
            LOGGER.error(e.getMessage(), e);
        } finally {
            returnResource(jedisindex);
        }
    }
```
##### 使用redis完成分布式锁
当一个用户满60条数据，进行数据入库，使用分布式锁
```java
   /**
     * xxxxxxx
     *
     * @param key
     * @param track
     * 满到60个 TIDD add
     */
        private void addTrack(String ucid, String track, Jedis jedisindex) {
        try {
            Boolean lockFlag = false;
            while (!lockFlag) {
                lockFlag = lock("lock" + ucid, jedisindex); //上锁
                if (!lockFlag) continue;
                jedisindex.lpush(ucid, track);

                long len = jedisindex.llen(ucid);
                //历史轨迹中总点数
                int pointNum = Integer.valueOf(PropertyUtil.get("POINT_NUM"));
                if (pointNum < 1) return;
                if (len >= pointNum) {
                    addHistoryTrack(ucid, jedisindex.lrange(ucid, 0, pointNum - 1), jedisindex, pointNum);
                }
                unlock("lock" + ucid, jedisindex); //释放锁
            }
        } catch (Exception e) {
            LOGGER.error(e.getMessage(), e);
        }
    }

    
    private static final int LOCK_TIMEOUT = 1; //加锁超时时间 单位秒  意味着加锁期间内执行完操作 如果未完成会有并发现象
    
    
     /**
     * 上锁
     */
    @Override
    public Boolean lock(String lock, Jedis jedisindex) {
        // 1. 通过SETNX试图获取一个lock
        boolean success = false;
        long value = System.currentTimeMillis() + LOCK_TIMEOUT * 1000 + 1;
        long acquired = jedis.setnx(lock, String.valueOf(value));
        jedisindex.expire(lock, LOCK_TIMEOUT);//设置1秒超时 ,到时候自动释放锁
        //SETNX成功，则成功获取一个锁  
        if (acquired == 1) success = true;
        return success;
    }

    /**
     * 解锁
     */
    @Override
    public void unlock(String lock, Jedis jedisindex) {
        jedisindex.del(lock);
    }
```

### 总结：
1. 使用jedis操作redis，使用的是spring 框架，可以使用``Spring Data Redis`` ,更符合java spring框架依赖注入的特性，使用上大同小异。
2. 使用多线程操作redis 不要把 jedis 存入到``ThreadLocal`` 或各种全局变量中， 可能出现冲突。需要重新从``jedisPool``获取``jedis``，然后用完关闭连接就行。

#### 学习：
1. 以后了解对jedis关于事务、管道和分布式的使用。

### 常用命令
```
	1）连接操作命令
    quit：关闭连接（connection）
    auth：简单密码认证
    help cmd： 查看cmd帮助，例如：help quit
    
    2）持久化
    save：将数据同步保存到磁盘
    bgsave：将数据异步保存到磁盘
    lastsave：返回上次成功将数据保存到磁盘的Unix时戳
    shundown：将数据同步保存到磁盘，然后关闭服务
    
    3）远程服务控制
    info：提供服务器的信息和统计
    monitor：实时转储收到的请求
    slaveof：改变复制策略设置
    config：在运行时配置Redis服务器
    
    4）对value操作的命令
    exists(key)：确认一个key是否存在
    del(key)：删除一个key
    type(key)：返回值的类型
    keys(pattern)：返回满足给定pattern的所有key
    randomkey：随机返回key空间的一个
    keyrename(oldname, newname)：重命名key
    dbsize：返回当前数据库中key的数目
    expire：设定一个key的活动时间（s）
    ttl：获得一个key的活动时间
    select(index)：按索引查询
    move(key, dbindex)：移动当前数据库中的key到dbindex数据库
    flushdb：删除当前选择数据库中的所有key
    flushall：删除所有数据库中的所有key
    
    5）String
    set(key, value)：给数据库中名称为key的string赋予值value
    get(key)：返回数据库中名称为key的string的value
    getset(key, value)：给名称为key的string赋予上一次的value
    mget(key1, key2,…, key N)：返回库中多个string的value
    setnx(key, value)：添加string，名称为key，值为value
    setex(key, time, value)：向库中添加string，设定过期时间time
    mset(key N, value N)：批量设置多个string的值
    msetnx(key N, value N)：如果所有名称为key i的string都不存在
    incr(key)：名称为key的string增1操作
    incrby(key, integer)：名称为key的string增加integer
    decr(key)：名称为key的string减1操作
    decrby(key, integer)：名称为key的string减少integer
    append(key, value)：名称为key的string的值附加value
    substr(key, start, end)：返回名称为key的string的value的子串
    
    6）List 
    rpush(key, value)：在名称为key的list尾添加一个值为value的元素
    lpush(key, value)：在名称为key的list头添加一个值为value的 元素
    llen(key)：返回名称为key的list的长度
    lrange(key, start, end)：返回名称为key的list中start至end之间的元素
    ltrim(key, start, end)：截取名称为key的list
    lindex(key, index)：返回名称为key的list中index位置的元素
    lset(key, index, value)：给名称为key的list中index位置的元素赋值
    lrem(key, count, value)：删除count个key的list中值为value的元素
    lpop(key)：返回并删除名称为key的list中的首元素
    rpop(key)：返回并删除名称为key的list中的尾元素
    blpop(key1, key2,… key N, timeout)：lpop命令的block版本。
    brpop(key1, key2,… key N, timeout)：rpop的block版本。
    rpoplpush(srckey, dstkey)：返回并删除名称为srckey的list的尾元素，并将该元素添加到名称为dstkey的list的头部
    
    7）Set
    sadd(key, member)：向名称为key的set中添加元素member
    srem(key, member) ：删除名称为key的set中的元素member
    spop(key) ：随机返回并删除名称为key的set中一个元素
    smove(srckey, dstkey, member) ：移到集合元素
    scard(key) ：返回名称为key的set的基数
    sismember(key, member) ：member是否是名称为key的set的元素
    sinter(key1, key2,…key N) ：求交集
    sinterstore(dstkey, (keys)) ：求交集并将交集保存到dstkey的集合
    sunion(key1, (keys)) ：求并集
    sunionstore(dstkey, (keys)) ：求并集并将并集保存到dstkey的集合
    sdiff(key1, (keys)) ：求差集
    sdiffstore(dstkey, (keys)) ：求差集并将差集保存到dstkey的集合
    smembers(key) ：返回名称为key的set的所有元素
    srandmember(key) ：随机返回名称为key的set的一个元素
    
    8）Hash
    hset(key, field, value)：向名称为key的hash中添加元素field
    hget(key, field)：返回名称为key的hash中field对应的value
    hmget(key, (fields))：返回名称为key的hash中field i对应的value
    hmset(key, (fields))：向名称为key的hash中添加元素field 
    hincrby(key, field, integer)：将名称为key的hash中field的value增加integer
    hexists(key, field)：名称为key的hash中是否存在键为field的域
    hdel(key, field)：删除名称为key的hash中键为field的域
    hlen(key)：返回名称为key的hash中元素个数
    hkeys(key)：返回名称为key的hash中所有键
    hvals(key)：返回名称为key的hash中所有键对应的value
    hgetall(key)：返回名称为key的hash中所有的键（field）及其对应的value
```
