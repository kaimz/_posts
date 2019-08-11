---
title: 学习Spring Boot：（十三）配置 Shiro 权限认证
date: 2018-02-23 10:08:14
tags: [Spring Boot,shiro]
categories: 学习笔记
---

经过前面学习 Apache Shiro ，现在结合 Spring Boot 使用在项目里，进行相关配置。

<!--more-->

### 正文
#### 添加依赖
在 `pom.xml` 文件中添加 `shiro-spring` 的依赖：
```xml
		<dependency>
			<groupId>org.apache.shiro</groupId>
			<artifactId>shiro-spring</artifactId>
			<version>${shiro.version}</version>
		</dependency>
```
#### RBAC
RBAC [^1] 是基于角色的访问控制，权限与角色关联，给用户配置相关角色，来获取权限信息。

#### Shiro 配置
新建一个新的 `Shiro` 配置类 `ShiroConfig`:
```java
package com.wuwii.common.config;

import com.wuwii.module.sys.autho2.OAuth2Realm;
import org.apache.shiro.mgt.SecurityManager;
import org.apache.shiro.session.mgt.SessionManager;
import org.apache.shiro.spring.LifecycleBeanPostProcessor;
import org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor;
import org.apache.shiro.spring.web.ShiroFilterFactoryBean;
import org.apache.shiro.web.mgt.DefaultWebSecurityManager;
import org.apache.shiro.web.session.mgt.DefaultWebSessionManager;
import org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Apache Shiro 核心通过 Filter 来实现，就好像SpringMvc 通过DispachServlet 来主控制一样。
 * 既然是使用 Filter 一般也就能猜到，是通过URL规则来进行过滤和权限校验，
 * 所以我们需要定义一系列关于URL的规则和访问权限。
 *
 * @author KronChan
 * @version 1.0
 * @since <pre>2018/2/9 10:35</pre>
 */
@Configuration
public class ShiroConfig {

    @Bean
    public SessionManager sessionManager() {
        DefaultWebSessionManager sessionManager = new DefaultWebSessionManager();
        sessionManager.setSessionValidationSchedulerEnabled(true);
        sessionManager.setSessionIdCookieEnabled(true);
        return sessionManager;
    }

    /**
     * 注册安全管理,必须设置 SecurityManager
     *
     * @param oAuth2Realm    认证
     * @param sessionManager 缓存
     * @return
     */
    @Bean
    public SecurityManager securityManager(OAuth2Realm oAuth2Realm, SessionManager sessionManager) {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        // 可以添加多个认证，执行顺序是有影响的
        securityManager.setRealm(oAuth2Realm);
        securityManager.setSessionManager(sessionManager);
        return securityManager;
    }

    @Bean
    public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilter = new ShiroFilterFactoryBean();
        shiroFilter.setSecurityManager(securityManager);

        //自定义一个oauth2拦截器，不设置就是使用默认的拦截器
        /*Map<String, Filter> filters = new HashMap<>();
        filters.put("oauth2", new OAuth2Filter());
        shiroFilter.setFilters(filters);*/
        //拦截器
        //<!-- 过滤链定义，从上向下顺序执行，一般将 /**放在最为下边 -->
        //<!-- authc:所有url都必须认证通过才可以访问; anon:所有url都都可以匿名访问-->
        Map<String, String> filterMap = new LinkedHashMap<>();
        //配置退出过滤器,其中的具体的退出代码Shiro已经替我们实现了
        filterMap.put("/sys/logout", "logout");
        // 验证码
        filterMap.put("/sys/captcha.jpg", "anon");
        // 设置系统模块下访问需要权限
        filterMap.put("/sys/login", "anon");
        // 自定义的拦截
        //filterMap.put("/sys/**", "oauth2");
        filterMap.put("/sys/**", "authc");
        // 登陆的 url
        shiroFilter.setLoginUrl("/sys/login");
        // 登陆成功跳转的 url
        shiroFilter.setSuccessUrl("/");
        // 未授权的 url
        // shiroFilter.setUnauthorizedUrl("/login.html");
        //未授权界面;
        shiroFilter.setUnauthorizedUrl("/403");
        shiroFilter.setFilterChainDefinitionMap(filterMap);

        return shiroFilter;
    }

    /**
     * Shiro生命周期处理器
     * @return
     */
    @Bean
    public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
        return new LifecycleBeanPostProcessor();
    }

    /**
     * 开启Shiro的注解，
     * (如@RequiresRoles,@RequiresPermissions),需借助SpringAOP扫描使用Shiro注解的类,
     * 并在必要时进行安全逻辑验证 * 配置以下两个bean(DefaultAdvisorAutoProxyCreator(可选)和AuthorizationAttributeSourceAdvisor)即可实现此功能
     *
     * @return
     */
    @Bean
    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
        DefaultAdvisorAutoProxyCreator proxyCreator = new DefaultAdvisorAutoProxyCreator();
        proxyCreator.setProxyTargetClass(true);
        return proxyCreator;
    }

    /**
     * 开启 shiro aop注解支持.
     *
     * @param securityManager
     * @return
     */
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor advisor = new AuthorizationAttributeSourceAdvisor();
        advisor.setSecurityManager(securityManager);
        return advisor;
    }

    /**
     * 凭证匹配器
     * （由于我们的密码校验交给Shiro的SimpleAuthenticationInfo进行处理了
     * 所以我们需要修改下doGetAuthenticationInfo中的代码;
     * ）
     *  <b>需要在身份认证中添加 realm.setCredentialsMatcher(hashedCredentialsMatcher())</b>
     * @return
     */
    /*@Bean
    public HashedCredentialsMatcher hashedCredentialsMatcher() {
        HashedCredentialsMatcher hashedCredentialsMatcher = new HashedCredentialsMatcher();

        hashedCredentialsMatcher.setHashAlgorithmName("md5");//散列算法:这里使用MD5算法;
        hashedCredentialsMatcher.setHashIterations(2);//散列的次数，比如散列两次，相当于 md5(md5(""));

        return hashedCredentialsMatcher;
    }*/
}

```
Filter Chain定义说明：
1. 一个URL可以配置多个Filter，使用逗号分隔
2. 当设置多个过滤器时，全部验证通过，才视为通过
3. 部分过滤器可指定参数，如perms，roles

