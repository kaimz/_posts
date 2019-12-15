---
title: 学习Spring Boot：（六） 集成Swagger2
date: 2018-02-23 10:08:03
tags: [Spring Boot]
categories: 学习笔记

---

### 前言
Swagger是用来描述和文档化RESTful API的一个项目。Swagger Spec是一套规范，定义了该如何去描述一个RESTful API。类似的项目还有RAML、API Blueprint。 根据Swagger Spec来描述RESTful API的文件称之为Swagger specification file，它使用JSON来表述，也支持作为JSON支持的YAML。

Swagger specification file可以用来给swagger-ui生成一个Web的可交互的文档页面，以可以用swagger2markup生成静态文档，也可用使用swagger-codegen生成客户端代码。总之有了有个描述API的JSON文档之后，可以做各种扩展。

Swagger specification file可以手动编写，swagger-editor为了手动编写的工具提供了预览的功能。但是实际写起来也是非常麻烦的，同时还得保持代码和文档的两边同步。于是针对各种语言的各种框架都有一些开源的实现来辅助自动生成这个`Swagger specification file。

swagger-core是一个Java的实现，现在支持JAX-RS。swagger-annotation定义了一套注解给用户用来描述API。
spring-fox也是一个Java的实现，它支持Spring MVC， 它也支持swagger-annotation定义的部分注解。

<!--more-->

### 使用
#### 添加依赖
在pom文件添加：
```xml
<swagger.version>2.7.0</swagger.version>

<dependency>
			<groupId>io.springfox</groupId>
			<artifactId>springfox-swagger2</artifactId>
			<version>${swagger.version}</version>
		</dependency>
		<dependency>
			<groupId>io.springfox</groupId>
			<artifactId>springfox-swagger-ui</artifactId>
			<version>${swagger.version}</version>
		</dependency>
```

#### 配置docket
```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    /**
     * SpringBoot默认已经将classpath:/META-INF/resources/和classpath:/META-INF/resources/webjars/映射
     * 所以该方法不需要重写，如果在SpringMVC中，可能需要重写定义（我没有尝试）
     * 重写该方法需要 extends WebMvcConfigurerAdapter
     */
//    @Override
//    public void addResourceHandlers(ResourceHandlerRegistry registry) {
//        registry.addResourceHandler("swagger-ui.html")
//                .addResourceLocations("classpath:/META-INF/resources/");
//
//        registry.addResourceHandler("/webjars/**")
//                .addResourceLocations("classpath:/META-INF/resources/webjars/");
//    }
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.wuwii"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("Spring Boot中使用Swagger2构建RESTful APIs")
                .description("rest api 文档构建利器")
                .termsOfServiceUrl("https://blog.wuwii.com/")
                .contact("KronChan")
                .version("1.0")
                .build();
    }
}

```
##### builder说明
根据网上一位前辈的文章：
```java
@Bean
  public Docket petApi() {
    return new Docket(DocumentationType.SWAGGER_2)
        .select() //1
          .apis(RequestHandlerSelectors.any())
          .paths(PathSelectors.any())
          .build()
        .pathMapping("/") //2
        .directModelSubstitute(LocalDate.class, //3
            String.class)
        .genericModelSubstitutes(ResponseEntity.class) //4
        .alternateTypeRules( //5
            newRule(typeResolver.resolve(DeferredResult.class,
                    typeResolver.resolve(ResponseEntity.class, WildcardType.class)),
                typeResolver.resolve(WildcardType.class)))
        .useDefaultResponseMessages(false) //6
        .globalResponseMessage(RequestMethod.GET, //7
            newArrayList(new ResponseMessageBuilder()
                .code(500)
                .message("500 message")
                .responseModel(new ModelRef("Error"))
                .build()))
        .securitySchemes(newArrayList(apiKey())) //8
        .securityContexts(newArrayList(securityContext())) //9
        ;
  }
