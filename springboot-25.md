---
title: 学习Spring Boot：（二十五）使用 Redis 实现数据缓存
date: 2018-03-17 22:08:08
tags: [Spring Boot,redis]
categories: 学习笔记
---

### 前言
由于 Ehcache 存在于单个 java 程序的进程中，无法满足多个程序分布式的情况，需要将多个服务器的缓存集中起来进行管理，需要一个缓存的寄存器，这里使用的是 Redis。

### 正文
当应用程序要去缓存中读取数据，但是缓存中没有找到该数据，则重新去数据库中获取数据，然后将数据存入缓存中。
还有当我们需要更新或者删除缓存中的数据时候，需要让缓存失效。

![cache](https://zqnight.gitee.io/kaimz.github.io/image/hexo/springboot-cache/3.png)

#### 配置

加入依赖:

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
```

在系统配置文件中加入 redis 的连接参数:

```yaml
spring:
  redis:
    host: 192.168.19.200 # 120.79.208.199 # host ,默认 localhost
    port: 6379 # 端口号，默认6379
    pool:
    # 设置都是默认值，可以按需求设计
      max-active: 8 # 可用连接实例的最大数目，默认值为8；如果赋值为-1，则表示不限制；
      max-idle: 8  # 控制一个pool最多有多少个状态为idle(空闲的)的redis实例，默认值也是8。
      max-wait: -1 # 等待可用连接的最大时间，单位毫秒，默认值为-1，表示永不超时。
      min-idle: 0 # 控制一个pool最少有多少个状态为idle(空闲的)的redis实例，默认值为0。
    timeout: 0 # 连接超时时间 单位 ms，默认为0
    password: master # 密码，根据自己的 redis 设计，默认为空
  cache:
    type: redis # 设置缓存类型
```
然后在系统入口启动类上面加入打开缓存的注解 `@EnableCaching`。
如果没启用其他缓存，这样就自动打开 redis 缓存。

#### 1.5.X 版本

自定义注册 RedisCacheManager，设置相关参数：

```java
    @Bean
    public CacheManager cacheManager(RedisTemplate redisTemplate) {
        RedisCacheManager redisCacheManager = new RedisCacheManager(redisTemplate);
        // 设置缓存最大时间 24 h
        redisCacheManager.setDefaultExpiration(24 * 60 * 60);
        return redisCacheManager;
    }
```
#### 2.0.X 版本

```java
@Configuration
@EnableCaching
public class RedisCacheConfig {
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        // 初始化为默认设置
        RedisCacheManager redisCacheManager = RedisCacheManager.create(factory);
        // 设置缓存最大时间
        /*RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig();  // 生成一个默认配置，通过config对象即可对缓存进行自定义配置
        config = config.entryTtl(Duration.ofMinutes(1))     // 设置缓存的默认过期时间，也是使用Duration设置
                .disableCachingNullValues();     // 不缓存空值
*/
        return redisCacheManager;
    }
}
```



#### 在业务层是用缓存

```java
@Service
@CacheConfig(cacheNames = "em")
public class EmployeeServiceImpl implements EmployeeService {
    @Autowired
    private EmployeeDao dao;

    @Override
    @Cacheable(key = "#p0")
    public Employee findOne(Long id) {
        return dao.findOne(id);
    }

    /**
     * 更新缓存中的数据，
     * 由于 redis 是存在外部，不是 ehcache 那样存在于项目进程中，需要我们主动去更新 缓存
     * @param employee
     * @return
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    @CachePut(key = "#p0.id")
    public Employee update(Employee employee) {
        return dao.save(employee);
    }

    /**
     * 同样主动去删除 cache
     * @param id
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    @CacheEvict(key = "#p0")
    public void delete(Long id) {
        dao.delete(id);
    }
}
```
注解的使用参考前面的[学习Spring Boot：（二十一）使用 EhCache 实现数据缓存](http://blog.wuwii.com/springboot-21.html#注解的使用) 