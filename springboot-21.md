---
title: 学习Spring Boot：（二十一）使用 EhCache 实现数据缓存
date: 2018-03-04 22:08:08
tags: [Spring Boot]
categories: 学习笔记
---

### 前言
当多次查询数据库影响到系统性能的时候，可以考虑使用缓存，来解决数据访问新能的问题。
SpringBoot 已经为我们提供了自动配置多个 CacheManager 的实现，只要去实现使用它就可以了。

<!--more-->

一般的系统都是优先使用 EhCache，它工作在 JAVA 进程中，在传统的应用没有太大要求的时候，使用它比较方便，分布式系统中去使用 Shiro 集中管理缓存。
### 正文
#### 加入依赖 
在 pom.xml 中加入依赖
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <dependency>
            <groupId>net.sf.ehcache</groupId>
            <artifactId>ehcache</artifactId>
        </dependency>
```
#### 添加缓存相关的配置
新建 `ehcache.xml`，加入缓存相关参数， 我新添加一个 name 为 users 的缓存设置：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
         updateCheck="false">
    <diskStore path="java.io.tmpdir/Tmp_EhCache"/>
    <defaultCache
            maxElementsInMemory="1000"
            maxEntriesLocalHeap="400"
            eternal="false"
            timeToIdleSeconds="120"
            timeToLiveSeconds="120"
            overflowToDisk="false"
            diskPersistent="false"
            diskExpiryThreadIntervalSeconds="120"/>

    <cache
            name="users"
            maxEntriesLocalHeap="200"
            timeToLiveSeconds="600"
    />

</ehcache>
```
参数详解：

* **name**:缓存名称。
* **maxElementsInMemory**：缓存最大个数。
* **eternal**:对象是否永久有效，一但设置了，timeout将不起作用。
* **timeToIdleSeconds**：设置对象在失效前的允许闲置时间（单位：秒）。仅当eternal=false对象不是永久有效时使用，可选属性，默认值是0，也就是可闲置时间无穷大。
* **timeToLiveSeconds**：设置对象在失效前允许存活时间（单位：秒）。最大时间介于创建时间和失效时间之间。仅当eternal=false对象不是永久有效时使用，默认是0.，也就是对象存活时间无穷大。
* **overflowToDisk**：当内存中对象数量达到maxElementsInMemory时，Ehcache将会对象写到磁盘中。
* **diskSpoolBufferSizeMB**：这个参数设置DiskStore（磁盘缓存）的缓存区大小。默认是30MB。每个Cache都应该有自己的一个缓冲区。
* **maxElementsOnDisk**：硬盘最大缓存个数。
* **diskPersistent**：是否缓存虚拟机重启期数据，默认为false。
* **diskExpiryThreadIntervalSeconds**：磁盘失效线程运行时间间隔，默认是120秒。
* **memoryStoreEvictionPolicy**：当达到maxElementsInMemory限制时，Ehcache将会根据指定的策略去清理内存。默认策略是LRU（最近最少使用）。你可以设置为FIFO（先进先出）或是LFU（较少使用）。
* **clearOnFlush**：内存数量最大时是否清除。
* **memoryStoreEvictionPolicy**：当达到maxElementsInMemory限制时，Ehcache将会根据指定的策略去清理内存。默认策略是LRU（最近最少使用）。你可以设置为FIFO（先进先出）或是LFU（较少使用）。   

#### 开启缓存

1. 在系统配置文件中指定
    在配置文件中加入指定我们设置的 `ehcache.xml` 作为 EhCache 的配置文件：
```yaml
spring:
  cache:
    ehcache:
      config: config/ehcache.xml # 指定 ehcache.xml 创建EhCache的缓存管理器
    type: ehcache # 指定缓存管理器
```
2. 在启动类上加上注解 `@EnableCaching`，开启缓存。

#### 使用
**使用的时候需要注意，我们之前在 shiro 缓存中 配置了相关的缓存的配置，现在需要把 shiro 相关的缓存的内容全部都要删除掉，不然两者的缓存会存在冲突。**
还是以 shiro 的获取权限列表的服务为例，不用 shiro-cache 后，直接在查询的这里自己添加上缓存就可以了。
```java
@CacheConfig(cacheNames = "users")
public interface ShiroService {
    /**
     * 获取用户权限
     *
     * @param userId 用户ID
     * @return 权限
     */
    @Cacheable
    Set<String> getUserPermissions(long userId);
```
debug 调试，
```java
@Autowired
private CacheManager cacheManager;

```
发现 key 为 `users`   中存储了相关内容。

