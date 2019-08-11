---
title: 学习 Spring Boot：（二十九）Spring Boot Junit 单元测试
date: 2018-04-26 22:08:03
tags: [Spring Boot]
categories: 学习笔记
---

### 前言

JUnit 是一个**回归测试框架**，被开发者用于实施对应用程序的单元测试，加快程序编制速度，同时提高编码的质量。

JUnit 测试框架具有以下重要特性：

- 测试工具
- 测试套件
- 测试运行器
- 测试分类

<!--more-->

### 了解 Junit 基础方法

#### 加入依赖

在 `pom.xml` 中加入依赖：

```xml
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <scope>test</scope>
    <version>4.12</version>
</dependency>
```

#### 创建测试类和测试方法

1. 测试类的的命名规则一般是 `xxxTest.java` ；
2. 测试类中测试的方法可以有前缀，这个看统一标准，所以有时候会发现别人的测试方法上有`test`前缀；
3. 并且测试方法上加上注解 `@Test`。

使用 IDEA 中，**选中当前类名**，使用快捷键 `ALT + ENTER`（WIN），向下选则 `Create Test` 回车，即可进入生成测试类的选项中，再次回车，就快速的生成测试类。

![img](https://zqnight.gitee.io/kaimz.github.io/image/hexo/junit-test/1.png) 

`OK` 完你会发现，生成的测试类在 `src/test` 目录下，测试类和源代码的**包名** 是一致的。生成后结果（注意下生成的方法名是不加 test）：

```java
public class HelloServiceImplTest {

    @Before
    public void setUp() throws Exception {
    }

    @After
    public void tearDown() throws Exception {
    }

    @Test
    public void say() {
    }
}
```

#### JUnit中的注解

- `@BeforeClass`：针对所有测试，只执行一次，且必须为`static void`
- `@Before`：初始化方法，执行当前测试类的每个测试方法前执行。
- `@Test`：测试方法，在这里可以测试期望异常和超时时间
- `@After`：释放资源，执行当前测试类的每个测试方法后执行
- `@AfterClass`：针对所有测试，只执行一次，且必须为`static void`
- `@Ignore`：忽略的测试方法（只在测试类的时候生效，单独执行该测试方法无效）
- ` @RunWith`:可以更改测试运行器 ，缺省值 `org.junit.runner.Runner`

一个单元测试类执行顺序为：

`@BeforeClass` –> `@Before` –> `@Test` –> `@After` –> `@AfterClass`

每一个测试方法的调用顺序为：

`@Before` –> `@Test` –> `@After`

#### 超时测试

如果一个测试用例比起指定的毫秒数花费了更多的时间，那么 Junit 将自动将它标记为失败。`timeout` 参数和 `@Test`注释一起使用。现在让我们看看活动中的 `@test(timeout)`。

```java
    @Test(timeout = 1000)
    public void testTimeout() throws InterruptedException {
        TimeUnit.SECONDS.sleep(2);
        System.out.println("Complete");
    }
```

上面测试会失败，在一秒后会抛出异常 `org.junit.runners.model.TestTimedOutException: test timed out after 1000 milliseconds`

#### 异常测试

你可以测试代码是否它抛出了想要得到的异常。expected 参数和 @Test 注释一起使用。现在让我们看看活动中的 `@Test(expected)`。

```java
    @Test(expected = NullPointerException.class)
    public void testNullException() {
        throw new NullPointerException();
    }
```

上面代码会测试成功。

#### 套件测试

```java
public class TaskOneTest {
    @Test
    public void test() {
        System.out.println("Task one do.");
    }
}

public class TaskTwoTest {
    @Test
    public void test() {
        System.out.println("Task two do.");
    }
}

public class TaskThreeTest {
    @Test
    public void test() {
        System.out.println("Task Three.");
    }
}

@RunWith(Suite.class) // 1. 更改测试运行方式为 Suite
// 2. 将测试类传入进来
@Suite.SuiteClasses({TaskOneTest.class, TaskTwoTest.class, TaskThreeTest.class})
public class SuitTest {
    /**
     * 测试套件的入口类只是组织测试类一起进行测试，无任何测试方法，
     */
}
```

#### 参数化测试

Junit 4 引入了一个新的功能**参数化测试**。参数化测试允许开发人员使用不同的值反复运行同一个测试。你将遵循 5 个步骤来创建**参数化测试**。

- 用 `@RunWith(Parameterized.class) `来注释 test 类。
- 创建一个由 `@Parameters` 注释的**公共的静态方法**，它返回**一个对象的集合(数组)**来作为测试数据集合。
- 创建一个公共的构造函数，它接受和一行测试数据相等同的东西。
- 为每一列测试数据创建一个实例变量。
- 用实例变量作为测试数据的来源来创建你的测试用例。

```java
//1.更改默认的测试运行器为RunWith(Parameterized.class)
@RunWith(Parameterized.class)
public class ParameterTest {
    // 2.声明变量存放预期值和测试数据
    private String firstName;
    private String lastName;

    //3.声明一个返回值 为Collection的公共静态方法，并使用@Parameters进行修饰
    @Parameterized.Parameters //
    public static List<Object[]> param() {
        // 这里我给出两个测试用例
        return Arrays.asList(new Object[][]{{"Mike", "Black"}, {"Cilcln", "Smith"}});
    }

    //4.为测试类声明一个带有参数的公共构造函数，并在其中为之声明变量赋值
    public ParameterTest(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }
    // 5. 进行测试，发现它会将所有的测试用例测试一遍
    @Test
    public void test() {
        String name = firstName + " " + lastName;
        assertThat("Mike Black", is(name));
    }
}
```

#### Hamcrest

JUnit 4.4 结合 Hamcrest 提供了一个全新的断言语法——assertThat。

语法：

```java
assertThat( [actual], [matcher expected] );
```

assertThat 使用了 Hamcrest 的 Matcher 匹配符，用户可以使用匹配符规定的匹配准则精确的指定一些想设定满足的条件，具有很强的易读性，而且使用起来更加灵活。

具体使用的一些匹配规则可以查看源码。

### Spring Boot 中使用 JUnit

Spring 框架提供了一个专门的测试模块（spring-test），用于应用程序的集成测试。 在 Spring Boot 中，你可以通过spring-boot-starter-test启动器快速开启和使用它。

#### 加入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

#### Spring Boot 测试

```java
// 获取启动类，加载配置，确定装载 Spring 程序的装载方法，它回去寻找 主配置启动类（被 @SpringBootApplication 注解的）
@SpringBootTest
// 让 JUnit 运行 Spring 的测试环境， 获得 Spring 环境的上下文的支持
@RunWith(SpringRunner.class)
public class EmployeeServiceImplTest {
    // do 
}
```

#### Spring MVC 测试

当你想对 `Spring MVC` 控制器编写单元测试代码时，可以使用`@WebMvcTest`注解。它提供了自配置的 `MockMvc`，可以不需要完整启动 `HTTP` 服务器就可以快速测试 `MVC 控制器`。

1. 需要测试的 Controller：

   ```java
   @RestController
   @RequestMapping(value = "/emp", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
   public class EmployeeController {
       private final EmployeeService employeeService;

       @Autowired
       public EmployeeController(EmployeeService employeeService) {
           this.employeeService = employeeService;
       }

       @GetMapping
       public ResponseEntity<List<EmployeeResult>> listAll() {
           return ResponseEntity.ok(employeeService.findEmployee());
       }
   }
   ```

2. 编写 `MockMvc` 的测试类：

   ```java
   @RunWith(SpringRunner.class)
   @WebMvcTest(EmployeeController.class)
   public class EmployeeController2Test {
       @Autowired
       private MockMvc mvc;

       @MockBean
       private EmployeeService employeeService;

       public void setUp() {
           // 数据打桩，设置该方法返回的 body一直 是空的
           Mockito.when(employeeService.findEmployee()).thenReturn(new ArrayList<>());
       }

       @Test
       public void listAll() throws Exception {
           mvc.perform(MockMvcRequestBuilders.get("/emp"))
                   .andExpect(status().isOk()) // 期待返回状态吗码200
                   // JsonPath expression  https://github.com/jayway/JsonPath
                   //.andExpect(jsonPath("$[1].name").exists()) // 这里是期待返回值是数组，并且第二个值的 name 存在，所以这里测试是失败的
                   .andDo(print()); // 打印返回的 http response 信息
       }
   }
   ```

   使用`@WebMvcTest`注解时，只有一部分的 Bean 能够被扫描得到，它们分别是：

   - `@Controller`
   - `@ControllerAdvice`
   - `@JsonComponent`
   - `Filter`
   - `WebMvcConfigurer`
   - `HandlerMethodArgumentResolver`
     其他常规的`@Component`（包括`@Service`、`@Repository`等）Bean 则不会被加载到 Spring 测试环境上下文中。
     所以我在上面使用了数据打桩，`Mockito` 在这篇文章最后一节。 

3. 我们也可以注入Spring 上下文的环境到 MockMvc 中，如下编写 `MockMvc` 的测试类：

   ```java
   @RunWith(SpringRunner.class)
   @SpringBootTest
   public class EmployeeControllerTest {
       /**
        * Interface to provide configuration for a web application.
        */
       @Autowired
       private WebApplicationContext ctx;

       private MockMvc mockMvc;

       /**
        * 初始化 MVC 的环境
        */
       @Before
       public void before() {
           mockMvc = MockMvcBuilders.webAppContextSetup(ctx).build();
       }

       @Test
       public void listAll() throws Exception {
           mockMvc
                   .perform(get("/emp") // 测试的相对地址
                   .accept(MediaType.APPLICATION_JSON_UTF8) // accept response content type
                   )
                   .andExpect(status().isOk()) // 期待返回状态吗码200
                   // JsonPath expression  https://github.com/jayway/JsonPath
                   .andExpect(jsonPath("$[1].name").exists()) // 这里是期待返回值是数组，并且第二个值的 name 存在
                   .andDo(print()); // 打印返回的 http response 信息
       }
   }
   ```

   值得注意的是需要首先使用  `WebApplicationContext ` 构建 `MockMvc`。

#### Spring Boot Web 测试

当你想启动一个完整的 HTTP 服务器对 Spring Boot 的 Web 应用编写测试代码时，可以使用`@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)`注解开启一个随机的可用端口。Spring Boot 针对 REST 调用的测试提供了一个 `TestRestTemplate` 模板，它可以解析链接服务器的相对地址。

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class EmployeeController1Test {
    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    public void listAll() {
        ResponseEntity<List> result = restTemplate.getForEntity("/emp", List.class);
        Assert.assertThat(result.getBody(), Matchers.notNullValue());
    }
}
```

其实之前上面的测试返回结果不是很正确，只能接收个List，给测试代码添加了不少麻烦，还好最终找到了解决办法：

```java
    @Test
    public void listAll() {
        // 由于我返回的是 List 类型的，一直想不到办法解决，网上给出了解决办法，使用 exchange 函数代替
        //public <T> ResponseEntity<T> exchange(String url, HttpMethod method,
        //			HttpEntity<?> requestEntity, ParameterizedTypeReference<T> responseType,
        //			Object... urlVariables) throws RestClientException {
        ParameterizedTypeReference<List<EmployeeResult>> type = new ParameterizedTypeReference<List<EmployeeResult>>() {};
        ResponseEntity<List<EmployeeResult>> result = restTemplate.exchange("/emp", HttpMethod.GET, null, type);
        Assert.assertThat(result.getBody().get(0).getName(), Matchers.notNullValue());
    }
```



#### Spring Data JPA 测试

我们可以使用 `@DataJpaTest`注解表示只对 JPA 测试；`@DataJpaTest`注解它只扫描`@Entity`Bean 和装配 Spring Data JPA 存储库，其他常规的`@Component`（包括`@Service`、`@Repository`等）Bean 则不会被加载到 Spring 测试环境上下文。

`@DataJpaTest` 还提供两种测试方式：

1. 使用内存数据库 `h2database`，Spring Data Jpa 测试默认采取的是这种方式；
2. 使用真实环境的数据库。

##### 使用内存数据库测试

1. 默认情况下，`@DataJpaTest`使用的是内存数据库进行测试，你无需配置和启用真实的数据库。只需要在 `pom.xml` 配置文件中声明如下依赖即可：

   ```xml
   <dependency>
       <groupId>com.h2database</groupId>
       <artifactId>h2</artifactId>
   </dependency>
   ```

   gradle file:

   ```groovy
   testCompile('com.h2database:h2')
   ```

2. 编写测试方法：

   ```java
   @RunWith(SpringRunner.class)
   @DataJpaTest
   public class EmployeeDaoTest {

       @Autowired
       private EmployeeDao employeeDao;

       @Test
       public void testSave() {
           Employee employee = new Employee();
           EmployeeDetail detail = new EmployeeDetail();
           detail.setName("kronchan");
           detail.setAge(24);
           employee.setDetail(detail);
           assertThat(detail.getName(), Matchers.is(employeeDao.save(employee).getDetail().getName()));;
       }
   }
   ```

##### 使用真实数据库测试

如要需要使用真实环境中的数据库进行测试，需要替换掉默认规则，使用`@AutoConfigureTestDatabase(replace = Replace.NONE)`注解：

```java
@RunWith(SpringRunner.class)
@DataJpaTest
// 加入 AutoConfigureTestDatabase 注解
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
public class EmployeeDaoTest {

    @Autowired
    private EmployeeDao employeeDao;

    @Test
    public void testSave() {
        Employee employee = new Employee();
        EmployeeDetail detail = new EmployeeDetail();
        detail.setName("kronchan");
        detail.setAge(24);
        employee.setDetail(detail);
        assertThat(detail.getName(), Matchers.is(employeeDao.save(employee).getDetail().getName()));;
    }
}
```

#### 事务控制

执行上面的新增数据的测试，发现测试通过，但是数据库却并没有新增数据。**默认情况下，在每个 JPA 测试结束时，事务会发生回滚。这在一定程度上可以防止测试数据污染数据库。**

如果你不希望事务发生回滚，你可以使用`@Rollback(false)`注解，该注解可以标注在类级别做全局的控制，也可以标注在某个特定不需要执行事务回滚的方法级别上。

也可以显式的使用注解 `@Transactional` 设置事务和事务的控制级别，放大事务的范围。

### Mockito

这部分参考 [使用Mockito和SpringTest进行单元测试](http://sunxiang0918.cn/2016/03/28/%E4%BD%BF%E7%94%A8Mockito%E5%92%8CSpringTest%E8%BF%9B%E8%A1%8C%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95/)

`JUnit`和`SpringTest`,基本上可以满足绝大多数的单元测试了，但是，由于现在的系统越来越复杂，相互之间的依赖越来越多。特别是微服务化以后的系统，往往一个模块的代码需要依赖几个其他模块的东西。因此，在做单元测试的时候，往往很难构造出需要的依赖。一个单元测试，我们只关心一个小的功能，但是为了这个小的功能能跑起来，可能需要依赖一堆其他的东西，这就导致了单元测试无法进行。所以，我们就需要再测试过程中引入`Mock`测试。

所谓的`Mock`测试就是在测试过程中，对于一些不容易构造的、或者和这次单元测试无关但是上下文又有依赖的对象，用一个虚拟的对象（Mock对象）来模拟，以便单元测试能够进行。

比如有一段代码的依赖为：
![img](https://zqnight.gitee.io/kaimz.github.io/image/hexo/mock/1.jpg)
当我们要进行单元测试的时候，就需要给`A`注入`B`和`C`,但是`C`又依赖了`D`，`D`又依赖了`E`。这就导致了，A的单元测试很难得进行。
但是，当我们使用了Mock来进行模拟对象后，我们就可以把这种依赖解耦，只关心A本身的测试，它所依赖的B和C，全部使用**Mock出来**的对象，并且给`MockB`和`MockC`指定一个明确的行为。就像这样：
![img](https://zqnight.gitee.io/kaimz.github.io/image/hexo/mock/2.jpg)

因此，当我们使用Mock后，对于那些难以构建的对象，就变成了个模拟对象，只需要提前的做`Stubbing`（桩）即可，所谓做桩数据，也就是告诉Mock对象，当与之交互时执行何种行为过程。比如当调用B对象的b()方法时，我们期望返回一个`true`，这就是一个设置桩数据的预期。

#### 基础

[Mockito 简明教程](http://www.importnew.com/21540.html)

#### Spring Boot 中使用

上面的 `Spring MVC 测试` 中也使用到了 `Mockito`，

`spring-boot-starter-test` 自带了 `mockito-core`。

##### 基础业务

```java
@Entity
@Data
@NoArgsConstructor
public class User implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false, length = 50)
    private String username;

    private String password;

    @CreationTimestamp
    private Date createDate;

    public User(Long id, String username) {
        this.id = id;
        this.username = username;
    }
}

