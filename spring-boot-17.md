---
title: 学习Spring Boot：（十七）使用 Redis
date: 2018-02-27 22:38:03
tags: [Spring Boot,redis]
categories: 学习笔记
---

### 前言
>Redis [^1] 是一个由Salvatore Sanfilippo写的key-value存储系统。  
>edis是一个开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。  
>通常被称为数据结构服务器，因为值（value）可以是 字符串(String), 哈希(Map), 列表(list), 集合(sets) 和 有序集合(sorted sets)等类型。

<!--more-->

* [Redis官网](http://redis.io/)
* [Redis中文社区](http://www.redis.cn/)

[^1]: REmote DIctionary Server

### 正文
#### 引入依赖
```xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>
```
>需要注意的是，上面是 `Spring Boot 1.5` 版本后的名称，1.5版本前是 `spring-boot-starter-redis`。

#### 参数配置
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

#### 使用
在 配置类 中注册一个 `RedisTemplate` 用来支持序列化和反序列化:
```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory factory) {
        StringRedisTemplate template = new StringRedisTemplate(factory);
        // 使用 Jackson2JsonRedisSerializer 进行序列化，它继承 RedisSerializer，
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        template.setValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }

}
```
测试使用：
```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class RedisTest {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    @Autowired
    private RedisTemplate redisTemplate;

    @Test
    public void test() {
        stringRedisTemplate.opsForValue().set("id", "1");
        Assert.assertEquals("1", stringRedisTemplate.opsForValue().get("id"));
    }

    /**
     * 测试存储对象，redis 需要对对象进行序列化，取出对象数据后比对，又要进行反序列化
     * 所以注册了 RedisTemplate ，专门处理这类情况
     */
    @Test
    public void test1() {
        SysUserEntity sysUserEntity = new SysUserEntity();
        sysUserEntity.setId(2L);
        sysUserEntity.setEmail("k@wuwii.com");
        ValueOperations<String, SysUserEntity> operations = redisTemplate.opsForValue();
        operations.set("user1", sysUserEntity);
        Assert.assertThat(sysUserEntity, Matchers.equalTo(operations.get("user1")));
    }

}
```

[Spring Data Redis 使用文档](https://docs.spring.io/spring-data/redis/docs/)
