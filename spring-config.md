---
title: Spring中使用Configuration注入Bean
date: 2017-11-2 15:08:03
tags: [java]
categories: 学习笔记
---

在Spring容器中使用``applicationContext.xml``中来给对应的类注入对应的属性，来完成初始化，最典型的就是配置数据库连接池了。

<!--more-->

```xml
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"
		init-method="init" destroy-method="close">
		<!-- 基本属性 url、user、password -->
		<property name="url" value="${connection.url}" />
		<property name="username" value="${connection.username}" />
		<property name="password" value="${connection.password}" />
		<!-- 配置初始化大小、最小、最大 -->
		<property name="initialSize" value="${druid.initialSize}" />
		<property name="minIdle" value="${druid.minIdle}" />
		<property name="maxActive" value="${druid.maxActive}" />
		<!-- 配置获取连接等待超时的时间 -->
		<property name="maxWait" value="${druid.maxWait}" />
		<!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
		<property name="timeBetweenEvictionRunsMillis" value="${druid.timeBetweenEvictionRunsMillis}" />
		<!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
		<property name="minEvictableIdleTimeMillis" value="${druid.minEvictableIdleTimeMillis}" />
		<property name="removeAbandoned" value="${druid.removeAbandoned}" />
		<!-- 超时时间；单位为秒。180秒=3分钟 -->
		<property name="removeAbandonedTimeout" value="${druid.removeAbandonedTimeoutSeconds}" />
		<property name="validationQuery" value="${druid.validationQuery}" />
		<property name="testWhileIdle" value="${druid.testWhileIdle}" />
		<property name="testOnBorrow" value="${druid.testOnBorrow}" />
		<property name="testOnReturn" value="${druid.testOnReturn}" />
		<!-- 打开PSCache，并且指定每个连接上PSCache的大小 如果用Oracle，则把poolPreparedStatements配置为true，mysql可以配置为false。 -->
		<property name="poolPreparedStatements" value="${druid.poolPreparedStatements}" />
		<property name="maxPoolPreparedStatementPerConnectionSize"
			value="${druid.maxPoolPreparedStatementPerConnectionSize}" />
		<!-- 配置监控统计拦截的filters -->
		<property name="filters" value="${druid.filters}" />
	</bean>
```
配置参数就不贴出。

## 使用@Configuration创建Bean

Configuration 是 Spring 3.X 后提供的注解，用于取代 XML 来配置 Spring, 

``@Configuration``可理解为用spring的时候xml里面的``<beans>``标签；

``@Bean``可理解为用spring的时候xml里面的``<bean>``标签。

这样就很好理解了。

需要注意的时配置``spring`` 扫描的包 ``<context:component-scan base-package="com.xxx.xxx" />`` 不然注解不起效果（springboot不需要设置）。
### 读取json文件的属性注入Bean

这次使用Json文件来配置bean；

首先写出实体类，和需要配置的数据；
#### 编写实体类

```java
package com.devframe.util;

import java.util.List;

/**
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/2 11:31</pre>
 */
public class PersonCfg {

    private String name;
    private int age;
    private String city;
    private List<Contact> contacts;

    @Override
    public String toString() {
        return "PersonCfg{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", city='" + city + '\'' +
                ", contacts=" + contacts +
                ", hobby=" + hobby +
                '}';
    }

    public List<Contact> getContacts() {
        return contacts;
    }

    public void setContacts(List<Contact> contacts) {
        this.contacts = contacts;
    }

    public List getHobby() {
        return hobby;
    }

    public void setHobby(List hobby) {
        this.hobby = hobby;
    }

    private List hobby;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getCity() {
        return city;
    }

    public void setCity(String city) {
        this.city = city;
    }

}

class Contact {

    private String phone;

    private String email;

    public String getPhone() {
        return phone;
    }

    public void setPhone(String phone) {
        this.phone = phone;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    @Override
    public String toString() {
        return "Contact{" +
                "phone='" + phone + '\'' +
                ", email='" + email + '\'' +
                '}';
    }
}

```

#### 需要注入的数据
创建文件命名``data.json``，(注意属性名对应):
```json
{
  "name": "wuwii",
  "age": 23,
  "city": "WuHan",
  "hobby": ["骑行", "跑步","足球"],
  "contacts": [{
    "phone": "18772383543",
    "email": "k@wuwii.com"
  },{
    "phone": "12345678912",
    "email": "1075199251@qq.com"
  }]
}
```
#### 创建Beans
spring 容器初始化，自动扫描，去初始化Bean，加载进Environment，后面调用的直接自动装配（Autowired）；
```java
package com.devframe.util;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.io.File;
import java.io.IOException;

/**
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/2 11:23</pre>
 */
@Configuration
public class Configs {
    @Value("classpath:data.json")
    protected File configFile;

    @Bean
    public PersonCfg readServerConfig() throws IOException {
        return new ObjectMapper().readValue(configFile, PersonCfg.class);
    }

}


```
``@Bean`` 注解方法的返回值，将注入到容器中，可以使用自动装配。

