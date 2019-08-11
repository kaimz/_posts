---
title: 学习Spring Boot：（二十八）Spring Security 权限认证
date: 2018-04-22 22:12:03
tags: [Spring Boot]
categories: 学习笔记
---

### 前言

主要实现 `Spring Security` 的安全认证，结合 `RESTful API` 的风格，使用无状态的环境。

主要实现是通过请求的 URL ，通过过滤器来做不同的授权策略操作，为该请求提供某个认证的方法，然后进行认证，授权成功返回授权实例信息，供服务调用。

<!--more-->

基于Token的身份验证的过程如下:

1. 用户通过用户名和密码发送请求。
2. 程序验证。
3. 程序返回一个签名的token 给客户端。
4. 客户端储存token,并且每次用于每次发送请求。
5. 服务端验证token并返回数据。

每一次请求都需要token，所以每次请求都会去验证用户身份，所以这里必须要使用缓存，

### 基本使用

#### 加入相关依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

#### 了解基础配置

##### 认证的基本信息

```java
public interface UserDetails extends Serializable {

    //返回分配给用户的角色列表
    Collection<? extends GrantedAuthority> getAuthorities();
    
    //返回密码
    String getPassword();

    //返回帐号
    String getUsername();

    // 账户是否未过期
    boolean isAccountNonExpired();

    // 账户是否未锁定
    boolean isAccountNonLocked();

    // 密码是否未过期
    boolean isCredentialsNonExpired();

    // 账户是否激活
    boolean isEnabled();
}
```

##### 获取基本信息

```java
// 根据用户名查找用户的信息
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

我们只要实现这个扩展，就能够自定义方式获取认证的基本信息

##### WebSecurityConfigurerAdapter

 `WebSecurityConfigurerAdapter` 提供了一种便利的方式去创建 `WebSecurityConfigurer`的实例，只需要重写 `WebSecurityConfigurerAdapter` 的方法，即可配置拦截什么URL、设置什么权限等安全控制。

下面是主要会是要到的几个配置：

```java
    /**
     * 主要是对身份认证的设置
     * @param auth
     * @throws Exception
     */  
	@Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		this.disableLocalConfigureAuthenticationBldr = true;
    }
    /**
     * 复写这个方法来配置 {@link HttpSecurity}. 
     * 通常，子类不能通过调用 super 来调用此方法，因为它可能会覆盖其配置。 默认配置为：
     * 
     */
    protected void configure(HttpSecurity http) throws Exception {
        logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");

        http
            .authorizeRequests()
                .anyRequest().authenticated()
                .and()
            .formLogin().and()
            .httpBasic();
    }

	/**
	 * Override this method to configure {@link WebSecurity}. For example, if you wish to
	 * ignore certain requests.
	 * 主要是对某些 web 静态资源的设置
	 */
	public void configure(WebSecurity web) throws Exception {
	}
```



### 认证流程

阅读源码了解。

![Spring Security.jpg](https://upload-images.jianshu.io/upload_images/3424642-7418a70abdfc7287.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### AbstractAuthenticationProcessingFilter.doFilter

```java
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;
		// 判断是否是需要验证方法（是否是登陆的请求），不是的话直接放过
        if (!requiresAuthentication(request, response)) {
            chain.doFilter(request, response);
            return;
        }
        // 登陆的请求开始进行验证
        Authentication authResult;
        try {
            // 开始认证，attemptAuthentication在 UsernamePasswordAuthenticationFilter 中实现
            authResult = attemptAuthentication(request, response);
            // return null 认证失败
            if (authResult == null) {
                return;
            }
		// 篇幅问题，中间很多代码删了
        successfulAuthentication(request, response, chain, authResult);
    }
```

#### UsernamePasswordAuthenticationFilter.attemptAuthentication

```java
// 接收并解析用户登陆信息，为已验证的用户返回一个已填充的身份验证令牌，表示成功的身份验证，
// 如果身份验证过程失败，就抛出一个AuthenticationException
public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException {
		if (postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException(
					"Authentication method not supported: " + request.getMethod());
		}
        // 方法将 request 中的 username 和 password 生成 UsernamePasswordAuthenticationToken 对象，用于 AuthenticationManager 的验证
		String username = obtainUsername(request);
		String password = obtainPassword(request);
		if (username == null) {
			username = "";
		}
		if (password == null) {
			password = "";
		}
		username = username.trim();
		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, password);
		// Allow subclasses to set the "details" property
		setDetails(request, authRequest);
		return this.getAuthenticationManager().authenticate(authRequest);
	}