public interface IUserRepository extends JpaRepository<User, Long> {
    boolean updateUser(User user);
}

@Service
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class UserServiceImpl implements IUserService {
    private final IUserRepository userRepository;

    @Override
    public User findOne(Long id) {
        return userRepository.getOne(id);
    }

    @Override
    public boolean updateUsername(Long id, String username) {
        User user = findOne(id);
        if (user == null) {
            return false;
        }
        user.setUsername(username);
        return userRepository.updateUser(user);
    }
}
```

##### 测试类

```java
public class IUserServiceTest {
    private IUserService userService;

    //@Mock
    private IUserRepository userRepository;

    @Before
    public void setUp() throws Exception {
        // 对所有注解了@Mock的对象进行模拟
        // MockitoAnnotations.initMocks(this);
        // 不使用注解，可以对单个对象进行 mock
        userRepository = Mockito.mock(IUserRepository.class);
        // 构造被测试对象
        userService = new UserServiceImpl(userRepository);
        // 打桩，构建当 userRepository的 getOne 函数执行参数为 1的时候，设置返回的结果 User
        Mockito.when(userRepository.getOne(1L)).thenReturn(new User(1L, "kronchan"));
        // 打桩，构建当 userRepository的 getOne 函数执行参数为 2的时候，设置返回的结果 null
        Mockito.when(userRepository.getOne(2L)).thenReturn(null);
        // 打桩，构建当 userRepository的 getOne 函数执行参数为 3的时候，设置结果抛出异常
        Mockito.when(userRepository.getOne(3L)).thenThrow(new IllegalArgumentException("The id is not support"));
        // 打桩，当 userRepository.updateUser 执行任何User类型的参数，返回的结果都是true
        Mockito.when(userRepository.updateUser(Mockito.any(User.class))).thenReturn(true);
    }

