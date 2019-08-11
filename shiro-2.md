---
title: Apache Shiro的拦截器和认证
date: 2018-02-23 10:08:19
tags: [shiro]
categories: 学习笔记
---

### Apache Shiro的拦截器
#### FormAuthenticationFilter 
`FormAuthenticationFilter` 是shiro 包自带的一个拦截器，继承了 `AuthenticatingFilter` 这个抽象类，再上一级就是 `AuthenticationFilter` 。

<!--more-->

1. 在进入拦截器之前，会进入 `onPreHandle` 方法中：
```java
//只要isAccessAllowed或者onAccessDenied有一个为真，就返回true，继续执行
判断这两个方法需不需要执行，按顺序
public boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
    return isAccessAllowed(request, response, mappedValue) || onAccessDenied(request, response, mappedValue);
}
```
2. `isAccessAllowed` 方法验证请求是否认证
    如果认证结果为 `false`，根据上面的流程控制，则进入 `onAccessDenied` 方法中进行认证。 
```java
/**如果满足（1）.当前的subject是被认证过的。
（2）.用户请求的不是登录页面，但是在定义该过滤器时，使用了PERMISSIVE=”permissive”参数。
只要满足两个条件的一个即可允许操作
*/
@Override
protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        return super.isAccessAllowed(request, response, mappedValue) ||(!isLoginRequest(request, response) && isPermissive(mappedValue));
    }


//其实就是判断当前的subject是不是被认证过的
protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        Subject subject = getSubject(request, response);
        return subject.isAuthenticated();
    }

//判断当前的请求是不是登陆请求
protected boolean isLoginRequest(ServletRequest request, ServletResponse response) {
        return pathsMatch(getLoginUrl(), request);
    }

//判断当前的拦截器是不是配置了PERMISSIVE=”permissive”参数，如果配置了就可以通过
protected boolean isPermissive(Object mappedValue) {
        if(mappedValue != null) {
            String[] values = (String[]) mappedValue;
            return Arrays.binarySearch(values, PERMISSIVE) >= 0;
        }
        return false;
    }
```
3. `onAccessDenied` 
```java
protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
    if (isLoginRequest(request, response)) {
        if (isLoginSubmission(request, response)) {
            if (log.isTraceEnabled()) {
                log.trace("Login submission detected.  Attempting to execute login.");
            }
            return executeLogin(request, response);
        } else {
            if (log.isTraceEnabled()) {
                log.trace("Login page view.");
            }
            //allow them to see the login page ;)
            return true;
        }
    } else {
        if (log.isTraceEnabled()) {
            log.trace("Attempting to access a path which requires authentication.  Forwarding to the " +
                    "Authentication url [" + getLoginUrl() + "]");
        }

        saveRequestAndRedirectToLogin(request, response);
        return false;
    }
}


//判断当前的其实是不是一个HTTP的POST请求
protected boolean isLoginSubmission(ServletRequest request, ServletResponse response) {
        return (request instanceof HttpServletRequest) && WebUtils.toHttp(request).getMethod().equalsIgnoreCase(POST_METHOD);
    }

//经过前面的判断是不是POST请求后，程序就为我们创建一个token，但是我们并没有传入userName和password在创建的时候就会抛出异常了
protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
        AuthenticationToken token = createToken(request, response);
        if (token == null) {
            String msg = "createToken method implementation returned null. A valid non-null AuthenticationToken " +
                    "must be created in order to execute a login attempt.";
            throw new IllegalStateException(msg);
        }
        try {
            Subject subject = getSubject(request, response);
            subject.login(token);
            return onLoginSuccess(token, subject, request, response);
        } catch (AuthenticationException e) {
            return onLoginFailure(token, e, request, response);
        }
    }


/**saveRequest就是把一个request保存在session中，redirectToLogin这里就是返回到设置的登录页面，在开头自定义的过滤器中就是重载了这个函数，在实际项目中，一般都会重载这个函数，比方说返回到指定的页面*/
protected void saveRequestAndRedirectToLogin(ServletRequest request, ServletResponse response) throws IOException {
        saveRequest(request);
        redirectToLogin(request, response);
    }
```
可以看到如果前面 `isAccessAllowed` 检查是否认证过的请求，则进入 `onAccessDenied` 进行认证的操作。