#### 测试
直接使用spring-test 的JUnit4 单元测试;
直接装配Bean ，来输出它的属性，查看是否装配成功。
```java
package com.devframe.util; 

import org.junit.Test; 
import org.junit.Before; 
import org.junit.After;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

/** 
* Configs Tester. 
* 
* @author Zhang Kai 
* @since <pre>11/02/2017</pre> 
* @version 1.0 
*/
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:spring/applicationContext-base.xml"})
public class ConfigsTest {
    @Autowired
    private PersonCfg personCfg;

@Before
public void before() throws Exception { 
} 

@After
public void after() throws Exception { 
} 

/** 
* 
* Method: 名字随便起的，不规范。
* 
*/ 
@Test
public void testConfigBeans() throws Exception {
    System.out.printf("Use '@Configuration' autowired beans : %s%n", personCfg);
} 


} 

```

测试结果：
```java
Use '@Configuration' autowired beans : PersonCfg{name='wuwii', age=23, city='WuHan', contacts=[Contact{phone='18772383543', email='k@wuwii.com'}, Contact{phone='12345678912', email='1075199251@qq.com'}], hobby=[骑行, 跑步]}
```
### 读取properties 文件的属性注入Bean 
上面的的方法中除了测试类的方法相同而已，为了方便其余都有改动；

#### 首先实体类，通过构造方法传入值

```java
package com.devframe.util;

import java.util.List;

/**
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/2 11:31</pre>
 */
public class PersonCfg {

    private String name;
    private int age;
    private String city;
    private List<Contact> contacts;
    private List hobby;

    public PersonCfg() {
    }


    public PersonCfg(String name, int age, String city, List<Contact> contacts, List hobby) {
        this.name = name;
        this.age = age;
        this.city = city;
        this.contacts = contacts;
        this.hobby = hobby;
    }

    @Override
    public String toString() {
        return "PersonCfg{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", city='" + city + '\'' +
                ", contacts=" + contacts +
                ", hobby=" + hobby +
                '}';
    }

    public List<Contact> getContacts() {
        return contacts;
    }

    public void setContacts(List<Contact> contacts) {
        this.contacts = contacts;
    }

    public List getHobby() {
        return hobby;
    }

    public void setHobby(List hobby) {
        this.hobby = hobby;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getCity() {
        return city;
    }

    public void setCity(String city) {
        this.city = city;
    }

}

class Contact {

    private String phone;

    private String email;

    public Contact(String phone, String email) {
        this.phone = phone;
        this.email = email;
    }

    public String getPhone() {
        return phone;
    }

    public void setPhone(String phone) {
        this.phone = phone;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    @Override
    public String toString() {
        return "Contact{" +
                "phone='" + phone + '\'' +
                ", email='" + email + '\'' +
                '}';
    }
}
```
#### 配置文件
由于properties 文件不能写 只能写那些单一属性，数组和对象需要自己设置规则，去后台解析出来使用。
创建``person.properties`` 文件：
```
name=wuwii
age=23
city=WuHan
hobby=football,running
contacts=18772383543,k@wuwii.com;12345678912,1075199251@qq.com
```
#### 创建Bean
通过@Configuration完成spring 初始化，设置@PropertySource，读取配置文件：
```java
package com.devframe.util;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.core.env.Environment;

import javax.annotation.Resource;
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

/**
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/2 11:23</pre>
 */
@Configuration
@PropertySource("classpath:person.properties")
public class Configs {
    @Resource
    private Environment env;

    @Bean
    public PersonCfg getPersonFromProp() {
        return new PersonCfg(env.getProperty("name"), Integer.valueOf(env.getProperty("age")),
                env.getProperty("city"), string2contacts(env.getProperty("contacts")), string2list(env.getProperty("hobby")));
    }
    /**
     *  按照预先定义规则的列表字符串 转换成列表
     * @param s 预先定义规则的列表字符串
     * @return java.util.List
     */
    private List string2list (String s) {
        return StringUtil.isNull(s) ? null : Arrays.asList(s.split(","));
    }

    /**
     *  <p>按照预先定义规则</p>
     *  <p>将配置文件 Contact 列表的字符串 转换成 列表对象</p>
     * @param s 读取配置文件 Contact 列表的字符串
     * @return java.util.List<com.devframe.util.Contact>
     */
    private List<Contact> string2contacts (String s) {
        if (StringUtil.isNull(s)) return null;
        List<String> list1 = Arrays.asList(s.split(";"));
        return list1.stream().map(this::contactStr2contact).collect(Collectors.toList());
    }

    /**
     * 按照预定义规则转换成 contact对象
     * @param contactStr contact类的字符串
     * @return com.devframe.util.Contact
     */
    private Contact contactStr2contact (String contactStr) {
        String[] index = contactStr.split(",");
        // 传入字段数，自己控制，有点蠢了
        if (index.length != 2) {
            return null;
        }
        return new Contact(index[0], index[1]);
    }
}

```
#### 测试
最后JUnit4 测试类没变，重新测试，打印出来结果：
```
Use '@Configuration' autowired beans : PersonCfg{name='wuwii', age=23, city='WuHan', contacts=[Contact{phone='18772383543', email='k@wuwii.com'}, Contact{phone='12345678912', email='1075199251@qq.com'}], hobby=[football, running]}
```
成功。

