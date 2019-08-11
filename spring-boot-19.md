---
title: 学习Spring Boot：（十九）Shiro 中使用缓存
date: 2018-02-28 22:39:03
tags: [Spring Boot,shiro]
categories: 学习笔记
---

### 前言
在 shiro 中每次去拦截请求进行权限认证的时候，都会去数据库查询该用户的所有权限信息， 这个时候就是有一个问题了，因为用户的权限信息在短时间内是不可变的，每次查询出来的数据其实都是重复数据，没必要每次都去重新获取这个数据，统一放在缓存中进行管理，这个时候，我们只需要获取一次权限信息，存入到缓存中，待缓存过期后，再次重新获取即可。

<!--more-->

例如，我执行一个查询多次，它执行多次权限查询。
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/springboot-cache/1.png)

### 使用 Reids 缓存
#### 加入 shiro-redis 依赖
```xml
<!-- shiro-redis -->
<dependency>
    <groupId>org.crazycake</groupId>
    <artifactId>shiro-redis</artifactId>
    <version>2.4.2.1-RELEASE</version>
</dependency>
```
#### 配置 Redis
和前面系统配置 Reids 一样。
1. 在系统配置文件中加入 Redis 配置参数：
```yaml
spring:
  redis:
    host: 192.168.19.200 # host ,默认 localhost
    port: 6379 # 端口号，默认6379
    pool:
    # 设置都是默认值，可以按需求设计
      max-active: 8 # 可用连接实例的最大数目，默认值为8；如果赋值为-1，则表示不限制；
      max-idle: 8  # 控制一个pool最多有多少个状态为idle(空闲的)的redis实例，默认值也是8。
      max-wait: -1 # 等待可用连接的最大时间，单位毫秒，默认值为-1，表示永不超时。
      min-idle: 0 # 控制一个pool最少有多少个状态为idle(空闲的)的redis实例，默认值为0。
    timeout: 0 # 连接超时时间 单位 ms，默认为0
    password: master # 密码，根据自己的 redis 设计，默认为空
```
2. 这个我们是要使用 RedisManager 管理我们的 Redis，它默认没有注入我们设置的这些参数，需要我们自己手动创建一个注入我们设置的参数。
```java
@ConfigurationProperties(prefix = "spring.redis")
public class CustomRedisManager extends RedisManager {

}
```
#### 配置 Shiro 缓存
```java
    /**
     * redis 管理
     */
    @Bean
    public CustomRedisManager customRedisManager() {
        return new CustomRedisManager();
    }

    /**
     * redis 缓存
     */
    @Bean
    public RedisCacheManager cacheManager(CustomRedisManager redisManager) {
        RedisCacheManager redisCacheManager = new RedisCacheManager();
        redisCacheManager.setRedisManager(redisManager);
        return redisCacheManager;
    }
    
    @Bean
    public SecurityManager securityManager(OAuth2Realm oAuth2Realm, SessionManager sessionManager, RedisCacheManager cacheManager) {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        // 可以添加多个认证，执行顺序是有影响的
        securityManager.setRealm(oAuth2Realm);
        securityManager.setSessionManager(sessionManager);
        // 注册 缓存
        securityManager.setCacheManager(cacheManager);
        return securityManager;
    }
```
#### 使用测试
我们加入缓存后，看是个什么情况：
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/springboot-cache/2.png)

执行多次请求，只执行了一次查询权限的 SQL。

可以去 redis-cli 查看 keys，检查是否存在权限对象的 key。

### 使用 Ehcache 缓存
#### 加入依赖
```xml
<!-- shiro ehcache -->
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-ehcache</artifactId>
    <version>1.3.2</version>
</dependency>
<!-- ehchache -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```
#### Ehcache 配置
新增一个 Ehcache 配置文件 `shiro-ehcache.xml`：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
    updateCheck="false">
    <diskStore path="java.io.tmpdir/Tmp_EhCache" />
    <defaultCache
        maxElementsInMemory="10000"
        eternal="false"
        timeToIdleSeconds="120"
        timeToLiveSeconds="120"
        overflowToDisk="false"
        diskPersistent="false"
        diskExpiryThreadIntervalSeconds="120" />
    
    <!-- 登录记录缓存锁定1小时 -->
    <cache 
        name="passwordRetryCache"
        maxEntriesLocalHeap="2000"
        eternal="false"
        timeToIdleSeconds="3600"
        timeToLiveSeconds="0"
        overflowToDisk="false"
        statistics="true" />
</ehcache>
```
#### 配置 EhCache 缓存
```java
    /**
     * EhCache 缓存
     */
    @Bean
    public EhCacheManager ehCacheManager() {
        EhCacheManager em = new EhCacheManager();
        em.setCacheManagerConfigFile("classpath:config/shiro-ehcache.xml");
        return em;
    }
    
        @Bean
    public SecurityManager securityManager(OAuth2Realm oAuth2Realm, SessionManager sessionManager,
                                           RedisCacheManager cacheManager, EhCacheManager ehCacheManager) {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        // 可以添加多个认证，执行顺序是有影响的
        securityManager.setRealm(oAuth2Realm);
        securityManager.setSessionManager(sessionManager);
        // 设置缓存
        securityManager.setCacheManager(ehCacheManager);
        return securityManager;
    }
```
### 参考文章
* [Spring Boot Shiro中使用缓存](https://mrbird.cc/Spring-Boot-Shiro%20cache.html)