#### 自定义认证拦截器
继承 `FormAuthenticationFilter` 或者 `AuthenticatingFilter`，并改写核心认证逻辑即可，这个需要阅读下 `FormAuthenticationFilter的源码` 的源码，这个时候思路就很清晰了。

### 认证
#### AuthorizingRealm 抽象类
Shiro的认证过程最终会交由Realm执行，这时会调用Realm的 `getAuthenticationInfo(token)` 方法。
该方法主要执行以下操作:
1. 检查提交的进行认证的令牌信息
2. 根据令牌信息从数据源(通常为数据库)中获取用户信息
3. 对用户信息进行匹配验证。
4. 验证通过将返回一个封装了用户信息的 `AuthenticationInfo` 实例。
5. 验证失败则抛出 `AuthenticationException` 异常信息。
#### 拓展认证
需要自己定义一个 realm 类，来继承 `AuthorizingRealm` 抽象类，然后重写相关需要修改的方法即可。
一般重写这两个方法即可：
```java
    /**
     * 获取授权信息
     PrincipalCollection是一个身份集合，因为我们现在就一个Realm，所以直接调用getPrimaryPrincipal得到之前传入的用户名即可；然后根据用户名调用UserService接口获取角色及权限信息。
     */
protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals)

/**
 * 获取身份验证信息
 * 认证回调函数,登录时调用
 * 首先根据传入的用户名获取User信息；然后如果user为空，那么抛出没找到帐号异常UnknownAccountException；
 * 如果user找到但锁定了抛出锁定异常LockedAccountException；最后生成AuthenticationInfo信息，
 * 交给间接父类AuthenticatingRealm使用CredentialsMatcher进行判断密码是否匹配，
 * 如果不匹配将抛出密码错误异常IncorrectCredentialsException；
 */
protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException
```
**UserRealm父类AuthorizingRealm将获取Subject相关信息分成两步：获取身份验证信息（doGetAuthenticationInfo）及授权信息（doGetAuthorizationInfo）**；
1. doGetAuthenticationInfo获取身份验证相关信息：首先根据传入的用户名获取User信息；然后如果user为空，那么抛出没找到帐号异常UnknownAccountException；如果user找到但锁定了抛出锁定异常LockedAccountException；最后生成AuthenticationInfo信息，交给间接父类AuthenticatingRealm使用CredentialsMatcher进行判断密码是否匹配，如果不匹配将抛出密码错误异常IncorrectCredentialsException；另外如果密码重试此处太多将抛出超出重试次数异常ExcessiveAttemptsException；在组装SimpleAuthenticationInfo信息时，需要传入：身份信息（用户名）、凭据（密文密码）、盐（username+salt），CredentialsMatcher使用盐加密传入的明文密码和此处的密文密码进行匹配。
2. doGetAuthorizationInfo获取授权信息：PrincipalCollection是一个身份集合，因为我们现在就一个Realm，所以直接调用getPrimaryPrincipal得到之前传入的用户名即可；然后根据用户名调用UserService接口获取角色及权限信息。

### AuthenticationToken
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/learn-shiro/5.png)
AuthenticationToken用于收集用户提交的身份（如用户名）及凭据（如密码）：
```java
public interface AuthenticationToken extends Serializable {  
    Object getPrincipal(); //身份  
    Object getCredentials(); //凭据  
}   
```
* 扩展接口 `RememberMeAuthenticationToken`：提供了 `boolean isRememberMe()` 现“记住我”的功能；
* 扩展接口是 `HostAuthenticationToken`：提供了 `String getHost()` 方法用于获取用户“主机”的功能。

