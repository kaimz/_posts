---
title: 学习 Apache Shiro 架构
date: 2018-02-23 10:08:18
tags: [shiro]
categories: 学习笔记
---

很久以前就在公司的项目接触过 `Shiro`，但是现在想在自己的项目中集成它，并且结合 `JWT`，自己需要定义一些特殊的验证，却出现了很多问题，并且还没好好解决，当我重新把所有基础的架构学习了一遍，很快就找到了问题所在，并且对 Shiro 的认证流程有比较清晰的了解。
学习的时候，不能总是停留在怎么使用它的地步，许多的配置，都是从网上直接 copy 过来的，却不知道为很么需要这么写。这是工作中存在的非常大的隐患，直到需要你架构一个项目的时候，才知道，即使不需要阅读源码，也要清楚相关代码写的到底是为了什么，不能一知半解。

<!--more-->

### 架构体系
#### 架构介绍
Apache Shiro是一个安全框架，提供了认证（登陆）、授权（权限）、加密（密码）和会话管理等功能： 

Shiro 的三大组件：

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/learn-shiro/1.png)

* `Subject`：即`当前操作用户`。但是，在Shiro中，Subject这一概念并不仅仅指人，也可以是第三方进程、后台帐户（Daemon Account）或其他类似事物。它仅仅意味着`当前跟软件交互的东西`。但考虑到大多数目的和用途，你可以把它认为是Shiro的`用户`概念。 
  Subject代表了当前用户的安全操作，SecurityManager则管理所有用户的安全操作。 
* `SecurityManager`：它是Shiro框架的核心，典型的Facade模式，Shiro通过SecurityManager来管理内部组件实例，并通过它来提供安全管理的各种服务。 
* `Realm`： Realm充当了Shiro与应用安全数据间的“桥梁”或者“连接器”。也就是说，当对用户执行认证（登录）和授权（访问控制）验证时，Shiro会从应用配置的Realm中查找用户及其权限信息。 
  从这个意义上讲，Realm实质上是一个安全相关的DAO：它封装了数据源的连接细节，并在需要时将相关数据提供给Shiro。当配置Shiro时，你必须至少指定一个Realm，用于认证和（或）授权。配置多个Realm是可以的，但是至少需要一个。 
  Shiro内置了可以连接大量安全数据源（又名目录）的Realm，如LDAP、关系数据库（JDBC）、类似INI的文本配置资源以及属性文件等。如果缺省的Realm不能满足需求，你还可以插入代表自定义数据源的自己的Realm实现。

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/learn-shiro/2.png)

#### 认证
认证大致上就是系统的登陆相关的功能，我们有两种方法进行验证：
* 一是提交需要验证的实体信息，也就是我们平常的登陆时候需要的用户名和密码；
* 二是使用凭据信息进行验证合法性，一般就是使用一个加密并且有效时常的 token，服务端进行解密，查询数据库，得到该用户的信息。

##### 收集实体/凭据信息
```java
//Example using most common scenario of username/password pair:  
UsernamePasswordToken token = new UsernamePasswordToken(username, password);  
//”Remember Me” built-in:  
token.setRememberMe(true); 
```
>`UsernamePasswordToken` 支持最常见的用户名/密码的认证机制。同时，由于它实现了 `RememberMeAuthenticationToken` 接口，我们可以通过令牌设置“记住我”的功能。 
>是，“已记住”和“已认证”是有区别的： 
>*已记住的用户仅仅是非匿名用户，你可以通过 `subject.getPrincipals()` 获取用户信息。但是它并非是完全认证通过的用户，当你访问需要认证用户的功能时，你仍然需要重新提交认证信息**。 
>一区别可以参考亚马逊网站，网站会默认记住登录的用户，再次访问网站时，对于非敏感的页面功能，页面上会显示记住的用户信息，但是当你访问网站账户信息时仍然需要再次进行登录认证。 

##### 提交实体/凭据信息 
```java
Subject currentUser = SecurityUtils.getSubject();  
currentUser.login(token);  
```
收集了实体/凭据信息之后，我们可以通过SecurityUtils工具类，获取当前的用户，然后通过调用login方法提交认证。 

##### 认证处理
```java
try {  
    currentUser.login(token);  
} catch ( UnknownAccountException uae ) { ...  
} catch ( IncorrectCredentialsException ice ) { ...  
} catch ( LockedAccountException lae ) { ...  
} catch ( ExcessiveAttemptsException eae ) { ...  
} ... catch your own ...  
} catch ( AuthenticationException ae ) {  
    //unexpected error?  
}  
```
如果login方法执行完毕且没有抛出任何异常信息，那么便认为用户认证通过。之后在应用程序任意地方调用SecurityUtils.getSubject() 都可以获取到当前认证通过的用户实例，使用subject.isAuthenticated()判断用户是否已验证都将返回true. 
相反，如果login方法执行过程中抛出异常，那么将认为认证失败。Shiro有着丰富的层次鲜明的异常类来描述认证失败的原因，如代码示例。 

