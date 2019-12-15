---
title: 学习Spring Boot：（十）使用hibernate validation完成数据后端校验
date: 2018-02-23 10:08:11
tags: [Spring Boot]
categories: 学习笔记

---

### 前言
后台数据的校验也是开发中比较注重的一点，用来校验数据的正确性，以免一些非法的数据破坏系统，或者进入数据库，造成数据污染，由于数据检验可能应用到很多层面，所以系统对数据校验要求比较严格且追求可变性及效率。

<!--more-->

### 了解
了解一点概念性的东东。
* JSR 303 是 Java 为 Bean 数据合法性校验提供的标准框架,它已经包含在 JavaEE 6.0 中 。
* Hibernate Validator 是 JSR 303 的一个参考实现，所以它多实现了几个校验规则。
* Spring 4.0 拥有自己独立的数据校验框架,同时支持 JSR303 标准的校验框架。
* 在已经标注了 JSR303 注解的表单/命令对象前标注一个@Valid,Spring MVC 框架在将请求参数绑定到该入参对象后,就会调用校验框架根据注解声明的校验规则实施校验
* Spring MVC 是通过对处理方法签名的规约来保存校验结果的:前一个表单/命令对象的校验结果保存到随后的入参中,这个保存校验结果的入参必须是 BindingResult 或Errors 类型,这两个类都位于org.springframework.validation 包中。
* 需校验的 Bean 对象和其绑定结果对象或错误对象时成对出现的,它们之间不允许声明其他的入参
* Errors 接口提供了获取错误信息的方法,如 getErrorCount() 或getFieldErrors(String field)
* BindingResult 扩展了 Errors 接口。

#### 支持的注解
`JSR` 提供的校验注解:
```
@Null   被的注解元素必须为 null    
@NotNull    被注解的元素必须不为 null    
@AssertTrue     被注解的元素必须为 true    
@AssertFalse    被注解的元素必须为 false    
@Min(value)     被注解的元素必须是一个数字，其值必须大于等于指定的最小值    
@Max(value)     被注解的元素必须是一个数字，其值必须小于等于指定的最大值    
@DecimalMin(value)  被注解的元素必须是一个数字，其值必须大于等于指定的最小值    
@DecimalMax(value)  被注解的元素必须是一个数字，其值必须小于等于指定的最大值    
@Size(max=, min=)   被注解的元素的大小必须在指定的范围内   集合或数组 集合或数组的大小是否在指定范围内 
@Digits (integer, fraction)     被注解的元素必须是一个数字，验证是否是符合指定格式的数字，interger指定整数精度，fraction指定小数精度。   
@Past   被注解的元素必须是一个过去的日期    
@Future     被注解的元素必须是一个将来的日期    
@Pattern(regex=,flag=)  被注解的元素必须符合指定的正则表达式
```
`Hibernate Validator` 提供的校验注解：
```
@NotBlank(message =)   验证字符串非null，且长度必须大于0    
@Email  被注释的元素必须是电子邮箱地址    
@Length(min=,max=)  被注解的值大小必须在指定的范围内    
@NotEmpty   被注解的字符串的必须非空    
@Range(min=,max=,message=)  验证该值必须在合适的范围内

```
可以在需要验证的属性上，使用多个验证方式，它们同时生效。  
`spring boot web` 已经有 `hibernate-validation` 的依赖，所以不需要再手动添加依赖。


### 使用
首先我在我的实体类上写了几个校验注解。
```java
public class SysUserEntity implements Serializable {
    private static final long serialVersionUID = 1L;

    //主键
    private Long id;
    //用户名
    @NotBlank(message = "用户名不能为空", groups = {AddGroup.class, UpdateGroup.class})
    private String username;
    //密码
    @NotBlank(message = "密码不能为空", groups = {AddGroup.class})
    private String password;
    //手机号
    @Pattern(regexp = "^1([345789])\\d{9}$",message = "手机号码格式错误")
    @NotBlank(message = "手机号码不能为空")
    private String mobile;
    //邮箱
    @Email(message = "邮箱格式不正确")
    private String email;
    //创建者
    private Long createUserId;
    //创建时间
    private Date createDate;
// ignore set and get
```