    @Test
    public void testUpdateUsernameSuccess() {
        long userId = 1L;
        String newUsername = "new kronchan";
        // 测试某个 service 的方法
        boolean updated = userService.updateUsername(userId, newUsername);
        // 检查结果
        Assert.assertThat(updated, Matchers.is(true));
        // Verifies certain behavior <b>happened once</b>.
        // mock对象一旦创建，就会自动记录自己的交互行为。通过verify(mock).someMethod()方法，来验证方法是否被调用。
        // 验证调用上面的service 方法后是否 userRepository.getOne(1L) 调用过，
        Mockito.verify(userRepository).getOne(userId);
        // 有条件可以测试下没有被调用过的方法：
        //   Mockito.verify(userRepository).deleteById(userId);
        //   则会测试失败：
        //    Wanted but not invoked:
        //      userRepository.deleteById(1L);
        //    However, there were exactly 2 interactions with this mock:
        //      userRepository.getOne(1L);
        //      userRepository.updateUser(
        //         User(id=1, username=new kronchan, password=null, createDate=null)
        //      );

        //  updateUsername 函数中我们调用了已经打桩了的其他的函数，现在我们来验证进入其他函数中的参数
        //构造参数捕获器，用于捕获方法参数进行验证
        ArgumentCaptor<User> userCaptor = ArgumentCaptor.forClass(User.class);
        // 验证updateUser方法是否被调用过，并且捕获入参
        Mockito.verify(userRepository).updateUser(userCaptor.capture());
        // 获取参数 updatedUser
        User updatedUser = userCaptor.getValue();
        // 验证入参是否是预期的
        Assert.assertThat(updatedUser.getUsername(), Matchers.is(newUsername));
        //保证这个测试用例中所有被Mock的对象的相关方法都已经被Verify过了
        Mockito.verifyNoMoreInteractions(userRepository);
        // 如果有一个交互，但是我们没有verify ，则会报错，
        //      org.mockito.exceptions.verification.NoInteractionsWanted:
        //      No interactions wanted here:
        //      -> at com.wuwii.service.IUserServiceTest.testUpdateUsernameSuccess(IUserServiceTest.java:74)
        //      But found this interaction on mock 'iUserRepository':
        //      -> at com.wuwii.service.impl.UserServiceImpl.findOne(UserServiceImpl.java:21)
        //      ***
    }

