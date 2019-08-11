---
title: 学习Spring Boot：（二十四）多数据源配置与使用
date: 2018-03-15 22:12:03
tags: [Spring Boot]
categories: 学习笔记
---

### 前言
随着业务量增大，可能有些业务不是放在同一个数据库中，所以系统有需求使用多个数据库完成业务需求，我们需要配置多个数据源，从而进行操作不同数据库中数据。

<!--more-->
### 正文
#### JdbcTemplate 多数据源
##### 配置
需要在 Spring Boot 中配置多个数据库连接，当然怎么设置连接参数的 key 可以自己决定，
>需要注意的是 `Spring Boot 2.0` 的默认连接池配置参数好像有点问题，由于默认连接池已从 `Tomcat` 更改为 `HikariCP`，以前有一个参数 `url`，已经改成 `hikari.jdbcUrl` ，不然无法注册。我下面使用的版本是 `1.5.9`。
```yaml
server:
  port: 8022
spring:
  datasource:
     url: jdbc:mysql://localhost:3306/learn?useSSL=false&allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8
     username: root
     password: 123456
     driver-class-name: com.mysql.jdbc.Driver

  second-datasource:
    url: jdbc:mysql://localhost:3306/learn1?useSSL=false&allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8
    username: root
    password: 123457
    driver-class-name: com.mysql.jdbc.Driver
```
##### 注册 DataSource
注册两个数据源，分别注册两个 `JdbcTemplate`，
```java
@Configuration
public class DataSourceConfig {

    /**
     * 注册 data source
     *
     * @return
     */
    @ConfigurationProperties(prefix = "spring.datasource")
    @Bean("firstDataSource")
    @Primary // 有相同实例优先选择
    public DataSource firstDataSource() {
        return DataSourceBuilder.create().build();
    }

    @ConfigurationProperties(prefix = "spring.second-datasource")
    @Bean("secondDataSource")
    public DataSource secondDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean("firstJdbcTemplate")
    @Primary
    public JdbcTemplate firstJdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }

    @Bean("secondJdbcTemplate")
    public JdbcTemplate secondJdbcTemplate(@Qualifier("secondDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```