Shiro提供了一个直接拿来用的UsernamePasswordToken，用于实现用户名/密码Token组，另外其实现了RememberMeAuthenticationToken和HostAuthenticationToken，可以实现记住我及主机验证的支持。
#### 自己实现 AuthenticationToken
主要实现能够实现获取用户的身份信息的 token :
```java
public class OAuth2Token implements AuthenticationToken {
    private String token;

    public OAuth2Token(String token) {
        this.token = token;
    }

    @Override
    public String getPrincipal() {
        return token;
    }

    @Override
    public Object getCredentials() {
        return token;
    }
}
```
### AuthenticationInfo
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/learn-shiro/6.png)

AuthenticationInfo有两个作用：
1. 如果Realm是AuthenticatingRealm子类，则提供给AuthenticatingRealm内部使用的CredentialsMatcher进行凭据验证；（如果没有继承它需要在自己的Realm中自己实现验证）；
2. 提供给SecurityManager来创建Subject（提供身份信息）；
### PrincipalCollection
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/learn-shiro/7.png)
因为我们可以在Shiro中同时配置多个Realm，所以呢身份信息可能就有多个；因此其提供了PrincipalCollection用于聚合这些身份信息：
```java
public interface PrincipalCollection extends Iterable, Serializable {  
    Object getPrimaryPrincipal(); //得到主要的身份  
    <T> T oneByType(Class<T> type); //根据身份类型获取第一个  
    <T> Collection<T> byType(Class<T> type); //根据身份类型获取一组  
    List asList(); //转换为List  
    Set asSet(); //转换为Set  
    Collection fromRealm(String realmName); //根据Realm名字获取  
    Set<String> getRealmNames(); //获取所有身份验证通过的Realm名字  
    boolean isEmpty(); //判断是否为空  
}   
```
因为PrincipalCollection聚合了多个，此处最需要注意的是getPrimaryPrincipal，如果只有一个Principal那么直接返回即可，如果有多个Principal，则返回第一个（因为内部使用Map存储，所以可以认为是返回任意一个）；oneByType / byType根据凭据的类型返回相应的Principal；fromRealm根据Realm名字（每个Principal都与一个Realm关联）获取相应的Principal。

如果我们还需要实现别的类型的多个认证，具体参考文章第二篇，我先不研究了。
### AuthorizationInfo
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/learn-shiro/8.png)
AuthorizationInfo用于聚合授权信息的：
```java
public interface AuthorizationInfo extends Serializable {  
    Collection<String> getRoles(); //获取角色字符串信息  
    Collection<String> getStringPermissions(); //获取权限字符串信息  
    Collection<Permission> getObjectPermissions(); //获取Permission对象信息  
}   
```
当我们使用AuthorizingRealm时，如果身份验证成功，在进行授权时就通过 `doGetAuthorizationInfo` 方法获取角色/权限信息用于授权验证。

Shiro提供了一个实现 `SimpleAuthorizationInfo`，大多数时候使用这个即可。

### Subject
Subject是Shiro的核心对象，基本所有身份验证、授权都是通过Subject完成。
对于Subject的构建一般没必要我们去创建；一般通过 `SecurityUtils.getSubject()` 获取;

对于Subject我们一般这么使用：
1. 身份验证（login）
2. 授权（`hasRole*/isPermitted*`或`checkRole*/checkPermission*`）
3. 将相应的数据存储到会话（Session）
4. 切换身份（RunAs）/多线程身份传播
5. 退出 logout

### 参考文章
* <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/makecontral/article/details/77387026">Apache Shiro源码 拦截器过程</a>
* <a rel="external nofollow" target="_blank" href="http://jinnianshilongnian.iteye.com/blog/2022468">第六章 Realm及相关对象——《跟我学Shiro》</a>