#### 补充
20171103 早上来看了下源码：
```java
	/**
	 * Return the property value associated with the given key, or {@code null}
	 * if the key cannot be resolved.
	 * @param key the property name to resolve
	 * @see #getProperty(String, String)
	 * @see #getProperty(String, Class)
	 * @see #getRequiredProperty(String)
	 */
	String getProperty(String key);

	/**
	 * Return the property value associated with the given key, or
	 * {@code defaultValue} if the key cannot be resolved.
	 * @param key the property name to resolve
	 * @param defaultValue the default value to return if no value is found
	 * @see #getRequiredProperty(String)
	 * @see #getProperty(String, Class)
	 */
	String getProperty(String key, String defaultValue);

	/**
	 * Return the property value associated with the given key, or {@code null}
	 * if the key cannot be resolved.
	 * @param key the property name to resolve
	 * @param targetType the expected type of the property value
	 * @see #getRequiredProperty(String, Class)
	 */
	<T> T getProperty(String key, Class<T> targetType);
	
	/**
	 * Return the property value associated with the given key, or
	 * {@code defaultValue} if the key cannot be resolved.
	 * @param key the property name to resolve
	 * @param targetType the expected type of the property value
	 * @param defaultValue the default value to return if no value is found
	 * @see #getRequiredProperty(String, Class)
	 */
	<T> T getProperty(String key, Class<T> targetType, T defaultValue);
```
在``PropertyResolver``接口中发现：
```java
<T> T getProperty(String key, Class<T> targetType);
```
这个方法可以直接读取文件内容转换成我们的需要类型，虽然说很好，调试了半天代码不知道properties文件怎么写对象来让它转换，这个以后再看，list列表很好转，将上面的方法加载hobby属性改成这个：
```java
env.getProperty(("age"), Integer.class)
env.getProperty(("hobby"), List.class)
```
person文件中hobby属性为：

```
hobby=running,football
```


执行结果：
```
Use '@Configuration' autowired beans : PersonCfg{name='wuwii', age=23, city='WuHan', contacts=[Contact{phone='18772383543', email='k@wuwii.com'}, Contact{phone='12345678912', email='1075199251@qq.com'}], hobby=[running, football]}
```

没问题
#### 直接使用@Value占位符注入
##### 方法一
使用``@Component`` 方式注入，需要再applicationContext.xml中引入properties文件：

```xml
	<!-- 参数占位符 -->
	<bean
		class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer"
		lazy-init="true">
		<property name="systemPropertiesModeName" value="SYSTEM_PROPERTIES_MODE_OVERRIDE" />
		<property name="ignoreResourceNotFound" value="false" />
		<property name="locations">
			<list>
				<value>classpath:spring/database.properties</value>
				<value>classpath:person.properties</value>
			</list>
		</property>
	</bean>
```
改下实体类，直接在属性上注入``@Value``，占位符符号``${ }``
```java
package com.devframe.util;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.PropertySource;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/2 11:31</pre>
 */
@Component
public class PersonCfg {

    @Value("${name}")
    private String name;
    @Value("${age}")
    private int age;
    @Value("${city}")
    private String city;
    //这个不会，对象属性不会写
    //@Value("${contacts1}")
    private List<Contact> contacts;
    @Value("${hobby}")
    private List hobby;

//省略代码

}


```
测试结果:
```
Use '@Configuration' autowired beans : PersonCfg{name='wuwii', age=23, city='WuHan', contacts=null, hobby=[running,football]}
```
发现数组列表也能直接注入。
##### 方法二

在配置类中设置引入配置文件，还需引入占位符，等价于XML中的``<context:property-placeholder/>``配置。
```java
@Configuration
@PropertySource("classpath:person.properties")
public class Configs {
    private static final Logger LOGGER = LoggerFactory.getLogger(Configs.class);
    @Autowired
    private Environment env;

    @Bean
    public PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }
    //省略
}
```
就可以在类中的属性上使用@Value占位符 注入了。

## 总结
1. 还可以读取xml文件进行装配，当然也不使用配置文件，直接在Beans的``@Value``注解上写出需要注解的值，但是那样后期部署修改起来麻烦。
2. 常用的应该时这么两个 比较好，properties 可能用的多点吧；因为平时使用这个外部需要修改的参数 的基本都是一些常量，不会存在这么多转换，这个只是我的测试的代码，所以有一些鬼转换。
3. 还有我使用properties 中为什么没使用中文，因为乱码了。尴尬。这是需要注意的地方，因为电脑默认编码是gbk，但是读的时候，又没有设置编码。解决办法：在读取properties文件的工具类上，加上指定编码格式``utf-8``:
```java
URL url = PropertyUtil.class.getResource("/config.properties");
FileInputStream in = new FileInputStream(url.getPath());
//这段代码不是 以前的  PROP.load(in);
PROP.load(new InputStreamReader(in, "utf-8"));
in.close();
```