#### 使用@Validated进行校验
首先了解下：
**关于@Valid和@Validated的区别联系**
* `@Valid`: `javax.validation`， 是javax，也是就是**jsr303中定义的规范注解** 
* `@Validated`: `org.springframework.validation.annotation`， 是spring自己封装的注解。参数校验失败抛出 `org.springframework.validation.BindException` 异常。

`@Validated` 是 `@Valid` 的一个变种，扩展了 `@Valid` 的功能，支持 `group分组校验` 的写法，所以为了校验统一，尽量使用 `@Validated`

在controller自定义一个接口
```java
@PostMapping("/valid")
    public ResponseEntity<String> valid(@Validated @RequestBody SysUserEntity user, BindingResult result) {
        if (result.hasErrors()) {
            return ResponseEntity.status(BAD_REQUEST).body("校验失败");
        }
        return ResponseEntity.status(OK).body("校验成功");
    }
```
需要注意的有几点：
* 需要校验对象的时候，需要加上 spring 的校验注解 `@Validated` ，表示我们需要 spring 对它进行校验，而校验的信息会存放到其后的BindingResult中。
* BindingResult 必须和检验对象紧邻，中间不能穿插任何参数，如果有多个校验对象 `@Validated @RequestBody SysUserEntity user, BindingResult result, @Validated @RequestBody SysUserEntity user1, BindingResult result1`。

我在前端用 Swagger 进行测试下。
我发送一个 body，将 手机号输错：
```json
{
  "createDate": "",
  "createUserId": 0,
  "email": "k@wuwii.com",
  "id": 0,
  "mobile": "12354354",
  "password": "123",
  "username": "12312"
}
```

后端调试下 BindingResult 的结果，发现结果：
![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/hibernate-validation/1.png)
只要注意下 `errors` 属性，它是校验**所有**不符合规则的，是一个数组。

#### 分组校验
有时候 ，我们在新增和更新的时候校验效果是不一样的。例如上面，我在User新增的时候需要判断密码是不是为空，但是更新的时候我不做校验。这个时候就也要用到分组校验了。
```java
@NotBlank(message = "密码不能为空", groups = {AddGroup.class})
private String password;
```
将Contoller中的校验修改下。
```java
(@Validated({AddGroup.class}) @RequestBody SysUserEntity user, BindingResult result)
```
上面的意思是只有分组是AddGroup的校验才生效，其余的校验忽略。

经过我测试，把分组情况分下：
1. 在controller校验没加分组的时候，只对实体类的没有分组的注解有效。
2. 在controller校验加分组的时候，只对实体类的当前分组的注解有效，没有注解的也无效。
3. 当校验有两个分组的时候`@Validated({AddGroup.class, UpdateGroup.class})`，满足当前两个分组其中任意一个都可以校验，两个注解同时一起出现，也没问题，而且检验不通过的信息不会重复。

#### 自定义校验
有时候系统提供给我们的校验注解，并不够用，我们可以自定义校验，来满足我们的业务需求。

例如：现在我们有一个需求，需要检测一条信息的敏感词汇，如`sb` ……文明人，举个栗子 ……

##### 自定义校验注解
```java
// 注解可以用在哪些地方
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER})
@Retention(RUNTIME)
@Documented
// 指定校验规则实现类
@Constraint(validatedBy = {NotHaveSBValidator.class})
public @interface NotHaveSB {
    //默认错误消息
    String message() default "不能包含字符sb";
    //分组
    Class<?>[] groups() default {};
    //负载
    Class<? extends Payload>[] payload() default {};
    //指定多个时使用
    @Target({FIELD, METHOD, PARAMETER, ANNOTATION_TYPE})
    @Retention(RUNTIME)
    @Documented
    @interface List {
        NotHaveSB[] value();
    }

}
```


