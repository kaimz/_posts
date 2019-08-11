---
title: 学习Spring Boot：（七）集成Mybatis
date: 2018-02-23 10:08:08
tags: [Spring Boot]
categories: 学习笔记
---

前面都是用的是spring data JPA，现在学习下Mybatis，而且现在Mybatis也像JPA那样支持注解形式了，也非常方便，学习一下。

* 数据库 mysql 5.7

<!--more-->

### 添加依赖
在pom文件中添加：
```xml
<mybatis.version>1.3.1</mybatis.version>
<druid.version>1.1.3</druid.version>

        <dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>${mybatis.version}</version>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid-spring-boot-starter</artifactId>
			<version>${druid.version}</version>
		</dependency>
```
由于springboot 默认使用的 tomcat-jdbc数据源，为了方便，我添加了阿里巴巴的数据源。

首先了解下`mybatis-spring-boot-starter` 会做哪些事情：
* 自动检测现有的DataSource
* 将创建并注册SqlSessionFactory的实例，该实例使用SqlSessionFactoryBean将该DataSource作为输入进行传递
* 将创建并注册从SqlSessionFactory中获取的SqlSessionTemplate的实例。
* 自动扫描您的mappers，将它们链接到SqlSessionTemplate并将其注册到Spring上下文，以便将它们注入到您的bean中。

只要使用这个springboot mybatis starter 只需要DataSource的配置就可以使用了，它会自动创建使用该DataSource的SqlSessionFactoryBean以及SqlSessionTemplate。会自动扫描你的Mappers，连接到SqlSessionTemplate，并注册到Spring上下文中。

### 配置数据源
在`resources/applicaiton.yml`文件中添加一些数据源的连接参数配置（可选）：
```yaml
spring:
    datasource:
        type: com.alibaba.druid.pool.DruidDataSource
        driverClassName: com.mysql.jdbc.Driver
        druid:
            url: jdbc:mysql://localhost:3306/learnboot?allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8
            username: root
            password: 123456
            initial-size: 10
            max-active: 100
            min-idle: 10
            max-wait: 60000
            pool-prepared-statements: true
            max-pool-prepared-statement-per-connection-size: 20
            time-between-eviction-runs-millis: 60000
            min-evictable-idle-time-millis: 300000
            validation-query: SELECT 1
            test-while-idle: true
            test-on-borrow: false
            test-on-return: false
            stat-view-servlet:
                enabled: true
                url-pattern: /druid/*
            filter:
                stat:
                    log-slow-sql: true
                    slow-sql-millis: 1000
                    merge-sql: true
                wall:
                    config:
                        multi-statement-allow: true
```
springboot会自动加载spring.datasource.*相关配置，数据源就会自动注入到sqlSessionFactory中，sqlSessionFactory会自动注入到Mapper中，

### 使用Mybatis
首先我有一个`SysUser`，
```java
public class SysUserEntity implements Serializable {
    private static final long serialVersionUID = 1L;

    //主键
    private Long id;
    //用户名
    @NotBlank(message = "用户名不能为空", groups = {AddGroup.class, UpdateGroup.class})
    private String username;
    //密码
    private String password;
```

#### 使用XMl形式
我们来创建User的映射SysUserDao，也可以命名Mapper作为尾缀，这里我们写个新增一条数据的接口，需要注意的是每个Mapper类上要加上`@Mapper`注解：
```java
@Mapper
public interface SysUserDao {
    void save(SysUserEntity user);
    /**
     * 根据条件查询User
     * @param user User
     * @return 符合条件列表
     */
    List<SysUserEntity> query(SysUserEntity user);
}
```
使用xml的时候需要注意的是Mybatis扫描mapper.xml并且装配，需要在系统的配置文件``resources/applicaiton.yml``加入：
```
# Mybatis配置
mybatis:
    mapperLocations: classpath:mapper/**/*.xml
    configuration:
        map-underscore-to-camel-case: true

```
根据自己的xml目录，进行配置。
例如：我在`resources/mapper/sys`目录下面加入`SysUserDao.xml`文件，添加我们查询的SQL：
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.wuwii.module.sys.dao.SysUserDao">

    <!-- 可根据自己的需求，是否要使用 -->
    <resultMap type="com.wuwii.module.sys.entity.SysUserEntity" id="sysUserMap">
        <result property="id" column="id"/>
        <result property="username" column="username"/>
        <result property="password" column="password"/>
    </resultMap>
    
        <insert id="save" parameterType="com.wuwii.module.sys.entity.SysUserEntity" useGeneratedKeys="true"
            keyProperty="id">
        INSERT INTO sys_user
        (
            `username`,
            `password`
        )
        VALUES
            (
                #{username},
                #{password}
            )
    </insert>
    
    <select id="query" resultType="com.wuwii.module.sys.entity.SysUserEntity">
        SELECT *
        FROM sys_user
       <where>
           <if test="username != null"> and `username` = #{username}</if>
           <if test="password != null">and `password` = #{password}</if>
       </where>
    </select>
```

#### 使用注解形式
这个就要方便很多，没有Mapper.xml文件了，也不要配置它的文件路径的映射了，只要把xml中的SQL 写到注解上就可以了。
直接在`SysUserDao`中改成：
```java
@Mapper
public interface SysUserDao {
    @Insert("INSERT INTO sys_user(username,password) VALUES(#{username}, #{password})")
    void save(SysUserEntity user);
    
    @Select("SELECT * FROM sys_user WHERE id = #{id}")
	@Results({
		@Result(property = "username",  column = "username", javaType = String.class),
		@Result(property = "password", column = "password")
	})
	UserEntity getOne(Long id);
}
```
根据对数据库的操作不同，使用不同的注解：
* @Select 是查询类的注解，所有的查询均使用这个
* @Result 修饰返回的结果集，关联实体类属性和数据库字段一一对应，如果实体类属性和数据库属性名保持一致，就不需要这个属性来修饰。
* @Insert 插入数据库使用，直接传入实体类会自动解析属性到对应的值
* @Update 负责修改，也可以直接传入对象
* @Delete 负责删除

**注意，使用#符号和$符号的不同：**
```java
// This example creates a prepared statement, something like select * from teacher where name = ?;
@Select("Select * from teacher where name = #{name}")
Teacher selectTeachForGivenName(@Param("name") String name);

// This example creates n inlined statement, something like select * from teacher where name = 'someName';
@Select("Select * from teacher where name = '${name}'")
Teacher selectTeachForGivenName(@Param("name") String name);
```

### 单元测试
```java
@SpringBootTest
@RunWith(SpringJUnit4ClassRunner.class)
public class UserDaoTest {
    @Resource
    private SysUserDao userDao;

    @Test
    @Transactional
    @Rollback(false)
    public void testSave() {
        SysUserEntity user = new SysUserEntity();
        user.setUsername("wuwii");
        user.setPassword("123");
        userDao.save(user);
        Assert.assertEquals(user.getUsername(), userDao.query(user).get(0).getUsername());
    }
}

```