```

#### ProviderManager.authenticate

验证 Authentication 对象（里面包含着验证对象）

1. 如果有多个 AuthenticationProvider 支持验证传递过来的Authentication 对象，那么由第一个来确定结果，覆盖早期支持AuthenticationProviders 所引发的任何可能的AuthenticationException。 成功验证后，将不会尝试后续的AuthenticationProvider。
2. 如果最后所有的 AuthenticationProviders 都没有成功验证 Authentication 对象，将抛出 AuthenticationException。

最后它调用的是 `Authentication result = provider.authenticate(authentication);`

只要我们自定义  `AuthenticationProvider` 就能完成自定义认证。

### 动手实现安全框架

#### 使用的依赖

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>commons-lang</groupId>
            <artifactId>commons-lang</artifactId>
            <version>2.6</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.7.0</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.7.0</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.7.0</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>net.sf.ehcache</groupId>
            <artifactId>ehcache</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

#### 数据表关系

![img](https://zqnight.gitee.io/kaimz.github.io/image/hexo/springboot-security/1.png)

##### User

```java
@Data
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false, length = 50)
    private String username;

    @Column(nullable = false)
    private String password;

    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private Date createDate;

    @OneToMany(targetEntity = UserRole.class, mappedBy = "userId", fetch = FetchType.EAGER) // mappedBy 只有在双向关联的时候设置，表示关系维护的一端，否则会生成中间表A_B
    @org.hibernate.annotations.ForeignKey(name = "none") // 注意这里不能使用 @JoinColumn 不然会生成外键
    private Set<UserRole> userRoles;
}
```

##### Role

```java
@Entity
@Data
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false, unique = true)
    private String name;
}
```

##### UserRole

```java
@Entity
@Data
public class UserRole {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(length = 50, nullable = false)
    private Long userId;

    @ManyToOne(targetEntity = Role.class)
    @JoinColumn(name = "roleId", nullable = false, foreignKey = @ForeignKey(name = "none", value = ConstraintMode.NO_CONSTRAINT))
    private Role role;
}
```

#### 流程实现

认证流程：
![img](https://zqnight.gitee.io/kaimz.github.io/image/hexo/springboot-security/3.png)

##### JWT

我使用的是服务端无状态的token 交换的形式，所以引用的是 jwt，首先实现 jwt：

```java
# jwt 配置
jwt:
  # 加密密钥
  secret: 61D73234C4F93E03074D74D74D1E39D9 #blog.wuwii.com
  # token有效时长
  expire: 7 # 7天，单位天
  # token 存在 header 中的参数
  header: token
  
@ConfigurationProperties(prefix = "jwt")
@Data
public class JwtUtil {
    /**
     * 密钥
     */
    private String secret;
    /**
     * 有效期限
     */
    private int expire;
    /**
     * 存储 token
     */
    private String header;

    /**
     * 生成jwt token
     *
     * @param username
     * @return token
     */
    public String generateToken(String username) {
        Date nowDate = new Date();

        return Jwts.builder()
                .setHeaderParam("typ", "JWT")
                // 后续获取 subject 是 username
                .setSubject(username)
                .setIssuedAt(nowDate)
                .setExpiration(DateUtils.addDays(nowDate, expire))
                // 这里我采用的是 HS512 算法
                .signWith(SignatureAlgorithm.HS512, secret)
                .compact();
    }

    /**
     * 解析 token，
     * 利用 jjwt 提供的parser传入秘钥，
     *
     * @param token token
     * @return 数据声明 Map<String, Object>
     */
    private Claims getClaimByToken(String token) {
        try {
            return Jwts.parser()
                    .setSigningKey(secret)
                    .parseClaimsJws(token)
                    .getBody();
        } catch (Exception e) {
            return null;
        }
    }