##### 规则校验实现类
```java
// 可以指定检验类型，这里选择的是 String
public class NotHaveSBValidator implements ConstraintValidator<NotHaveSB, String> {
    @Override
    public void initialize(NotHaveSB notHaveSB) {

    }
    
    /**
     *
     * @param s 待检验对象
     * @param constraintValidatorContext 检验上下文，可以设置检验的错误信息
     * @return false 代表检验失败
     */
    @Override
    public boolean isValid(String s, ConstraintValidatorContext constraintValidatorContext) {
        return !StringUtils.isNotBlank(s) || !s.toLowerCase().contains("sb");
    }
}
```
 所有的验证者都需要实现ConstraintValidator接口，它的接口也很形象，包含一个初始化事件方法，和一个判断是否合法的方法。

##### 测试一下喂
现在我的用户类上，也没什么多余的字段拿出来测试，暂时把 password 字段拿来测试吧。
```java
//@NotBlank(message = "密码不能为空", groups = AddGroup.class)
    @NotHaveSB
    private String password;
```

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/hibernate-validation/2.png)

#### 手动校验
这个是我最终想要的处理方式。
由于现在都是前后端分离开发的，校验失败的时候，抛出自定义的异常，然后统一处理这些异常，最后将相关的错误提示信息返回给前端处理。

~~新建一个验证工具类~~
```java
public class ValidatorUtils {
    private static Validator validator;

    static {
        validator = Validation.buildDefaultValidatorFactory().getValidator();
    }

    /**
     * 手动校验对象
     *
     * @param object 待校验对象
     * @param groups 待校验的组
     * @throws KCException 校验不通过，则抛出 KCException 异常
     */
    public static void validateEntity(Object object, Class<?>... groups)
            throws KCException {
        Set<ConstraintViolation<Object>> constraintViolations = validator.validate(object, groups);
        if (!constraintViolations.isEmpty()) {
            String msg = constraintViolations.parallelStream()
                    .map(ConstraintViolation::getMessage)
                    .collect(Collectors.joining("，"));
            throw new KCException(msg);
        }
    }
}
```
~~它主要做的事情就是验证我们的待验证对象，验证不同通过的时候，抛出自定义异常，在后台统一处理异常就可以了。~~

~~在业务中直接调用就可以了，有分组添加分组就行~~：
```java
@PostMapping("/valid1")
    public ResponseEntity<String> customValid(@RequestBody SysUserEntity user) {
        ValidatorUtils.validateEntity(user);
        return ResponseEntity.status(OK).body("校验成功");
    }
```

最后测试一下，查看返回结果是否符合预期：

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/hibernate-validation/3.png)

##### 手动校验的补充
决定还是采用注解的形式进行编码，本来想用处理方法参数的装配进行检验，写好了发现和 `@responseBody` 不能同时使用，然后发现还是可以使用 `@Validated` 直接校验，抛出异常， 进行捕捉异常统一处理。
```java
    @PostMapping()
    @ApiOperation("新增")
    public ResponseEntity insert(@Validated SysUserAddForm user) 
```

在全局异常处理里面加上 处理绑定参数异常 `org.springframework.validation.BindException`：
```java
    /**
     * 参数检验违反约束（数据校验）
     * @param e BindException
     * @return error message
     */
    @org.springframework.web.bind.annotation.ExceptionHandler(BindException.class)
    public ResponseEntity<String> handleConstraintViolationException(BindException e) {
        LOGGER.debug(e.getMessage(), e);
        return ResponseEntity.status(BAD_REQUEST).body(
                e.getBindingResult()
                        .getAllErrors()
                        .stream()
                        .map(DefaultMessageSourceResolvable::getDefaultMessage)
                        .collect(Collectors.joining(",")));
    }
```