    @Test
    public void testUpdateUsernameFailed() {
        long userId = 2L;
        String newUsername = "new kronchan";
        // 没有经过 mock 的 updateUser 方法，它返回的是 false
        boolean updated = userService.updateUsername(userId, newUsername);
        Assert.assertThat(updated, Matchers.not(true));
        //验证userRepository的getOne(2L)这个方法是否被调用过，（这个是被测试过的，此步骤通过）
        Mockito.verify(userRepository).getOne(2L);
        // 验证 userRepository 的 updateUser(null)这个方法是否被调用过，（这个没有被测试过，此步骤不通过）
        //Mockito.verify(userRepository).updateUser(null);
        Mockito.verifyNoMoreInteractions(userRepository);
    }
}
```

##### 分析

###### 创建MOCK的对象

我需要对 `userService` 进行测定，就需要模拟 `userRepository` 对象。

我在 `setUp()` 方法中，模拟对象并打桩。

模拟对象有两种方式：

1. 对注解了@Mock的对象进行模拟 `MockitoAnnotations.initMocks(this);`
2. 对单个对象手动 mock ：`userRepository = Mockito.mock(IUserRepository.class);`

数据打桩，除了上面我代码上用的几个方法，还有非常多的方法，具体可以在使用的时候看到，主要分下面几种：

1. 最基本的用法就是调用 `when`以及`thenReturn`方法了。他们的作用就是指定当我们调用被代理的对象的某一个方法以及参数的时候，返回什么值。

2. 提供参数匹配器，灵活匹配参数。`any()`、`any(Class<T> type)`、`anyBoolean()`、`anyByte()`、`anyChar()`、`anyInt()`、`anyLong()`等等，它支持复杂的过滤，可以使用正则 `Mockito.matches(".*User$"))`，开头结尾验证`endsWith(String suffix)`，`startsWith(String prefix)`、判空验证`isNotNull()` `isNull()`
   也还可以使用 argThat(ArgumentMatcher<T> matcher)，如：`ArgumentMatcher`只有一个方法`boolean matches(T argument);`传入入参，返回一个boolean表示是否匹配。

   `Mockito.argThat(argument -> argument.getUsername.length() > 6;`

3. *Mockito*还提供了两个表示行为的方法：`thenAnswer(Answer<?> answer);`、`thenCallRealMethod();`,分别表示自定义处理调用后的行为，以及调用真实的方法。这两个方法在有些测试用例中还是很有用的。

4. 对于同一个方法，*Mockito*可以是顺序与次数关心的。也就是说可以实现同一个方法，第一次调用返回一个值，第二次调用返回一个值，甚至第三次调用抛出异常等等。只需要连续的调用`thenXXXX`即可。

5. 如果为一个返回为*Void*的方法设置桩数据。上面的方法都是表示的是有返回值的方法，而由于一个方法没有返回值，因此我们不能调用`when`方法(编译器不允许)。因此，对于无返回值的方法，*Mockito*提供了一些列的`doXXXXX`方法，比如：`doAnswer(Answer answer)`、`doNothing()`、`doReturn(Object toBeReturned)`、`doThrow(Class<? extends Throwable> toBeThrown)`、`doCallRealMethod()`。他们的使用方法其实和上面的`thenXXXX`是一样的，但是`when`方法传入的是*Mock的对象*：

   ```java
   /*对void的方法设置模拟*/
   Mockito.doAnswer(invocationOnMock -> {
       System.out.println("进入了Mock");
       return null;
   }).when(fileRecordDao).insert(Mockito.any());
   ```

   当 Mockito 监视一个真实的对象的时候，我们也可以模拟这个对象的方法返回我们设置的期望值，

   ```java
   List spy = spy(new LinkedList());  
   List spy = spy(new LinkedList());  
   // IndexOutOfBoundsException (the list is yet empty)  
   when(spy.get(0)).thenReturn("foo");  
   // You have to use doReturn() for stubbing  
   doReturn("foo").when(spy).get(0);  
   ```

   when方法参数中`spy.get(0)`，调用的是真实list对象的get(0)，这会产生 `IndexOutOfBoundsException`异常，所以这时需要用到 `doReturn` 方法来设置返回值。

###### 验证测试方法的结果

使用断言语句检查结果。

###### 验证MOCK对象的调用

其实，在这里我们如果只是验证方法结果的正确的话，就非常简单，但是，在复杂的方法调用堆栈中，往往可能出现结果正确，但是过程不正确的情况。比如，`updateUserName`方法返回false是有两种可能的，一种可能是用户没有找到，还有一种可能就是`userRepository.updateUser(userPO)`返回false。因此，如果我们只是使用`Assert.assertFalse(updated);`来验证结果，可能就会忽略某些错误。

1. 因此我在上面的测试中还需要验证指定的方法 `userRepository).getOne(userId);`是否运行过，而且我还使用了参数捕获器，抓取中间的方法参数，用来验证。

2. 提供了`verify(T mock, VerificationMode mode)`方法。`VerificationMode` 有很多作用，

   ```java
     // 验证指定方法 get(3) 没有被调用  
       verify(mock, never()).get(3);  
   ```

3. `verifyZeroInteractions`和`verifyNoMoreInteractions` 验证所有 mock 的方法是否都调用过了。