    /**
     * token是否过期
     *
     * @return true：过期
     */
    public boolean isTokenExpired(Date expiration) {
        return expiration.before(new Date());
    }

    public String getUsernameFromToken(String token) {
        if (StringUtils.isBlank(token)) {
            throw new KCException("无效 token", HttpStatus.UNAUTHORIZED.value());
        }
        Claims claims = getClaimByToken(token);
        if (claims == null || isTokenExpired(claims.getExpiration())) {
            throw new KCException(header + "失效，请重新登录", HttpStatus.UNAUTHORIZED.value());
        }
        return claims.getSubject();
    }
}
```

##### 实现 UserDetails 和 UserDetailsService

![img](https://zqnight.gitee.io/kaimz.github.io/image/hexo/springboot-security/2.png)

###### 实现 UserDetails

```java
public class UserDetailsImpl implements UserDetails {
    private User user;

    public UserDetailsImpl(User user) {
        this.user = user;
    }

    /**
     * 获取权限信息
     * @return
     */
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Set<UserRole> userRoles = user.getUserRoles();
        List<GrantedAuthority> auths = new ArrayList<>(userRoles.size());
        userRoles.parallelStream().forEach(userRole -> {
            // 默认ROLE_  为前缀，可以更改
            auths.add(new SimpleGrantedAuthority("ROLE_" + userRole.getRole().getName()));
        });
        return auths;
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    // 账户是否未过期
    @JsonIgnore
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    // 账户是否未锁定
    @JsonIgnore
    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    // 密码是否未过期
    @JsonIgnore
    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    // 账户是否激活
    @JsonIgnore
    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

###### 实现 UserDetailsService

```java
@Slf4j
@CacheConfig(cacheNames = "users")
public class UserDetailServiceImpl implements UserDetailsService {

    @Autowired
    private UserDao userDao;

    @Override
    @Cacheable
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userDao.findByUsername(username);
        if (user == null) {
            throw new UsernameNotFoundException("Username is not valid.");
        }
        log.debug("The User is {}", user);
        return SecurityModelFactory.create(user);
    }
}
```

###### SecurityModelFactory

转换 UserDetails 的工厂类

```java
public class SecurityModelFactory {
    public static UserDetails create(User user) {
        return new UserDetailsImpl(user);
    }
}
```

##### 授权认证

###### 登陆过滤器

```java
public class LoginFilter extends UsernamePasswordAuthenticationFilter {
    @Autowired
    private JwtUtil jwtUtil;

    /**
     * 过滤，我目前使用的是默认的，可以自己看源码按需求更改
     */
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        // todo 在这里可以按需求进行过滤，根据源码来修改扩展非常方便
        super.doFilter(request, response, chain);
    }

    /**
     * 如果需要进行登陆认证，会在这里进行预处理
     */
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request,
                                                HttpServletResponse response) throws AuthenticationException {
        // todo 在登陆认证的时候，可以做些其他的验证操作，比如验证码
        return super.attemptAuthentication(request, response);
    }

    /**
     * 登陆成功调用，返回 token
     */
    @Override
    protected void successfulAuthentication(HttpServletRequest request,
                                            HttpServletResponse response,
                                            FilterChain chain, Authentication authResult) throws IOException {
        String token = jwtUtil.generateToken(authResult.getName());
        response.setStatus(HttpStatus.OK.value());
        response.getWriter().print(token);
    }
}
```

1. 首先会进入 `doFilter` 方法中，这里可以自定义定义过滤；
2. 然后如果是`登陆`的请求，会进入 `attemptAuthentication` 组装登陆信息，并且进行登陆认证；
3. 如果登陆成功，会调用 `successfulAuthentication`方法。

###### 登陆验证

```java
@Slf4j
public class CustomAuthenticationProvider implements AuthenticationProvider {
    @Autowired
    private UserDetailsService userDetailsService;

    @Autowired
    private PasswordEncoder passwordEncoder;

    /**
     * 验证登录信息,若登陆成功,设置 Authentication
     */
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = authentication.getName();
        String password = (String) authentication.getCredentials();
        UserDetails user = userDetailsService.loadUserByUsername(username);
        if (passwordEncoder.matches(password, user.getPassword())) {
            Collection<? extends GrantedAuthority> authorities = user.getAuthorities();
            return new UsernamePasswordAuthenticationToken(username, password, authorities);
        }
        throw new BadCredentialsException("The password is not correct.");
    }

    /**
     * 当前 Provider 是否支持对该类型的凭证提供认证服务
     */
    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.equals(authentication);
    }
}
```

我们自己定义的 `AuthenticationProvider ` 主要是实现前面经过过滤器封装的认证对象 `UsernamePasswordAuthenticationToken ` 进行解析认证，

如果认证成功 就给改 `UsernamePasswordAuthenticationToken` 设置对应的权限,然后返回 `Authentication`

1. 获得认证的信息；
2. 去数据库查询信息，获取密码解密验证认证信息；
3. 认证成功，设置权限信息，返回 `Authentication`，失败抛出异常。

##### JWT 拦截器

```java
/**
 * token 校验
 * BasicAuthenticationFilter 滤器负责处理任何具有HTTP请求头的请求的请求，
 * 以及一个基本的身份验证方案和一个base64编码的用户名:密码令牌。
 */
public class JwtAuthenticationFilter extends BasicAuthenticationFilter {
    @Autowired
    private JwtUtil jwtUtil;
    @Autowired
    private UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(AuthenticationManager authenticationManager) {
        super(authenticationManager);
    }

    /**
     * 在此方法中检验客户端请求头中的token,
     * 如果存在并合法,就把token中的信息封装到 Authentication 类型的对象中,
     * 最后使用  SecurityContextHolder.getContext().setAuthentication(authentication); 改变或删除当前已经验证的 pricipal
     *
     * @param request
     * @param response
     * @param chain
     * @throws IOException
     * @throws ServletException
     */
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {

        String token = request.getHeader(jwtUtil.getHeader());

        //判断是否有token
        if (token == null) {
            chain.doFilter(request, response);
            return;
        }
        // 通过token 获取账户信息，并且存入到将身份信息存放在安全系统的上下文。
        UsernamePasswordAuthenticationToken authenticationToken = getAuthentication(token);
        SecurityContextHolder.getContext().setAuthentication(authenticationToken);
        //放行
        chain.doFilter(request, response);
    }

    /**
     * 解析token中的信息
     */
    private UsernamePasswordAuthenticationToken getAuthentication(String token) {
        String username = jwtUtil.getUsernameFromToken(token);
        UserDetails userDetails = userDetailsService.loadUserByUsername(username);
        if (username != null) {
            return new UsernamePasswordAuthenticationToken(username, null, userDetails.getAuthorities());
        }
        return null;
    }
}
```

1. 请求进入 `doFilterInternal` 方法中，对请求是否带`token`进行判断，
2. 如果没有token，则直接放行请求；
3. 如果有 token，则解析它的 post；

##### 配置权限和相关设置

自定义配置  Spring Security 配置类 `WebSecurityConfig`，进项相关配置，并且将所需要的类注入到系统中。

```java
@Configuration
@EnableWebSecurity // 开启 Security
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true, jsr250Enabled = true)
//jsr250Enabled有三种注解，分别是@RolesAllowed,@PermitAll,@DenyAll,功能跟名字一样，
// securedEnabled 开启注解
// prePostEnabled  类似用的最多的是 @PreAuthorize
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    public JwtUtil jwtUtil() {
        return new JwtUtil();
    }

    /**
     * 注入 LoginFilter 时候需要，注入 authenticationManager
     */
    @Bean
    public LoginFilter loginFilter() throws Exception {
        LoginFilter loginFilter = new LoginFilter();
        loginFilter.setAuthenticationManager(authenticationManager());
        return loginFilter;
    }

    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() throws Exception {
        return new JwtAuthenticationFilter(authenticationManager());
    }
    @Bean
    public UserDetailsService customService() {
        return new UserDetailServiceImpl();
    }

    /**
     * 认证 AuthenticationProvider
     */
    @Bean
    public AuthenticationProvider authenticationProvider() {
        return new CustomAuthenticationProvider();
    }

    /**
     * BCrypt算法免除存储salt
     * BCrypt算法将salt随机并混入最终加密后的密码，验证时也无需单独提供之前的salt，从而无需单独处理salt问题。
     * @return
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(5);
    }

    /**
     * 主要是对身份验证的设置
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {

        auth
                // 注入身份的 Bean
                .authenticationProvider(authenticationProvider())
                .userDetailsService(userDetailsService())
                // 默认登陆的加密，自定义登陆的时候无效
                .passwordEncoder(passwordEncoder());
        // 在内存中设置固定的账户密码以及身份信息
        /*auth
                .inMemoryAuthentication().withUser("user").password("password").roles("USER").and()
                .withUser("admin").password("password").roles("USER", "ADMIN");*/
    }

    /**
     *
     * @param http
     * @throws Exception
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                // 关闭 csrf
                .csrf().disable()
                // 设置 session 状态 STATELESS 无状态
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                // 需要权限验证
                .mvcMatchers("/user/**").authenticated()
                .and()
                // 登陆页面
                .formLogin()
                //.loginPage("/login.html")
                // 登陆成功跳转页面
                .defaultSuccessUrl("/")
                //.failureForwardUrl("/login.html")
                .permitAll()
                .and()
                // 登出
                //.logout()
                // 注销的时候删除会话
                //.deleteCookies("JSESSIONID")
                // 默认登出请求为 /logout，可以用下面自定义
                //.logoutRequestMatcher(new AntPathRequestMatcher("/logout"))
                // 自定义登出成功的页面，默认为登陆页
                //.logoutSuccessUrl("/logout.html")
                //.permitAll()
                //.and()
                // 开启 cookie 保存用户信息
                //.rememberMe()
                // cookie 有效时间
                //.tokenValiditySeconds(60 * 60 * 24 * 7)
                // 设置cookie 的私钥，默认为随机生成的key
                //.key("remember")
                //.and()
                //验证登陆的 filter
                .addFilter(loginFilter())
                //验证token的 filter
                .addFilter(jwtAuthenticationFilter());
    }

    /**
     * Web层面的配置，一般用来配置无需安全检查的路径
     * @param web
     * @throws Exception
     */
    @Override
    public void configure(WebSecurity web) throws Exception {
        web
                .ignoring()
                .antMatchers(
                        "**.js",
                        "**.css",
                        "/images/**",
                        "/webjars/**",
                        "/**/favicon.ico"
                );
    }
}
```

##### 权限控制

```java
@RestController
@RequestMapping(value = "/user", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
@PreAuthorize("hasRole('USER')")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping
    @PreAuthorize("hasRole('admin')")
    public ResponseEntity<List<UserVO>> getAllUser() {
        List<User> users = userService.findAll();
        List<UserVO> userViews = userService.castUserVO(users);
        return ResponseEntity.ok(userViews);
    }
}
```

请求上面的`getAllUser` 方法，需要当前用户同时拥有 `ROLE_USER`和 `ROLE_admin` 两个权限，才能通过权限验证。

在 @PreAuthorize 中我们可以利用内建的 SPEL 表达式：比如 'hasRole()' 来决定哪些用户有权访问。需注意的一点是 hasRole 表达式认为每个角色名字前都有一个前缀 'ROLE_'。

### 迭代上个版本

后来，我发现进行用户认证的时候，会将所有的 provider 都尝试一遍，那么外面将登陆的 `UsernameAndPasswordToken` 和 `JwtTToken` 都可以分别进行验证进行了啊，所有我预先定义 `UsernamePasswordAuthenticationToken` 包装登陆的信息，然后进入登陆的 `AuthenticationProvider` 进行认证，`token` 验证形式，使用 `PreAuthenticatedAuthenticationToken` 的包装，然后进入例外一个 ``AuthenticationProvider` ` 中认证。

现在我们的流程就更加清晰了。

![img](https://zqnight.gitee.io/kaimz.github.io/image/hexo/springboot-security/4.png)

所以现在我对以前的权限配置以及认证进行了一些更改：

#### 过滤器

在这里，我根据不同请求的类型，进行不同的适配，然后进行加工分装成不同的认证凭证，然后根据凭证的不同，进行不同的认证。

```java
@Slf4j
public class AuthenticationFilter extends UsernamePasswordAuthenticationFilter {
    @Autowired
    private JwtUtil jwtUtil;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        try {
            if (isLoginRequest(httpRequest, httpResponse)) {
                Authentication authResult = processLogin(httpRequest, httpResponse);
                successfulAuthentication(httpRequest, httpResponse, chain, authResult);
                return;
            }
            String token = obtainToken(httpRequest);
            if (StringUtils.isNotBlank(token)) {
                processTokenAuthentication(token);
            }
        } catch (AuthenticationException e) {
            unsuccessfulAuthentication(httpRequest, httpResponse, e);
            return;
        }
        chain.doFilter(request, response);
    }
    /**
     * 登陆成功调用，返回 token
     */
    @Override
    protected void successfulAuthentication(HttpServletRequest request,
                                            HttpServletResponse response,
                                            FilterChain chain, Authentication authResult) throws IOException {
        String token = jwtUtil.generateToken(authResult.getName());
        response.setStatus(HttpStatus.OK.value());
        response.getWriter().print(token);
    }

    private boolean isLoginRequest(HttpServletRequest request, HttpServletResponse response) {
        return requiresAuthentication(request, response) && "POST".equalsIgnoreCase(request.getMethod());
    }

    private String obtainToken(HttpServletRequest request) {
        return request.getHeader(jwtUtil.getHeader());
    }

    private Authentication processLogin(HttpServletRequest request, HttpServletResponse response) {
        String username = obtainUsername(request);
        String password = obtainPassword(request);
        return tryAuthenticationWithUsernameAndPassword(username, password);
    }

    private void processTokenAuthentication(String token) {
        Authentication resultOfAuthentication = tryToAuthenticateWithToken(token);
        // 设置上下文用户信息以及权限
        SecurityContextHolder.getContext().setAuthentication(resultOfAuthentication);
    }

    private Authentication tryAuthenticationWithUsernameAndPassword(String username, String password) {
        Authentication authentication = new UsernamePasswordAuthenticationToken(username, password);
        return tryToAuthenticate(authentication);
    }

    private Authentication tryToAuthenticateWithToken(String token) {
        PreAuthenticatedAuthenticationToken requestAuthentication = new PreAuthenticatedAuthenticationToken(token, null);
        return tryToAuthenticate(requestAuthentication);
    }

    private Authentication tryToAuthenticate(Authentication requestAuth) {
        Authentication responseAuth = getAuthenticationManager().authenticate(requestAuth);
        if (responseAuth == null || !responseAuth.isAuthenticated()) {
            throw new InternalAuthenticationServiceException("Unable to authenticate User for provided credentials");
        }
        log.debug("User successfully authenticated");
        return responseAuth;
    }
}
```

#### 授权认证

根据提供的凭证的类型，进行相关的验证操作

##### LoginAuthenticationProvider

跟上个版本的 登陆验证中的 `CustomAuthenticationProvider` 代码一样实现一样。

##### TokenAuthenticateProvider

根据 token 查找它的 权限 信息，并装在到认证的凭证中。

```java
public class TokenAuthenticateProvider implements AuthenticationProvider {
    @Autowired
    private JwtUtil jwtUtil;
    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String token = authentication.getName();
        String username = jwtUtil.getUsernameFromToken(token);
        UserDetails userDetails = userDetailsService.loadUserByUsername(username);
        return new PreAuthenticatedAuthenticationToken(username, null, userDetails.getAuthorities());
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return PreAuthenticatedAuthenticationToken.class.equals(authentication);
    }
}
```

#### 配置权限和相关设置

和上个版本没什么变化，只是将类换了一下

```java
@Configuration
@EnableWebSecurity // 开启 Security
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true, jsr250Enabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    public JwtUtil jwtUtil() {
        return new JwtUtil();
    }

    @Bean
    public UserDetailsService customService() {
        return new UserDetailServiceImpl();
    }

    @Bean("loginAuthenticationProvider")
    public AuthenticationProvider loginAuthenticationProvider() {
        return new LoginAuthenticationProvider();
    }

    @Bean("tokenAuthenticationProvider")
    public AuthenticationProvider tokenAuthenticationProvider() {
        return new TokenAuthenticateProvider();
    }

    @Bean
    public AuthenticationFilter authenticationFilter() throws Exception {
        AuthenticationFilter authenticationFilter = new AuthenticationFilter();
        authenticationFilter.setAuthenticationManager(authenticationManager());
        return authenticationFilter;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(5);
    }

    @Bean
    @Override
    public UserDetailsService userDetailsService() {
        return new UserDetailServiceImpl();
    }
    /**
     * 主要是对身份验证的设置
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth
                .authenticationProvider(loginAuthenticationProvider())
                .authenticationProvider(tokenAuthenticationProvider())
                .userDetailsService(userDetailsService());

    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                // 关闭 csrf
                .csrf().disable()
                // 设置 session 状态 STATELESS 无状态
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                // 需要权限验证
                .mvcMatchers("/user/**").authenticated()
                .and()
                // 登陆页面
                .formLogin()
                //.loginPage("/login.html")
                // 登陆成功跳转页面
                .defaultSuccessUrl("/")
                .failureForwardUrl("/login.html")
                .permitAll()
                .and()
                .addFilter(authenticationFilter())
        ;
    }
}
```

### 后续完善

1. 修改密码，登出操作 token 的失效机制；
2. OAuth2 授权服务器的搭建；
3. 修改权限后，下次请求刷新权限；
4. ……

### 附录一：HttpSecurity常用方法

| 方法                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| `openidLogin()`       | 用于基于 OpenId 的验证                                       |
| `headers()`           | 将安全标头添加到响应                                         |
| `cors()`              | 配置跨域资源共享（ CORS ）                                   |
| `sessionManagement()` | 允许配置会话管理                                             |
| `portMapper()`        | 允许配置一个`PortMapper`(`HttpSecurity#(getSharedObject(class))`)，其他提供`SecurityConfigurer`的对象使用 `PortMapper` 从 HTTP 重定向到 HTTPS 或者从 HTTPS 重定向到 HTTP。默认情况下，Spring Security使用一个`PortMapperImpl`映射 HTTP 端口8080到 HTTPS 端口8443，HTTP 端口80到 HTTPS 端口443 |
| `jee()`               | 配置基于容器的预认证。 在这种情况下，认证由Servlet容器管理   |
| `x509()`              | 配置基于x509的认证                                           |
| `rememberMe`          | 允许配置“记住我”的验证                                       |
| `authorizeRequests()` | 允许基于使用`HttpServletRequest`限制访问                     |
| `requestCache()`      | 允许配置请求缓存                                             |
| `exceptionHandling()` | 允许配置错误处理                                             |
| `securityContext()`   | 在`HttpServletRequests`之间的`SecurityContextHolder`上设置`SecurityContext`的管理。 当使用`WebSecurityConfigurerAdapter`时，这将自动应用 |
| `servletApi()`        | 将`HttpServletRequest`方法与在其上找到的值集成到`SecurityContext`中。 当使用`WebSecurityConfigurerAdapter`时，这将自动应用 |
| `csrf()`              | 添加 CSRF 支持，使用`WebSecurityConfigurerAdapter`时，默认启用 |
| `logout()`            | 添加退出登录支持。当使用`WebSecurityConfigurerAdapter`时，这将自动应用。默认情况是，访问URL”/ logout”，使HTTP Session无效来清除用户，清除已配置的任何`#rememberMe()`身份验证，清除`SecurityContextHolder`，然后重定向到”/login?success” |
| `anonymous()`         | 允许配置匿名用户的表示方法。 当与`WebSecurityConfigurerAdapter`结合使用时，这将自动应用。 默认情况下，匿名用户将使用`org.springframework.security.authentication.AnonymousAuthenticationToken`表示，并包含角色 “ROLE_ANONYMOUS” |
| `formLogin()`         | 指定支持基于表单的身份验证。如果未指定`FormLoginConfigurer#loginPage(String)`，则将生成默认登录页面 |
| `oauth2Login()`       | 根据外部OAuth 2.0或OpenID Connect 1.0提供程序配置身份验证    |
| `requiresChannel()`   | 配置通道安全。为了使该配置有用，必须提供至少一个到所需信道的映射 |
| `httpBasic()`         | 配置 Http Basic 验证                                         |
| `addFilterAt()`       | 在指定的Filter类的位置添加过滤器                             |