#### 注解的使用
* `@CacheConfig`：主要用于配置该类中会用到的一些共用的缓存配置。在这里`@CacheConfig(cacheNames = "users")`：配置了该数据访问对象中返回的内容将存储于名为users的缓存对象中，我们也可以不使用该注解，直接通过@Cacheable自己配置缓存集的名字来定义。
* @Cacheable：配置了`getUserPermissions(long userId)`函数的返回值将被加入缓存。同时在查询时，会先从缓存中获取，若不存在才再发起对数据库的访问。该注解主要有下面几个参数：
1. `value`、`cacheNames`：两个等同的参数（cacheNames为Spring 4新增，作为value的别名），用于指定缓存存储的集合名。由于Spring 4中新增了@CacheConfig，因此在Spring 3中原本必须有的value属性，也成为非必需项了。
2. `key`：缓存对象存储在`Map`集合中的`key`值，非必需，缺省按照函数的所有参数组合作为key值，若自己配置需使用`SpEL`表达式，比如：`@Cacheable(key = "#p0")`：使用函数第一个参数作为缓存的key值，更多关于SpEL表达式的详细内容可参考[官方文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#cache)
3. `condition`：缓存对象的条件，非必需，也需使用SpEL表达式，只有满足表达式条件的内容才会被缓存，比如：`@Cacheable(key = "#p0", condition = "#p0.length() < 3")`，表示只有当第一个参数的长度小于3的时候才会被缓存，
4. `unless`：另外一个缓存条件参数，非必需，需使用SpEL表达式。它不同于`condition`参数的地方在于它的判断时机，该条件是在函数被调用之后才做判断的，所以它可以通过对`result`进行判断。
5. `keyGenerator`：用于指定`key`生成器，非必需。若需要指定一个自定义的key生成器，我们需要去实现`org.springframework.cache.interceptor.KeyGenerator`接口，并使用该参数来指定。需要注意的是：**该参数与key是互斥的**。
6. `cacheManager`：用于指定使用哪个缓存管理器，非必需。只有当有多个时才需要使用。
7. `cacheResolver`：用于指定使用那个缓存解析器，非必需。需通过`org.springframework.cache.interceptor.CacheResolver`接口来实现自己的缓存解析器，并用该参数指定。

除了这里用到的两个注解之外，还有下面几个核心注解：
* `@CachePut`：配置于函数上，能够根据参数定义条件来进行缓存，它与`@Cacheable`不同的是，它每次都会真是调用函数，所以主要用于数据新增和修改操作上。它的参数与`@Cacheable`类似，具体功能可参考上面对@Cacheable参数的解析
* `@CacheEvict`：配置于函数上，通常用在删除方法上，用来从缓存中移除相应数据。除了同`@Cacheable`一样的参数之外，它还有下面两个参数：
1. `allEntries`：非必需，默认为`false`。当为`true`时，会移除所有数据
2. `beforeInvocation`：非必需，默认为`false`，会在调用方法之后移除数据。当为`true`时，会在调用方法之前移除数据。
#### 缓存配置
>在Spring Boot中通过@EnableCaching注解自动化配置合适的缓存管理器（CacheManager），Spring Boot根据下面的顺序去侦测缓存提供者：
1. Generic
2. JCache (JSR-107)
3. EhCache 2.x
4. Hazelcast
5. Infinispan
6. Redis
7. Guava
8. Simple

通常还是推荐去指定一个 缓存类型比较好，在系统配置文件配置：
```yaml
spring:
  cache:
    type: ehcache
```
### Spring 项目中使用

#### 引入依赖

```xml
		<!--需要 spring-context-support 依赖-->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context-support</artifactId>
		</dependency>
		<dependency>
			<groupId>net.sf.ehcache</groupId>
			<artifactId>ehcache</artifactId>
		</dependency>
```

#### 在主配置文件中配置bean

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:cache="http://www.springframework.org/schema/cache"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
    	http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd">
    <!-- 缓存配置 -->
    <!-- 启用缓存注解功能(请将其配置在Spring主配置文件中) -->
    <cache:annotation-driven cache-manager="cacheManager"/>
    <!-- Spring提供的基于的Ehcache实现的缓存管理器 -->
    <bean id="cacheManagerFactory" class="org.springframework.cache.ehcache.EhCacheManagerFactoryBean">
        <!-- 配置 ehcache -->
        <property name="configLocation" value="classpath:spring/ehcache.xml"/>
    </bean>
    <bean id="cacheManager" class="org.springframework.cache.ehcache.EhCacheCacheManager">
        <property name="cacheManager" ref="cacheManagerFactory"/>
    </bean>

</beans>
```



### 参考文章

* [Spring Boot中的缓存支持（一）注解配置与EhCache使用](http://blog.didispace.com/springbootcache1/)