Shiro内置的FilterChain:

| Filter Name | Class                                                        |
| ----------- | ------------------------------------------------------------ |
| anon        | org.apache.shiro.web.filter.authc.AnonymousFilter            |
| authc       | org.apache.shiro.web.filter.authc.FormAuthenticationFilter   |
| authcBasic  | org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter |
| perms       | org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter |
| port        | org.apache.shiro.web.filter.authz.PortFilter                 |
| rest        | org.apache.shiro.web.filter.authz.HttpMethodPermissionFilter |
| roles       | org.apache.shiro.web.filter.authz.RolesAuthorizationFilter   |
| ssl         | org.apache.shiro.web.filter.authz.SslFilter                  |
| user        | org.apache.shiro.web.filter.authc.UserFilter                 |

* anon:所有url都都可以匿名访问
* authc: 需要认证才能进行访问
* user:配置记住我或认证通过可以访问
#### 自定义的拦截器(可选)
如果需要按照自己的需要定义一个 oauth2 的拦截器，则需要 继承 `AuthenticatingFilter` 实现几个方法即可。
```java
/**
 * oauth2过滤器
 */
public class OAuth2Filter extends AuthenticatingFilter {

    /**
     * logger
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(OAuth2Filter.class);

    @Override
    protected AuthenticationToken createToken(ServletRequest request, ServletResponse response) throws Exception {
        //获取请求token
        String token = getRequestToken((HttpServletRequest) request);
        if (StringUtils.isBlank(token)) {
            return null;
        }
        return new OAuth2Token(token);
    }

    /**
     *  shiro权限拦截核心方法 返回true允许访问resource，
     * @param request
     * @param response
     * @param mappedValue
     * @return
     */
    @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        return false;
    }

    /**
     * 当访问拒绝时是否已经处理了；
     * 如果返回true表示需要继续处理；
     * 如果返回false表示该拦截器实例已经处理完成了，将直接返回即可。
     * @param request
     * @param response
     * @return
     * @throws Exception
     */
    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
        //获取请求token，如果token不存在，直接返回401
        String token = getRequestToken((HttpServletRequest) request);
        if (StringUtils.isBlank(token)) {
            HttpServletResponse httpResponse = (HttpServletResponse) response;
            ((HttpServletResponse) response).setStatus(401);
            response.getWriter().print("没有权限，请联系管理员授权");
            return false;
        }
        return executeLogin(request, response);
    }

    /**
     * 鉴定失败，返回错误信息
     * @param token
     * @param e
     * @param request
     * @param response
     * @return
     */
    @Override
    protected boolean onLoginFailure(AuthenticationToken token, AuthenticationException e, ServletRequest request, ServletResponse response) {
        try {
            ((HttpServletResponse) response).setStatus(401);
            response.getWriter().print("没有权限，请联系管理员授权");
        } catch (IOException e1) {
            LOGGER.error(e1.getMessage(), e1);
        }
        return false;
    }

    /**
     * 获取请求的token
     */
    private String getRequestToken(HttpServletRequest httpRequest) {
        //从header中获取token
        String token = httpRequest.getHeader("token");
        //如果header中不存在token，则从参数中获取token
        if (StringUtils.isBlank(token)) {
            return httpRequest.getParameter("token");
        }
        // 还可以实现从 cookie 获取
        Cookie[] cookies = httpRequest.getCookies();
        if(null==cookies||cookies.length==0){
            return null;
        }
        for (Cookie cookie : cookies) {
            if (cookie.getName().equals("token")) {
                token = cookie.getValue();
                continue;
            }
        }
        return token;
    }
}
```
具体实现可以参考我的上篇文章 《Apache Shiro的拦截器和认证》