##### 测试
```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class TestJDBC {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    @Autowired
    @Qualifier("secondJdbcTemplate")
    private JdbcTemplate jdbcTemplate1;

    @Before
    public void before() {
        jdbcTemplate.update("DELETE FROM employee");
        jdbcTemplate1.update("DELETE FROM employee");
    }

    @Test
    public void testJDBC() {
        jdbcTemplate.update("insert into employee(id,name,age) VALUES (1, 'wuwii', 24)");
        jdbcTemplate1.update("insert into employee(id,name,age) VALUES (1, 'kronchan', 23)");
        Assert.assertThat("wuwii", Matchers.equalTo(jdbcTemplate.queryForObject("SELECT name FROM employee WHERE id=1", String.class)));
        Assert.assertThat("kronchan", Matchers.equalTo(jdbcTemplate1.queryForObject("SELECT name FROM employee WHERE id=1", String.class)));
    }
}
```
[源码地址](https://github.com/kaimz/learning-code/tree/master/jdbc-muti-datasource)

#### 使用 JPA 支持多数据源
##### 配置
相比使用 `jdbcTemplate`，需要设置下 `JPA` 的相关参数即可，没多大变化：
```yaml
server:
  port: 8022
spring:
  datasource:
     url: jdbc:mysql://localhost:3306/learn?useSSL=false&allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8
     username: root
     password: 123456
     driver-class-name: com.mysql.jdbc.Driver

  second-datasource:
    url: jdbc:mysql://localhost:3306/learn1?useSSL=false&allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver

  jpa:
    show-sql: true
    database: mysql
    hibernate:
    # update 更新表结构
    # create 每次启动删除上次表，再创建表，会造成数据丢失
    # create-drop： 每次加载hibernate时根据model类生成表，但是sessionFactory一关闭,表就自动删除。
    # validate ：每次加载hibernate时，验证创建数据库表结构，只会和数据库中的表进行比较，不会创建新表，但是会插入新值。
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
```
首先一样的是我们要注册相应的 DataSource，还需要指定相应的数据源所对应的实体类和数据操作层 `Repository`的位置：
* `firstDataSource`
```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        entityManagerFactoryRef = "firstEntityManagerFactory",
        transactionManagerRef = "firstTransactionManager",
        basePackages = "com.wuwii.module.system.dao" // 设置该数据源对应 dao 层所在的位置
)
public class FirstDataSourceConfig {

    @Autowired
    private JpaProperties jpaProperties;
    @Primary
    @Bean(name = "firstEntityManager")
    public EntityManager entityManager(EntityManagerFactoryBuilder builder) {
        return entityManagerFactoryPrimary(builder).getObject().createEntityManager();
    }

    @ConfigurationProperties(prefix = "spring.datasource")
    @Bean("firstDataSource")
    @Primary // 有相同实例优先选择，相同实例只能设置唯一
    public DataSource firstDataSource() {
        return DataSourceBuilder.create().build();
    }
    @Primary
    @Bean(name = "firstEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean entityManagerFactoryPrimary (EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(firstDataSource())
                .properties(getVendorProperties(firstDataSource()))
                .packages("com.wuwii.module.system.entity") //设置该数据源对应的实体类所在位置
                .persistenceUnit("firstPersistenceUnit")
                .build();
    }

    private Map<String, String> getVendorProperties(DataSource dataSource) {
        return jpaProperties.getHibernateProperties(dataSource);
    }

    @Primary
    @Bean(name = "firstTransactionManager")
    public PlatformTransactionManager transactionManagerPrimary(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(entityManagerFactoryPrimary(builder).getObject());
    }

}
```
* `secondDataSource`：
```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        entityManagerFactoryRef = "secondEntityManagerFactory",
        transactionManagerRef = "secondTransactionManager",
        basePackages = "com.wuwii.module.user.dao" // 设置该数据源 dao 层所在的位置
)
public class SecondDataSourceConfig {

    @Autowired
    private JpaProperties jpaProperties;
    @Bean(name = "secondEntityManager")
    public EntityManager entityManager(EntityManagerFactoryBuilder builder) {
        return entityManagerFactoryPrimary(builder).getObject().createEntityManager();
    }


    @ConfigurationProperties(prefix = "spring.second-datasource")
    @Bean("secondDataSource")
    public DataSource secondDataSource() {
        return DataSourceBuilder.create().build();
    }
    @Bean(name = "secondEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean entityManagerFactoryPrimary (EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(secondDataSource())
                .properties(getVendorProperties(secondDataSource()))
                .packages("com.wuwii.module.user.entity") //设置该数据源锁对应的实体类所在的位置
                .persistenceUnit("secondPersistenceUnit")
                .build();
    }

    private Map<String, String> getVendorProperties(DataSource dataSource) {
        return jpaProperties.getHibernateProperties(dataSource);
    }

    @Bean(name = "secondTransactionManager")
    public PlatformTransactionManager transactionManagerPrimary(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(entityManagerFactoryPrimary(builder).getObject());
    }

}
```
##### 测试
```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class TestDemo {
    @Autowired
    private EmployeeDao employeeDao;
    @Autowired
    private UserDao userDao;
    @Before
    public void before() {
        employeeDao.deleteAll();
        userDao.deleteAll();
    }

    @Test
    public void test() {
        Employee employee = new Employee(null, "wuwii", 24);
        employeeDao.save(employee);
        User user = new User(null, "KronChan", 24);
        userDao.save(user);
        Assert.assertThat(employee, Matchers.equalTo(employeeDao.findOne(Example.of(employee))));
        Assert.assertThat(user, Matchers.equalTo(userDao.findOne(Example.of(user))));
    }
}
```
[源码地址](https://github.com/kaimz/learning-code/tree/master/jpa-muti-datasource)