```
方法说明：
1. 定义了需要生成API文档的endpoint，`api()`方法可以通过RequestHandlerSelectors的各种选择器来选择，比如说选择所有注解了@RsestController的类中的所有API e.g. `.apis(RequestHandlerSelectors.withClassAnnotation(RestController.class))`。`path()`方法可以通过PathSelectors的来匹配路径，提供了regex匹配或者ant匹配
2. 定义了API的根路径
3. 输出模型定义时的替换，比如遇到所有LocalDate的字段时，输出成String
4. 遇到对应泛型类型的外围类，直接解析成泛型类型，比如说`ResponseEntity<T>`，应该直接输出成类型T
5. 提供了自定义性更强的针对泛型的处理，示例中的代码的意思是将类型DeferredResult直接解析成类型T
6. 是否使用默认的ResponseMessage， 框架默认定义了一些针对各个HTTP方法时各种不同响应值对应的message
7. 全局的定义ResponseMessage，示例代码定义GET方法的500错误的消息以及错误模型。注意这里GET方法的所有ResponseMessage都会被这里的定义覆盖
8. 定义API支持的SecurityScheme，指的是认证方式，支持`OAuth`、`APIkey`。 P.S. 要让swagger-ui的oauth正常工作，需要定义个SecurityConfiguration的Bean
9. 定义具体上下文路径对应的认证方式
10. 还有一些接口可以定义API的名称等一些基本信息，定义API支持的数据格式等等。

#### 接口上添加文档
```java
@RestController
@Api(description = "这是一个控制器的描述 ")
public class PetController {
    /**
     * logger
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(PetController.class);

    private String no;
    private String kind;
    private String name;

    @ApiOperation(value="测试接口", notes="测试接口描述")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "id", value = "用户ID", required = true, dataType = "Long", paramType = "path"),
            @ApiImplicitParam(name = "pet", value = "宠物", required = true, dataType = "PetController")
    })
    @ApiResponses({
            @ApiResponse(code = 200, message = "请求完成"),
            @ApiResponse(code = 400, message = "请求参数错误")
    })
    @RequestMapping(path = "/index/{id}", method = RequestMethod.PUT)
    public PetController index1(@PathVariable("id") String id, @RequestBody PetController pet) {
        return pet;
    }
    
//…… get  / set
```

##### 常用的注解说明

![springboot-6-1.png](https://i.loli.net/2018/01/25/5a69955bf2645.png)

#### 查看API文档
启动Spring Boot程序，访问：http://host:port/swagger-ui.html
。就能看到RESTful API的页面。打开我们的测试接口的API ，可以查看这个接口的描述，以及参数等信息：
![springboot-6-2.png](https://i.loli.net/2018/01/25/5a6996ac5b3e6.png)
点击上图中右侧的Model Schema（黄色区域：它指明了这个requestBody的数据结构），此时pet中就有了pet对象的模板，修改上测试数据，点击下方`Try it out！`按钮，即可完成了一次请求调用！
![springboot-6-4.png](https://i.loli.net/2018/01/25/5a6997c3a1ce9.png)

调用完后，我们可以查看接口的返回信息：

![springboot-6-3.png](https://i.loli.net/2018/01/25/5a6996f6ab34d.png)

### 参考文章
* <a rel="external nofollow" target="_blank" href="http://blog.didispace.com/springbootswagger2/">Spring Boot中使用Swagger2构建强大的RESTful API文档档</a>
* <a rel="external nofollow" target="_blank" href="https://gumutianqi1.gitbooks.io/specification-doc/content/tools-doc/spring-boot-swagger2-guide.html">spring-boot-swagger2 使用手册</a>
* <a rel="external nofollow" target="_blank" href="http://yukinami.github.io/2015/07/07/%E4%BD%BF%E7%94%A8springfox%E7%94%9F%E6%88%90springmvc%E9%A1%B9%E7%9B%AE%E7%9A%84swagger%E7%9A%84%E6%96%87%E6%A1%A3/">使用springfox生成springmvc项目的swagger的文档</a>


### 例外补充点
#### 验证码
我使用的是 `com.github.axet.kaptcha` 的验证码
虽然按照别人的方法使用 `HttpServletResponse` 输出流，这种是暴露 `Servlet` 的接口。但是发现了一个问题了，在 swagger 的获取验证码接上测试的时候不能得到验证码图片，但是在 `img` 标签中是没问题，发现 swagger 还是把我的返回结果作为 `json` 处理。所以我还是想到使用下载中二进制流的方法，将 `BufferedImage` 转换成二进制流数组，总算是解决。

上最后解决的办法：
```java
/**
     * 获取验证码
     */
    @GetMapping(value = "/captcha.jpg", produces = MediaType.IMAGE_JPEG_VALUE)
    public ResponseEntity<byte[]> captcha()throws IOException {
        //生成文字验证码
        String text = producer.createText();
        //生成图片验证码
        BufferedImage image = producer.createImage(text);
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        ImageIO.write(image, "jpg", out);
        // 文字验证码保存到 shiro session
        ShiroUtils.setSessionAttribute(Constants.KAPTCHA_SESSION_KEY, text);
        HttpHeaders headers = new HttpHeaders();
        headers.setCacheControl("no-store, no-cache");
        return ResponseEntity
                .status(HttpStatus.OK)
                .headers(headers)
                .body(out.toByteArray());
```
我是采用 `ImageIO` 工具类的将 `BufferedImage` 转换成 输出流，从输出流中获取二进制流数组。
```java
ImageIO.write(BufferedImage image,String format,OutputStream out)
```

再补充一个 将二进制流数组 `byte[]` 转换成 `BufferedImage`
```java
// 将二进制流数组转换成输入流
ByteArrayInputStream in = new ByteArrayInputStream(byte[] byets);    
// 读取输入流
BufferedImage image = ImageIO.read(InputStream in); 
```

在 `swagger` 上是这样的了：

![](https://zqnight.gitee.io/kaimz.github.io/image/hexo/springboot-swagger/1.png)

#### 配置不同环境中是否启动
在不同环境种配置是否启用规则：
```java
swagger:
  enable: true  # or false
```
在 swagger 配置类中加入
```java
    /**
     * 启用
     */
    @Value("${swagger.enable}")
    private boolean enable;
    
    …… set / get
```
配置 Docket 中
```java
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                // 加入 enable
                .enable(enable)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.wuwii"))
                .paths(PathSelectors.any())
                .build();
    }
```

#### SpringMVC 中配置 Swagger2
Swagger 的配置文件：
```java
@Configuration
@EnableSwagger2
@EnableWebMvc
@ComponentScan("com.devframe.controller")
public class SwaggerConfig extends WebMvcConfigurerAdapter {
    /**
     * 静态文件过滤
     * @param registry
     */
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("swagger-ui.html")
                .addResourceLocations("classpath:/META-INF/resources/");

        registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/");
    }
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.devframe"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("RESTful APIs")
                .description("rest api 文档构建利器")
                .termsOfServiceUrl("https://blog.wuwii.com/")
                .version("1.0")
                .build();
    }

}
```
额外需要在 `web.xml` 配置：
```xml
	<!-- Springmvc前端控制器扫描路径增加“/v2/api-docs”，用于扫描Swagger的 /v2/api-docs，否则 /v2/api-docs无法生效。-->
	<servlet-mapping>
		<servlet-name>dispatcher</servlet-name>
		<url-pattern>/v2/api-docs</url-pattern>
	</servlet-mapping>

```
#### 单元测试中会出现的错误

发现加入 `Swagger` 后，以前的单元测试再运行的时候，会抛出一个异常，参考 [How to run integration tests with spring and springfox?](https://github.com/springfox/springfox/issues/654)

```
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'documentationPluginsBootstrapper' defined in URL [jar:file:/C:/Users/test/.m2/repository/io/springfox/springfox-spring-web/2.0.0-SNAPSHOT/springfox-spring-web-2.0.0-SNAPSHOT.jar!/springfox/documentation/spring/web/plugins/DocumentationPluginsBootstrapper.class]: Unsatisfied dependency expressed through constructor argument with index 1 of type [java.util.List]: : No qualifying bean of type [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping] found for dependency [collection of org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping]: expected at least 1 bean which qualifies as autowire candidate for this dependency. Dependency annotations: {}; nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type [org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping] found for dependency [collection of org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping]: expected at least 1 bean which qualifies as autowire candidate for this dependency. Dependency annotations: {}
```

解决，单元测试上加入 `@EnableWebMvc`
