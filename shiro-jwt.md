---
title: 学习Spring Boot：（十六）使用Shiro与JWT 实现认证服务
date: 2018-2-26 23:08:03
tags: [Spring Boot,shiro]
categories: 学习笔记
---

### 前言
需要把Web应用做成无状态的，即服务器端无状态，就是说服务器端不会存储像会话这种东西，而是每次请求时access_token进行资源访问。这里我们将使用 JWT [^1]，基于散列的消息认证码，使用一个密钥和一个消息作为输入，生成它们的消息摘要。该密钥只有服务端知道。访问时使用该消息摘要进行传播，服务端然后对该消息摘要进行验证。

<!--more-->

[^1]: JSON WEB Token

### 认证步骤
1. 客户端第一次使用用户名密码访问认证服务器，服务器验证用户名和密码，认证成功,使用用户密钥生成JWT并返回
2. 之后每次请求客户端带上JWT
3. 服务器对JWT进行验证

#### 自定义 jwt 拦截器
```java
/**
 * oauth2拦截器，现在改为 JWT 认证
 */
public class OAuth2Filter extends FormAuthenticationFilter {
    /**
     * 设置 request 的键，用来保存 认证的 userID,
     */
    private final static String USER_ID = "USER_ID";
    @Resource
    private JwtUtils jwtUtils;

    /**
     * logger
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(OAuth2Filter.class);


    /**
     * shiro权限拦截核心方法 返回true允许访问resource，
     *
     * @param request
     * @param response
     * @param mappedValue
     * @return
     */
    @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        String token = getRequestToken((HttpServletRequest) request);
        try {
            // 检查 token 有效性
            //ExpiredJwtException JWT已过期
            //SignatureException JWT可能被篡改
            Jwts.parser().setSigningKey(jwtUtils.getSecret()).parseClaimsJws(token).getBody();
        } catch (Exception e) {
            // 身份验证失败，返回 false 将进入onAccessDenied 判断是否登陆。
            onLoginFail(response);
            return false;
        }
        Long userId = getUserIdFromToken(token);
        // 存入到 request 中，在后面的业务处理中可以使用
        request.setAttribute(USER_ID, userId);
        return true;
    }

    /**
     * 当访问拒绝时是否已经处理了；
     * 如果返回true表示需要继续处理；
     * 如果返回false表示该拦截器实例已经处理完成了，将直接返回即可。
     *
     * @param request
     * @param response
     * @return
     * @throws Exception
     */
    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
        if (isLoginRequest(request, response)) {
            if (isLoginSubmission(request, response)) {
                return executeLogin(request, response);
            } else {
                return true;
            }
        } else {
            onLoginFail(response);
            return false;
        }
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
            ((HttpServletResponse) response).setStatus(HttpStatus.BAD_REQUEST.value());
            response.getWriter().print("账号活密码错误");
        } catch (IOException e1) {
            LOGGER.error(e1.getMessage(), e1);
        }
        return false;
    }

    /**
     * token 认证失败
     *
     * @param response
     */
    private void onLoginFail(ServletResponse response) {
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        ((HttpServletResponse) response).setStatus(HttpStatus.UNAUTHORIZED.value());
        try {
            response.getWriter().print("没有权限，请联系管理员授权");
        } catch (IOException e) {
            LOGGER.error(e.getMessage(), e);
        }
    }

    /**
     * 获取请求的token
     */
    private String getRequestToken(HttpServletRequest httpRequest) {
        //从header中获取token
        String token = httpRequest.getHeader(jwtUtils.getHeader());
        //如果header中不存在token，则从参数中获取token
        if (StringUtils.isBlank(token)) {
            return httpRequest.getParameter(jwtUtils.getHeader());
        }
        if (StringUtils.isBlank(token)) {
            // 从 cookie 获取 token
            Cookie[] cookies = httpRequest.getCookies();
            if (null == cookies || cookies.length == 0) {
                return null;
            }
            for (Cookie cookie : cookies) {
                if (cookie.getName().equals(jwtUtils.getHeader())) {
                    token = cookie.getValue();
                    break;
                }
            }
        }
        return token;
    }

    /**
     * 根据 token 获取 userID
     *
     * @param token token
     * @return userId
     */
    private Long getUserIdFromToken(String token) {
        if (StringUtils.isBlank(token)) {
            throw new KCException("无效 token", HttpStatus.UNAUTHORIZED.value());
        }
        Claims claims = jwtUtils.getClaimByToken(token);
        if (claims == null || jwtUtils.isTokenExpired(claims.getExpiration())) {
            throw new KCException(jwtUtils.getHeader() + "失效，请重新登录", HttpStatus.UNAUTHORIZED.value());
        }
        return Long.parseLong(claims.getSubject());
    }

}
```
将自定义shiro拦截器，设置到 `ShiroFilterFactoryBean` 中，然后将需要进行权限验证的 `path` 进行设置拦截过滤。

#### 登陆
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
        // 登陆成功后直接返回 token ,然后后续放到 header 中认证
        return ResponseEntity.status(HttpStatus.OK).body(jwtUtils.generateToken(getUserId()));
    }
```

#### JwtUtils
我前面给 jwt 设置了三个参数
```yml
# jwt 配置
jwt:
  # 加密密钥
  secret: 61D73234C4F93E03074D74D74D1E39D9 #blog.wuwii.com
  # token有效时长
  expire: 7 # 7天，单位天
  # token 存在 header 中的参数
  header: token
```
jwt 工具类的编写
```java
@ConfigurationProperties(prefix = "jwt")
@Component
public class JwtUtils {
    /**
     * logger
     */
    private Logger logger = LoggerFactory.getLogger(JwtUtils.class);

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
     * @param userId 用户ID
     * @return token
     */
    public String generateToken(long userId) {
        Date nowDate = new Date();

        return Jwts.builder()
                .setHeaderParam("typ", "JWT")
                // 后续获取 subject 是 userid
                .setSubject(userId + "")
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
    public Claims getClaimByToken(String token) {
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

    public String getSecret() {
        return secret;
    }

    public void setSecret(String secret) {
        this.secret = secret;
    }

    public int getExpire() {
        return expire;
    }

    public void setExpire(int expire) {
        this.expire = expire;
    }

    public String getHeader() {
        return header;
    }

    public void setHeader(String header) {
        this.header = header;
    }
}
```
### 总结
由于 JWT 这种方式，服务端不需要保存任何状态，所以服务端不需要使用 session 保存用户信息，单元测试也比较方便，虽然中间转码解码会消耗一些性能，但是影响不大，还比较方便的应用在 SSO [^2]。

[^2]: Single Sign On