#### 登出
登出操作可以通过调用subject.logout()来删除你的登录信息，如： 
```java
currentUser.logout(); //removes all identifying information and invalidates their session too.
```
#### 认证内部处理机制
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/learn-shiro/3.png)

如上图，我们通过Shiro架构图的认证部分，来说明Shiro认证内部的处理顺序： 
1. 应用程序构建了一个终端用户认证信息的AuthenticationToken 实例后，调用Subject.login方法。
2. Sbuject的实例通常是DelegatingSubject类（或子类）的实例对象，在认证开始时，会委托应用程序设置的securityManager实例调用securityManager.login(token)方法。 
3. SecurityManager接受到token(令牌)信息后会委托内置的Authenticator的实例（通常都是ModularRealmAuthenticator类的实例）调用authenticator.authenticate(token). ModularRealmAuthenticator在认证过程中会对设置的一个或多个Realm实例进行适配，它实际上为Shiro提供了一个可拔插的认证机制。 
4. 如果在应用程序中配置了多个Realm，ModularRealmAuthenticator会根据配置的AuthenticationStrategy(认证策略)来进行多Realm的认证过程。在Realm被调用后，AuthenticationStrategy将对每一个Realm的结果作出响应。  
    注：如果应用程序中仅配置了一个Realm，Realm将被直接调用而无需再配置认证策略。
5. 判断每一个Realm是否支持提交的token，如果支持，Realm将调用getAuthenticationInfo(token); getAuthenticationInfo 方法就是实际认证处理，我们通过覆盖Realm的doGetAuthenticationInfo方法来编写我们自定义的认证处理。 

#### 使用多个Realm认证
##### Authenticator（认证器）
默认实现是 `ModularRealmAuthenticator`,它既支持单一Realm也支持多个Realm。如果仅配置了一个Realm，ModularRealmAuthenticator 会直接调用该Realm处理认证信息，如果配置了多个Realm，它会根据认证策略来适配Realm，找到合适的Realm执行认证信息。 
##### AuthenticationStrategy（认证策略） 
当应用程序配置了多个Realm时，ModularRealmAuthenticator将根据认证策略来判断认证成功或是失败。 
例如，如果只有一个Realm验证成功，而其他Realm验证失败，那么这次认证是否成功呢？如果大多数的Realm验证成功了，认证是否就认为成功呢？或者，一个Realm验证成功后，是否还需要判断其他Realm的结果？认证策略就是根据应用程序的需要对这些问题作出决断。 
认证策略是一个无状态的组件，在认证过程中会经过4次的调用： 
* 在所有Realm被调用之前
* 在调用Realm的getAuthenticationInfo 方法之前
* 在调用Realm的getAuthenticationInfo 方法之后
* 在所有Realm被调用之后

**认证策略的另外一项工作就是聚合所有Realm的结果信息封装至一个AuthenticationInfo实例中，并将此信息返回，以此作为Subject的身份信息。** 

Shiro有三种认证策略的具体实现： 
* `AtLeastOneSuccessfulStrategy` 	只要有一个（或更多）的Realm验证成功，那么认证将被视为成功
   `FirstSuccessfulStrategy` 	第一个Realm验证成功，整体认证将被视为成功，且后续Realm将被忽略
    `AllSuccessfulStrategy` 	所有Realm成功，认证才视为成功

**`ModularRealmAuthenticator`  内置的认证策略默认实现是第一种 `AtLeastOneSuccessfulStrategy`  方式，因为这种方式也是被广泛使用的一种认证策略。当然，你也可以通过配置文件定义你需要的策略**，如：

```java
[main]  
...  
authcStrategy = org.apache.shiro.authc.pam.FirstSuccessfulStrategy  
securityManager.authenticator.authenticationStrategy = $authcStrategy  
...  

```
##### Realm的顺序 
由刚才提到的认证策略，可以看到Realm在ModularRealmAuthenticator 里面的顺序对认证是有影响的。 
ModularRealmAuthenticator 会读取配置在SecurityManager里的Realm。当执行认证是，它会遍历Realm集合，对所有支持提交的token的Realm调用getAuthenticationInfo 。 
因此，Realm的顺序对你使用的认证策略结果有影响。

### 授权
授权即访问控制，它将判断用户在应用程序中对资源是否拥有相应的访问权限。 如，判断一个用户有查看页面的权限，编辑数据的权限，拥有某一按钮的权限，以及是否拥有打印的权限等等。

