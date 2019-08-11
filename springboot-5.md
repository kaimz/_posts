---
title: 学习Spring Boot：（五）使用 devtools热部署
date: 2018-02-23 10:08:06
tags: [Spring Boot]
categories: 学习笔记
---

### 前言
>spring-boot-devtools 是一个为开发者服务的一个模块，其中最重要的功能就是自动应用代码更改到最新的App上面去。原理是在发现代码有更改之后，重新启动应用，但是比速度比手动停止后再启动还要更快，更快指的不是节省出来的手工操作的时间。

 

其深层原理是使用了两个ClassLoader，一个Classloader加载那些不会改变的类（第三方Jar包），另一个ClassLoader加载会更改的类，称为  restart ClassLoader

,这样在有代码更改的时候，原来的restart ClassLoader 被丢弃，重新创建一个restart ClassLoader，由于需要加载的类相比较少，所以实现了较快的重启时间（5秒以内）。

<!--more-->

### 正文
#### 第一步：添加依赖
```xml
<!--  
           devtools可以实现页面热部署（即页面修改后会立即生效，这个可以直接在application.properties文件中配置spring.thymeleaf.cache=false来实现），      
           实现类文件热部署（类文件修改后不会立即生效），实现对属性文件的热部署。   
           即devtools会监听classpath下的文件变动，并且会立即重启应用（发生在保存时机），注意：因为其采用的虚拟机机制，该项重启是很快的      
        -->  
       <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-devtools</artifactId>  
            <optional>true</optional>  
        </dependency>  
```

#### 第二步：配置spring-boot-maven-plugin
```xml
<build>  
       <plugins>  
           <!--  
             用于将应用打成可直接运行的jar（该jar就是用于生产环境中的jar） 值得注意的是，如果没有引用spring-boot-starter-parent做parent，  
                       且采用了上述的第二种方式，这里也要做出相应的改动  
             -->  
            <plugin>  
                <groupId>org.springframework.boot</groupId>  
                <artifactId>spring-boot-maven-plugin</artifactId>  
                <configuration>  
                   <!--fork :  如果没有该项配置，肯呢个devtools不会起作用，即应用不会restart -->  
                    <fork>true</fork>  
                </configuration>  
            </plugin>  
       </plugins>  
   </build>  
```

#### 第三步：开启编译器的自动编译
*  Eclipse Project 在运行选项中查看是否开启了Build Automatically，如果没勾上，就选中勾上。Eclipse默认是自动编译的。
*  IDEA 默认在非RUN或DEBUG情况下才会自动编译。因此，我们在启动Spring Boot后，再次修改类的时候不会自动编译的，开启在Run状态自动编译的流程如下：
1. setting -> build
    ![springboot-5-1.png](https://i.loli.net/2018/01/25/5a69665a66fe7.png)
2. Shift+Ctrl+Alt+/，选择Registry；
3. 找到下面的选项，勾选上就行了：
    ![springboot-5-2.png](https://i.loli.net/2018/01/25/5a69673f3c07e.png)
    我的是已经勾选过了的，所以是蓝颜色的提示，第一次就去找c开头的就可以了。

#### 第四步：运行测试
启动项目，在run dashboard发现有devtools：
![springboot-5-3.png](https://i.loli.net/2018/01/25/5a696dec06776.png)
1. 修改类文件，项目重启；
2. 修改配置文件，项目重启。

#### 补充
1. 如果设置`SpringApplication.setRegisterShutdownHook(false)`，则自动重启将不起作用。
2. 默认情况下，`/META-INF/maven`，`/META-INF/resources`，`/resources`，`/static`，`/templates`，`/public`这些文件夹下的文件修改不会使应用重启，但是会重新加载（devtools内嵌了一个LiveReload server，当资源发生改变时，浏览器刷新）。
3. 如果想改变默认的设置，可以自己设置不重启的目录：`spring.devtools.restart.exclude=static/**,public/**`，这样的话，就只有这两个目录下的文件修改不会导致restart操作了。
4. 如果要在保留默认设置的基础上还要添加其他的排除目录：`spring.devtools.restart.additional-exclude`;
5. 如果想要使得当非classpath下的文件发生变化时应用得以重启，使用：`spring.devtools.restart.additional-paths`，这样devtools就会将该目录列入了监听范围。
6. 设置 spring.devtools.restart.enabled 属性为false，可以关闭自动重启。可以在application.properties中设置，也可以通过设置环境变量的方式。
```java
public static void main(String[] args){
    System.setProperty("spring.devtools.restart.enabled","false");
    SpringApplication.run(MyApp.class, args);
}
```

### 参考文章：
* <a rel="external nofollow" target="_blank" href="http://412887952-qq-com.iteye.com/blog/2300313">40. springboot + devtools（热部署）【从零开始学Spring Boot】
* <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/wjc475869/article/details/52442484"> Intellij IDEA 使用Spring-boot-devTools无效解决办法
