---
title: JPA 使用 Specification 复杂查询和 Criteria 查询
date: 2018-03-8 22:08:03
tags: [Spring Boot,jpa]
categories: 学习笔记
---

### 前言
`JPA` 给我们提供了基础的 `CURD` 的功能，并且用起来也是特别的方便，基本都是一行代码完成各种数据库操作，但是在复杂的多表查询的时候，我总是遇到各种问题，以前一般都是用原生 SQL 就行查询，原来它自带了复杂查询的 `JpaSpecificationExecutor` 接口，可以完成各种复杂查询，而且配合 JAVA 8的新特性，使用起来也是特别的方便。

<!--more-->
环境：
* Spring Boot - 2.0.0.RELEASE 
* Jpa
* jdk - 1.8
* mysql
* lombok
[本文学习代码的地址](https://github.com/kaimz/learning-code/tree/master/jpa-criteria)

### 使用 JpaSpecificationExecutor 复杂查询
#### 了解 JpaSpecificationExecutor
JPA 提供动态接口，利用类型检查的方式，进行复杂的条件查询，这个比自己写 SQL 更加安全。
```java
public interface JpaSpecificationExecutor<T> {

	/**
	 * Returns a single entity matching the given {@link Specification}.
	 * 
	 * @param spec
	 * @return
	 */
	T findOne(Specification<T> spec);

	/**
	 * Returns all entities matching the given {@link Specification}.
	 * 
	 * @param spec
	 * @return
	 */
	List<T> findAll(Specification<T> spec);

	/**
	 * Returns a {@link Page} of entities matching the given {@link Specification}.
	 * 
	 * @param spec
	 * @param pageable
	 * @return
	 */
	Page<T> findAll(Specification<T> spec, Pageable pageable);

	/**
	 * Returns all entities matching the given {@link Specification} and {@link Sort}.
	 * 
	 * @param spec
	 * @param sort
	 * @return
	 */
	List<T> findAll(Specification<T> spec, Sort sort);

	/**
	 * Returns the number of instances that the given {@link Specification} will return.
	 * 
	 * @param spec the {@link Specification} to count instances for
	 * @return the number of instances
	 */
	long count(Specification<T> spec);
}
```
`Specification` 是我们传入进去的查询参数，实际上它是一个接口，并且只有一个方法：
```java
public interface Specification<T> {

    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
}
```
### 使用
#### 创建实体类
1. 我现在有三个实体类是关联关系：
```java
@Entity
@Data
public class Employee implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(cascade = CascadeType.ALL) // 拥有级联维护的一方，参考http://westerly-lzh.github.io/cn/2014/12/JPA-CascadeType-Explaining/
    @JoinColumn(foreignKey = @ForeignKey(name = "none", value = ConstraintMode.NO_CONSTRAINT))
    private EmployeeDetail detail;

    @ManyToOne(fetch = FetchType.LAZY) // 默认 lazy ，通过懒加载，知道需要使用级联的数据，才去数据库查询这个数据，提高查询效率。
    // 设置外键的问题，参考http://mario1412.github.io/2016/06/27/JPA%E4%B8%AD%E5%B1%8F%E8%94%BD%E5%AE%9E%E4%BD%93%E9%97%B4%E5%A4%96%E9%94%AE/
    @JoinColumn(name = "jobId", foreignKey = @ForeignKey(name = "none", value = ConstraintMode.NO_CONSTRAINT))
    private Job job;
}

@Entity
@Data
public class EmployeeDetail implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private String phone;

    private Integer age;
}

@Entity
@Data
public class Job implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(targetEntity = Employee.class, mappedBy = "job") // mappedBy 只有在双向关联的时候设置，表示关系维护的一端，否则会生成中间表A_B
    @org.hibernate.annotations.ForeignKey(name = "none") // 注意这里不能使用 @JoinColumn 中的 @ForeignKey 不然会生成外键
    private Set<Employee> employees;
}
```
#### 创建持久化元模型
这个可以不实现，但是在后面实现复杂查询的时候，只能手动输入相关的实体类的属性字段的字符串，然后进行强制转换类型，我认为这样相对来说好维护一些，关于持久化元模型，将在这篇文章最后部分再详细介绍下。
```java
/**
 * 不知道什么原因
 * 这个持久性单元模型需要与实体类相同的包中，否则相关的值不会注入到 spring 容器中
 *
 *
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2018/3/8 10:16</pre>
 */
@StaticMetamodel(Employee.class)
public class Employee_ {
    public static volatile SingularAttribute<Employee, Long> id;
    public static volatile SingularAttribute<Employee, EmployeeDetail> detail;
    public static volatile SingularAttribute<Employee, Job> job;
}

@StaticMetamodel(EmployeeDetail.class)
public class EmployeeDetail_ {
    public static volatile SingularAttribute<EmployeeDetail, Long> id;
    public static volatile SingularAttribute<EmployeeDetail, String> name;
    public static volatile SingularAttribute<EmployeeDetail, String> phone;
    public static volatile SingularAttribute<EmployeeDetail, Integer> age;
}

@StaticMetamodel(EmployeeDetail.class)
public class EmployeeDetail_ {
    public static volatile SingularAttribute<EmployeeDetail, Long> id;
    public static volatile SingularAttribute<EmployeeDetail, String> name;
    public static volatile SingularAttribute<EmployeeDetail, String> phone;
    public static volatile SingularAttribute<EmployeeDetail, Integer> age;
}
```
#### 创建 dao 接口，继承 `JpaSpecificationExecutor<T>`:
```java
@Repository
public interface EmployeeDao extends JpaSpecificationExecutor<Employee>, PagingAndSortingRepository<Employee, Id> {
}
```
#### 实现复杂动态查询：
```java
    /**
     * 多条件动态分页查询，
     * <p>
     * 如果后期还需要加入其他的查询的条件，可以直接添加代码逻辑就好了。
     * <p>
     * 分割，需要注意的是 spring data 还提供了<code>Specification</code> 这个类直接提供了 eq | gt | equal 等等 Specification
     * 接口的方法，但是它的方法已经过时了，不推荐使用，如果需要使用记录下
     * 来自 网站 https://www.tianmaying.com/tutorial/spring-jpa-page-sort
     * <code>
     * public Page<Person> findAll(SearchRequest request) {
     *      Specification<Person> specification = new Specifications<Person>()
     *          .eq(StringUtils.isNotBlank(request.getName()), "name", request.getName())
     *          .gt(Objects.nonNull(request.getAge()), "age", 18)
     *          .between("birthday", new Range<>(new Date(), new Date()))
     *          .like("nickName", "%og%", "%me")
     *          .build();
     *      return personRepository.findAll(specification, new PageRequest(0, 15));
     * }
     * </code>
     *
     * @param search   查询属性
     * @param pageable 分页和排序
     * @return 分页数据
     */
    @Override
    public Page<Employee> pageBySearch(EmployeeSearch search, Pageable pageable) {
        return employeeDao.findAll((Specification<Employee>) (root, criteriaQuery, criteriaBuilder) -> {
            List<Predicate> predicates = new LinkedList<>();
            Optional<EmployeeSearch> optional = Optional.ofNullable(search);
            // 根据 employee id 查询
            optional.map(EmployeeSearch::getEmployeeId).ifPresent(id -> {
                predicates.add(criteriaBuilder.equal(root.get(Employee_.id), id));
            });
            // 根据 employee detail name 模糊查询
            optional.map(EmployeeSearch::getEmployeeName).ifPresent(name -> {
                // 使用左联接，如果直接 get(Employee_.detail).get(EmployeeDetail_.name) 就是无条件内联，
                // 相当于 cross join，会产生 笛卡尔积
                Join<Employee, EmployeeDetail> join = root.join(Employee_.detail, JoinType.LEFT);
                predicates.add(criteriaBuilder.like(join.get(EmployeeDetail_.name),
                        "%" + name + "%"));
            });
            // 根据职位名查询
            optional.map(EmployeeSearch::getJobName).ifPresent(name -> {
                Join<Employee, Job> join = root.join(Employee_.job, JoinType.LEFT);
                predicates.add(criteriaBuilder.equal(join.get(Job_.name), name));
            });
            Predicate[] array = new Predicate[predicates.size()];
            return criteriaBuilder.and(predicates.toArray(array));
        }, pageable);
    }

```
`CriteriaBuilder` 有各种操作方法完成查询操作。
#### 进行测试
```java
    @Test
    public void testPageBySearch() throws Exception {
        EmployeeSearch employeeSearch = new EmployeeSearch(null, null, "程序猿");
        Sort sort = new Sort(Sort.Direction.ASC, "id");
        Page<Employee> employees = employeeService.pageBySearch(employeeSearch, new PageRequest(0, 5, sort));
        Assert.assertThat("18772383543", Matchers.equalTo(employees.getContent().get(0).getDetail().getPhone()));
    }
```
其实实现起来不是很复杂，但是用起来很舒坦，再完全不用手动拼接字符串，使用面向对象的类型检测，写起来 BUG 也少些。

### 使用 CriteriaQuery 查询和类型安全检测
#### 使用criteria 查询
例如我现在要实现一个接口，查询在大于或等于某个年纪的员工：
```java
    @PersistenceContext
    private EntityManager em;

    /**
     * Search age gt or eq the parameter
     *
     * @param age
     * @return
     */
    @Override
    public List<Employee> listByAge(Integer age) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Employee> query = cb.createQuery(Employee.class);
        Root<Employee> root = query.from(Employee.class); // 设置查询根，可以根据查询的类型设置不同的 就是 Form 语句 后面的 entity
        List<Predicate> predicates = new LinkedList<>();
        // 连表查询使用左连接
        Join<Employee, EmployeeDetail> join = root.join(Employee_.detail, JoinType.LEFT);
        predicates.add(cb.gt(join.get(EmployeeDetail_.age), age));
        predicates.add(cb.equal(join.get(EmployeeDetail_.age), age));
        // 设置排序规则
        Order order = cb.asc(root.get(Employee_.id));
        query.orderBy(order);
        query.where(cb.or(predicates.toArray(new Predicate[predicates.size()])));
        TypedQuery typedQuery = em.createQuery(query); // TypedQuery执行查询与获取元模型实例
        return typedQuery.getResultList();
    }
```
#### 构建CriteriaQuery 实例API说明
##### CriteriaBuilder 安全查询创建工厂
`CriteriaBuilder` 安全查询创建工厂,创建`CriteriaQuery`,创建查询具体具体条件`Predicate` 等
`CriteriaBuilder` 是一个工厂对象,安全查询的开始.用于构建JPA安全查询.可以从EntityManager 或`EntityManagerFactory`类中获得`CriteriaBuilder`. 
`CriteriaBuilder` 工厂类是调用`EntityManager.getCriteriaBuilder` 或 `EntityManagerFactory.getCriteriaBuilder`而得。 
##### CriteriaQuery 安全查询主语句
CriteriaQuery对象必须在实体类型或嵌入式类型上的Criteria 查询上起作用。
它通过调用 `CriteriaBuilder, createQuery` 或`CriteriaBuilder.createTupleQuery` 获得。
##### Root 定义查询的From子句中能出现的类型
AbstractQuery是CriteriaQuery 接口的父类。它提供得到查询根的方法。 
Criteria查询的查询根定义了实体类型，能为将来导航获得想要的结果，它与SQL查询中的FROM子句类似。 
Root实例也是类型化的，且定义了查询的FROM子句中能够出现的类型。 
查询根实例能通过传入一个实体类型给 AbstractQuery.from方法获得。 
Criteria查询，可以有多个查询根。 
```java
Root<Employee> employee = criteriaQuery.from(Employee.class);
```
##### Predicate 过滤条件
**过滤条件应用到SQL语句的FROM子句中。因此它是 `root` 创建的。**
在criteria 查询中，查询条件通过Predicate 或Expression 实例应用到CriteriaQuery 对象上。 
这些条件使用 CriteriaQuery .where 方法应用到CriteriaQuery 对象上。 
CriteriaBuilder 也是作为Predicate 实例的工厂，Predicate 对象通过调用CriteriaBuilder 的条件方法（ equal，notEqual， gt， ge，lt， le，between，like等）创建。 
Predicate 实例也可以用Expression 实例的 isNull， isNotNull 和 in方法获得，复合的Predicate 语句可以使用CriteriaBuilder的and, or andnot 方法构建。 
下面的代码片段展示了Predicate 实例检查年龄大于24岁的员工实例:
```java
Predicate condition = criteriaBuilder.gt(employee.get(Employee_.age), 24);
criteriaQuery.where(condition);
```
`Employee_`元模型类`age`属性，称之为路径表达式。**若`age`属性与`String`文本比较，编译器会抛出错误，这在JPQL中是不可能的。**

##### Predicate[] 多个过滤条件
支持复杂的 条件拼接， or 语句
```java
predicatesList.add(criteriaBuilder.or(criteriaBuilder.equal(root.get(RepairOrder_.localRepairStatus), LocalRepairStatus.repairing),criteriaBuilder.equal(root.get(RepairOrder_.localRepairStatus), LocalRepairStatus.diagnos)));
```
最后查询的时候
```java
 query.where(cb.or(predicates.toArray(new Predicate[predicates.size()])));
 TypedQuery typedQuery = em.createQuery(query); // TypedQuery执行查询与获取元模型实例
```
##### TypedQuery执行查询与获取元模型实例
注意，你使用EntityManager创建查询时，可以在输入中指定一个CriteriaQuery对象，它返回一个TypedQuery，它是JPA 2.0引入javax.persistence.Query接口的一个扩展，TypedQuery接口知道它返回的类型。

所以使用中,先创建查询得到TypedQuery,然后通过typeQuery得到结果.

当EntityManager.createQuery(CriteriaQuery)方法调用时，一个可执行的查询实例会创建，该方法返回指定从 criteria 查询返回的实际类型的TypedQuery 对象。

TypedQuery 接口是javax.persistence.Queryinterface.的子类型。在该片段中， TypedQuery 中指定的类型信息是Employee，调用getResultList时，查询就会得到执行 ：
```java
TypedQuery<Employee> typedQuery = em.createQuery(criteriaQuery);
List<Employee> result = typedQuery.getResultList();
```
元模型实例通过调用 `EntityManager.getMetamodel` 方法获得，`EntityType<Employee>`的元模型实例通过调用`Metamodel.entity(Employee.class)`而获得，其被传入 `CriteriaQuery.from` 获得查询根。
```java
Metamodel metamodel = em.getMetamodel();
EntityType<Employee> Employee_ = metamodel.entity(Employee.class);
Root<Employee> empRoot = criteriaQuery.from(Employee_);
```
##### Expression 用在查询语句的select，where和having子句中，该接口有 isNull, isNotNull 和 in方法
Expression对象用在查询语句的select，where和having子句中，该接口有 isNull, isNotNull 和 in方法，下面的代码片段展示了Expression.in的用法，employye的年龄检查在20或24的。 
```java
CriteriaQuery<Employee> criteriaQuery = criteriaBuilder .createQuery(Employee.class);
Root<Employee> employee = criteriaQuery.from(Employee.class);
criteriaQuery.where(employee.get(Employee_.age).in(20, 24));
em.createQuery(criteriaQuery).getResultList();
```
下面也是一个更贴切的例子:
```java
//定义一个Expression
Expression<String> exp = root.get(Employee.id);
//
List<String> strList=new ArrayList<>();	
strList.add("20");
strList.add("24");		
predicatesList.add(exp.in(strList));

criteriaQuery.where(predicatesList.toArray(new Predicate[predicatesList.size()]));
```
##### 复合谓词
Criteria Query也允许开发者编写复合谓词，通过该查询可以为多条件测试下面的查询检查两个条件。首先，name属性是否以M开头，其次，employee的age属性是否是25。逻辑操作符and执行获得结果记录。
```java
criteriaQuery.where(
 criteriaBuilder.and(
  criteriaBuilder.like(employee.get(Employee_.name), "M%"), 
  criteriaBuilder.equal(employee.get(Employee_.age), 25)
));
em.createQuery(criteriaQuery).getResultList();
```
##### 路径表达式
Root实例，Join实例或者从另一个Path对象的get方法获得的对象使用get方法可以得到Path对象，当查询需要导航到实体的属性时，路径表达式是必要的。 
**Get方法接收的参数是在实体元模型类中指定的属性。 **
##### 参数化表达式
在JPQL中，查询参数是在运行时通过使用命名参数语法(冒号加变量，如 `:age`传入的。在Criteria查询中，查询参数是在运行时创建`ParameterExpression`对象并为在查询前调用`TypeQuery,setParameter`方法设置而传入的。
```java
       ParameterExpression<Integer> ageParameter = cb.parameter(Integer.class);
        List<Predicate> predicates = new LinkedList<>();
        // 连表查询使用左连接
        Join<Employee, EmployeeDetail> join = root.join(Employee_.detail, JoinType.LEFT);
        predicates.add(cb.gt(join.get(EmployeeDetail_.age), ageParameter));
        predicates.add(cb.equal(join.get(EmployeeDetail_.age), ageParameter));
        query.where(cb.or(predicates.toArray(new Predicate[predicates.size()])));
        TypedQuery typedQuery = em.createQuery(query); // TypedQuery执行查询与获取元模型实例
        return typedQuery.setParameter(ageParameter, age).getResultList();
```
##### 排序结果
```java
        // 设置排序规则
        Order order = cb.asc(root.get(Employee_.id));
        query.orderBy(order);
```
可以设置多个 order。
##### 分组
```java
    /**
     * 分组统计重名数量
     * @param name
     * @return
     */
    @Override
    public List<Tuple> groupByName(String name) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Tuple> query = cb.createTupleQuery();
        Root<Employee> root = query.from(Employee.class);
        Join<Employee, EmployeeDetail> join = root.join(Employee_.detail, JoinType.LEFT);
        query.groupBy(join.get(EmployeeDetail_.name));
        if (name != null) {
            query.having(cb.like(join.get(EmployeeDetail_.name), "%" + name + "%"));
        }
        query.select(cb.tuple(join.get(EmployeeDetail_.name), cb.count(root)));
        TypedQuery<Tuple> typedQuery = em.createQuery(query);
        return typedQuery.getResultList();
        // print sql :
        //select employeede1_.name as col_0_0_, count(employee0_.id) as col_1_0_ from employee employee0_
        // left outer join employee_detail employeede1_ on employee0_.detail_id=employeede1_.id
        // group by employeede1_.name having employeede1_.name like ?
    }
```
##### 返回元组(Tuple)的查询
查询的时候需要查询 `单列` 的记录可以使用元组，
```java
CriteriaQuery<Tuple> criteriaQuery = criteriaBuilder.createTupleQuery();
   Root<Employee> employee = criteriaQuery.from(Employee.class);
   criteriaQuery.multiselect(employee.get(Employee_.name).alias("name"), employee.get(Employee_.age).alias("age"));
   em.createQuery(criteriaQuery).getResultList();
```
##### 使用 construct()
使用一个不是实体类来装载 查询出来的数据，但是必须要的是实体类必须有相应的构造函数才行，还需要注意的是，**该装载类必须继承实体类**
1. 首先实现一个装载数据的类，我只用来装载 name, age 两个属性即可，当然需要更多也可以设计的，毕竟我们继承了实体类，如果实体类有的字段我们可以不需要再制造了。
```java
@Data
@AllArgsConstructor
public class EmployeeResult extends Employee {
    private String name;
    private Integer age;
}
```
2. 编写查询的业务代码，很方便：
```java
    /**
     * 使用 构造函数 装载查询出来的数据
     *
     * @return
     */
    @Override
    public List<EmployeeResult> findEmployee() {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Employee> query = cb.createQuery(Employee.class);
        Root<Employee> root = query.from(Employee.class); // 设置查询根，可以根据查询的类型设置不同的
        Join<Employee, EmployeeDetail> join = root.join(Employee_.detail, JoinType.LEFT);
        // 使用构造函数 CriteriaBuilder.construct 来完成装载数据
        query.select(cb.construct(EmployeeResult.class, join.get(EmployeeDetail_.name), join.get(EmployeeDetail_.age)));
        // 设置排序规则
        Order order = cb.asc(root.get(Employee_.id));
        query.orderBy(order);
        TypedQuery typedQuery = em.createQuery(query); // TypedQuery执行查询与获取元模型实例
        return typedQuery.getResultList();
    }
```
##### 返回 Object[] 
Criteria查询也能通过设置值给CriteriaBuilder.array方法返回 `Object[]`的结果。
```java
 criteriaQuery.select(criteriaBuilder.array(root.get(xxx),join.get(xxx)));
```
### 关于持久化元模型
在JPA中,标准查询是以元模型的概念为基础的.元模型是为具体持久化单元的受管实体定义的.这些实体可以是实体类,嵌入类或者映射的父类.提供受管实体元信息的类就是元模型类. 
描述受管类的状态和他们之间的关系的静态元模型类可以 
1. 从注解处理器产生 
2. 从程序产生 
3. 用EntityManager访问. 

元模型类描述持久化类的元数据。如果一个类安装 JPA 2.0 规范精确地描述持久化实体的元数据，那么该元模型类就是 规范的。规范的元模型类是 静态的，因此它的所有成员变量都被声明为 静态的（也是 public的）。
Employee类的标准元模型类的名字将是使用 javax.persistence.StaticMetamodel注解的Employee_。元模型类的属性全部是static和public的。Employee的每一个属性都会使用在JPA2规范中描述的以下规则在相应的元模型类中映射： 
* 诸如id，name和age的非集合类型，会定义静态属性SingularAttribute<A, B> b，这里b是定义在类A中的类型为B的一个对象。 
* 对于Addess这样的集合类型，会定义静态属性`ListAttribute<A, B> b`，这里List对象b是定义在类A中类型B的对象。其它集合类型可以是`SetAttribute`, `MapAttribute` 或 `CollectionAttribute` 类型。 

### 简单总结下
1. 这次搭建的 jpa 框架是自动生成的数据库表，中间出了很多叉子，例如，自动生成了中间表，不想要外键，给你自己生成外键，而且还不好解决，最终通过谷歌终于找到了解决办法，相应的网页，也在代码上标注出来了，还是基础不行。
2. 主要理解怎么从 `CriteriaBuilder` 一步步的在下面创建查询的条件，例如 查询类型 `root`，查询语句 `CriteriaQuery ` ,查询条件 `Predicate ` ，这样就很容易构建一个 criteria 查询。
3. 了解了 JPA 复杂查询中 Specification 接口，给人第一体验就是完全面向对象，包括类型检查，基本上代码写了一遍就能编译运行它通过。再多条件动态条件查询的时候，利用 java 8 的 Optional 的空指针检查，比较方便，不用再被长长的 SQL 拼接发麻了。


### 参考文章 
* [JPA 2.0 中的动态类型安全查询](https://www.ibm.com/developerworks/cn/java/j-typesafejpa/)
* [JPA criteria 查询:类型安全与面向对象](https://my.oschina.net/zhaoqian/blog/133500)