#### 认证实现
Shiro的认证过程最终会交由Realm执行，这时会调用Realm的 `getAuthenticationInfo(token)` 方法。
该方法主要执行以下操作:
1. 检查提交的进行认证的令牌信息
2. 根据令牌信息从数据源(通常为数据库)中获取用户信息
3. 对用户信息进行匹配验证。
4. 验证通过将返回一个封装了用户信息的AuthenticationInfo实例。
5. 验证失败则抛出AuthenticationException异常信息。

而在我们的应用程序中要做的就是自定义一个Realm类，继承 `AuthorizingRealm` 抽象类，重载 `doGetAuthenticationInfo ()`，重写获取用户信息的方法。

```java
@Component
public class OAuth2Realm extends AuthorizingRealm {
    @Resource
    private ShiroService shiroService;
    @Resource
    private SysUserService sysUserService;


    /**
     * 此方法调用  hasRole,hasPermission的时候才会进行回调.
     *
     * 权限信息.(授权):
     * 1、如果用户正常退出，缓存自动清空；
     * 2、如果用户非正常退出，缓存自动清空；
     * 3、如果我们修改了用户的权限，而用户不退出系统，修改的权限无法立即生效。
     * （需要手动编程进行实现；放在service进行调用）
     * 在权限修改后调用realm中的方法，realm已经由spring管理，所以从spring中获取realm实例，
     * 调用clearCached方法；
     * :Authorization 是授权访问控制，用于对用户进行的操作授权，证明该用户是否允许进行当前操作，如访问某个链接，某个资源文件等。
     *
     *
     * 当没有使用缓存的时候，不断刷新页面的话，这个代码会不断执行，
     * 当其实没有必要每次都重新设置权限信息，所以我们需要放到缓存中进行管理；
     * 当放到缓存中时，这样的话，doGetAuthorizationInfo就只会执行一次了，
     * 缓存过期之后会再次执行。
     *
     * @param principals
     * @return
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        SysUserEntity user =(SysUserEntity) (principals.getPrimaryPrincipal());;

        // 获取该用户权限列表
        Set<String> permsSet = shiroService.getUserPermissions(user.getId());

        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        info.setStringPermissions(permsSet);
        return info;
    }

    /**
     * 认证回调函数,登录时调用
     * 首先根据传入的用户名获取User信息；然后如果user为空，那么抛出没找到帐号异常UnknownAccountException；
     * 如果user找到但锁定了抛出锁定异常LockedAccountException；最后生成AuthenticationInfo信息，
     * 交给间接父类AuthenticatingRealm使用CredentialsMatcher进行判断密码是否匹配，
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        UsernamePasswordToken usernamePasswordToken = (UsernamePasswordToken) token;
        SysUserEntity user = sysUserService.queryByUsername(usernamePasswordToken.getUsername());
        //账号不存在、密码错误
        if (user == null) {
            throw new KCException("账号或密码不正确");
        }

        // 交给 shiro 自己去验证，
        // 明文验证
        return new SimpleAuthenticationInfo(
                user, // 存入凭证的信息，登陆成功后可以使用 SecurityUtils.getSubject().getPrincipal();在任何地方使用它
                user.getPassword(),
                getName());

        // 加密的方式
        // 交给AuthenticatingRealm使用CredentialsMatcher进行密码匹配，如果觉得人家的不好可以自定义实现
        /*return new SimpleAuthenticationInfo(
                user,
                user.getPassword(),
                ByteSource.Util.bytes(user.getSalt()), // 加盐，可以注册凭证匹配器 HashedCredentialsMatcher 告诉它怎么加密的
                getName());*/

    }
}
```
实现上面两个方法即可完成身份验证和权限验证。