#### 授权的三要素 
授权有着三个核心元素：权限、角色和用户。 
##### 权限
权限是Apache Shiro安全机制最核心的元素。它在应用程序中明确声明了被允许的行为和表现。一个格式良好好的权限声明可以清晰表达出用户对该资源拥有的权限。 
大多数的资源会支持典型的CRUD操作（create,read,update,delete）,但是任何操作建立在特定的资源上才是有意义的。因此，权限声明的根本思想就是建立在资源以及操作上。 
而我们通过权限声明仅仅能了解这个权限可以在应用程序中做些什么，而不能确定谁拥有此权限。 
于是，我们就需要在应用程序中对用户和权限建立关联。 
通常的做法就是将权限分配给某个角色，然后将这个角色关联一个或多个用户。 
##### 权限声明及粒度 
Shiro权限声明通常是使用以冒号分隔的表达式。就像前文所讲，一个权限表达式可以清晰的指定资源类型，允许的操作，可访问的数据。同时，Shiro权限表达式支持简单的通配符，可以更加灵活的进行权限设置。 
下面以实例来说明权限表达式。 
* 可查询用户数据 
   `User:view` 
* 可查询或编辑用户数据 
   `User:view,edit` 
* 可对用户数据进行所有操作 
   `User:* 或 user`
* 可编辑id为123的用户数据 
   `User:edit:123`

##### 角色
Shiro支持两种角色模式：
1. 传统角色：一个角色代表着一系列的操作，当需要对某一操作进行授权验证时，只需判断是否是该角色即可。这种角色权限相对简单、模糊，不利于扩展。 
2. 权限角色：一个角色拥有一个权限的集合。授权验证时，需要判断当前角色是否拥有该权限。这种角色权限可以对该角色进行详细的权限描述，适合更复杂的权限设计。 

下面将详细描述对两种角色模式的授权实现。 
#### 授权实现
##### 角色验证
###### 编写代码
当需要验证用户是否拥有某个角色时，可以调用Subject 实例的 `hasRole*`方法验证。 
```java
Subject currentUser = SecurityUtils.getSubject();  
if (currentUser.hasRole("administrator")) {  
    //show the admin button  
} else {  
    //don't show the button?  Grey it out?  
}  
```
相关验证方法如下： 

| Subject方法                               | 描述                                    |
| ----------------------------------------- | --------------------------------------- |
| hasRole(String roleName)                  | 当用户拥有指定角色时，返回true          |
| hasRoles(List<String> roleNames)          | 按照列表顺序返回相应的一个boolean值数组 |
| hasAllRoles(Collection<String> roleNames) | 如果用户拥有所有指定角色时，返回true    |

##### 断言支持 
Shiro还支持以断言的方式进行授权验证。断言成功，不返回任何值，程序继续执行；断言失败时，将抛出异常信息。使用断言，可以使我们的代码更加简洁。 
```java
Subject currentUser = SecurityUtils.getSubject();  
//guarantee that the current user is a bank teller and  
//therefore allowed to open the account:  
currentUser.checkRole("bankTeller");  
openBankAccount();  
```

| Subject方法                              | 描述                         |
| ---------------------------------------- | ---------------------------- |
| checkRole(String roleName)               | 断言用户是否拥有指定角色     |
| checkRoles(Collection<String> roleNames) | 断言用户是否拥有所有指定角色 |
| checkRoles(String... roleNames)          | 对上一方法的方法重载         |

##### 权限验证
和上面的 role 差不多，验证权限字符串，方法名改了一下

##### 注解验证
一般我们都是使用注解的方式进行编码。
Shiro注解支持AspectJ、Spring、Google-Guice等，可根据应用进行不同的配置。 

相关注解：
1. `@RequiresAuthentication` 可以用户类/属性/方法，用于表明当前用户需是经过认证的用户。 
2. `@RequiresGuest` 表明该用户需为 `guest` 用户 
3. `@RequiresPermissions` 当前用户需拥有制定权限 
4. `@RequiresRoles` 当前用户需拥有制定角色 
5. `@RequiresUser` 当前用户需为已认证用户或已记住用户

#### Shiro授权的内部处理机制

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/learn-shiro/3.png)
1. 在应用程序中调用授权验证方法(Subject的 `isPermitted*` 或 `hasRole*` 等)
2. Sbuject的实例通常是 `DelegatingSubject` 类（或子类）的实例对象，在认证开始时，会委托应用程序设置的 `securityManager` 实例调用相应的 `isPermitted*` 或 `hasRole*` 方法。
3. 接下来SecurityManager会委托内置的 `Authorizer` 的实例（默认是 `ModularRealmAuthorizer` 类的实例，类似认证实例，它同样支持一个或多个Realm实例认证）调用相应的授权方法。 
4. 每一个Realm将检查是否实现了相同的 Authorizer 接口。然后，将调用Reaml自己的相应的授权验证方法。 

当使用多个Realm时，不同于认证策略处理方式，默认的授权处理过程中： 
1. 当调用Realm出现异常时，将立即抛出异常，结束授权验证。 
2. 只要有一个Realm验证成功，那么将认为授权成功，立即返回，结束认证。 

这篇文章主要是转载了 <a rel="external nofollow" target="_blank" href="http://kdboy.iteye.com/blog/1154644"> http://kdboy.iteye.com/blog/1154644</a> 下面的系列文章，进行学习，并且整理一部分内容，下面会继续学习 shiro 拦截器和自定义认证，包括 jwt 或者结合 redis 完成分布式认证。