#### 登陆实现
```java
    @PostMapping("/login")
    @ApiOperation("系统登陆")
    public ResponseEntity<String> login(@RequestBody SysUserLoginForm userForm) {
        String kaptcha = ShiroUtils.getKaptcha(Constants.KAPTCHA_SESSION_KEY);
        if (!userForm.getCaptcha().equalsIgnoreCase(kaptcha)) {
            throw new KCException("验证码不正确！");
        }
        UsernamePasswordToken token = new UsernamePasswordToken(userForm.getUsername(), userForm.getPassword());
        Subject currentUser = SecurityUtils.getSubject();
        currentUser.login(token);
        
         //账号锁定
        if (getUser().getStatus() == SysConstant.SysUserStatus.LOCK) {
            throw new KCException("账号已被锁定,请联系管理员");
        }
        return ResponseEntity.status(HttpStatus.OK).body("登陆成功！");
    }
```
#### 权限验证
```java
    @ApiOperation("用于测试，查询")
    @ApiImplicitParam(name = "string", value = "id", dataType = "string")
    @RequiresPermissions("sys:user:list1")
    @GetMapping()
    public ResponseEntity<List<SysUserEntity>> query(@CustomValid String string) {
        return new ResponseEntity<>(sysUserService.query(new SysUserEntity()), OK);
    }
```
简单测试一个例子，`sys:user:list1` 多加一个 `1` 肯定会验证失败，查看程序会做什么，它会去我们定义的 Realm 中的 `doGetAuthorizationInfo(PrincipalCollection principals)` 方法中，执行查询该用户的所有权限。
验证失败后最后程序结果如下：
```java
Caused by: org.apache.shiro.authz.AuthorizationException: Not authorized to invoke method: public org.springframework.http.ResponseEntity com.wuwii.module.sys.controller.SysUserController.query(java.lang.String)
	at org.apache.shiro.authz.aop.AuthorizingAnnotationMethodInterceptor.assertAuthorized(AuthorizingAnnotationMethodInterceptor.java:90)
	... 77 common frames omitted
```
##### 权限注解
```java
@RequiresAuthentication  
表示当前Subject已经通过login进行了身份验证；即Subject. isAuthenticated()返回true。 

@RequiresUser  
表示当前Subject已经身份验证或者通过记住我登录的。

@RequiresGuest  
表示当前Subject没有身份验证或通过记住我登录过，即是游客身份。
  
@RequiresRoles(value={“admin”, “user”}, logical= Logical.AND)  
表示当前Subject需要角色admin和user。

@RequiresPermissions (value={“user:a”, “user:b”}, logical= Logical.OR)  
表示当前Subject需要权限user:a或user:b。  
```

### 参考资料
* <a rel="external nofollow" target="_blank" href="https://segmentfault.com/a/1190000008847948"> Spring Boot [集成-Shiro]</a>
* <a rel="external nofollow" target="_blank" href="http://http://412887952-qq-com.iteye.com/blog/2299777"> 39.2. Spring Boot Shiro权限管理【从零开始学Spring Boot】</a>

[^1]: Role-Based Access Control 
