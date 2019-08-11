---
title: 学习Spring Boot：（二十）使用 MongoDB
date: 2018-03-03 22:08:08
tags: [Spring Boot,mongodb]
categories: 学习笔记
[1075199251,张凯,18772383543,wuwii,追光者w,有梦想的咸鱼,一棵树站在原野上,Slience]
---

### 前言
>MongoDB [^1] 是可以应用于各种规模的企业、各个行业以及各类应用程序的开源数据库。基于分布式文件存储的数据库。由C++语言编写。旨在为WEB应用提供可扩展的高性能数据存储解决方案。MongoDB是一个高性能，开源，无模式的文档型数据库，是当前NoSql数据库中比较热门的一种。

<!--more-->

[^1]: 来自于英文单词“Humongous”，中文含义为“庞大”

### 正文
Spring Boot 对 MongoDB 的数据源操作进行了封装。
#### 加入依赖
在 pom.xml 加入：
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>
```
#### 配置连接参数
在系统配置文件中配置：
```yaml
spring:
  data:
    mongodb:
      uri: mongodb://wuwii:123456@localhost:27017/learn
```

#### 测试使用
1. 创建实体
```java
@Data
@Document(collection = "pet") // 标识要持久化到MongoDB的域对象。模型名是 pet
public class Pet implements Serializable {
    @Id
    //@Indexed(unique = true) // 使用MongoDB的索引特性标记一个字段
    private Long id;
    @Field("pet_name") //自定义设置对应MongoDB中的key
    private String name;
    private String species;
}
```
2. 创建 dao 接口完成基础操作
```java
@Repository
public class PetDaoImpl implements PetDao {
    @Autowired
    private MongoTemplate mongoTemplate;

    @Override
    public Pet find(Long id) {
        return mongoTemplate.findById(id, Pet.class);
    }

    @Override
    public List<Pet> findAll() {
        return mongoTemplate.findAll(Pet.class);
    }

    @Override
    public void add(Pet pet) {
        mongoTemplate.insert(pet);
    }

    @Override
    public void update(Pet pet) {
        Query query = new Query();
        Criteria criteria = new Criteria("id");
        criteria.is(pet.getId());
        query.addCriteria(criteria);
        Update update = new Update();
        update.set("pet_name", pet.getName())
                .set("species", pet.getSpecies());
        mongoTemplate.updateFirst(query, update, Pet.class); // 条件，更新的数据，更新的类型
    }

    @Override
    public void delete(Long id) {
        Criteria criteria = new Criteria("id");
        criteria.is(id);
        Query query = new Query();
        query.addCriteria(criteria);
        mongoTemplate.remove(query, Pet.class); // 删除的条件、删除的类型
    }
}

```
3. 简单测试下

```java
@SpringBootTest
@RunWith(SpringRunner.class)
@FixMethodOrder(MethodSorters.NAME_ASCENDING)
public class PetDaoTest {

    @Autowired
    private PetDao petDao;

    private Pet pet;

    @Before
    public void before() {
        pet = new Pet();
        pet.setId(1L);
        pet.setName("Tom");
        pet.setSpecies("cat");
    }

    @After
    public void after() {
    }

    @Test
    public void test01Add() {
        Pet pet = new Pet();
        pet.setId(1L);
        pet.setName("Tom");
        pet.setSpecies("cat");
        petDao.add(pet);
    }

    @Test
    public void test02Find() {
        Assert.assertThat(pet, Matchers.equalTo(petDao.find(pet.getId())));
    }

    @Test
    public void test03FindAll() {
        System.out.println(petDao.findAll());
    }

    @Test
    public void test04Update() {
        pet.setName("KronChan");
        petDao.update(pet);
        Assert.assertThat(pet, Matchers.equalTo(petDao.find(pet.getId())));
    }

    @Test
    public void test05Delete() {
        petDao.delete(pet.getId());
        Assert.assertThat(null, Matchers.equalTo(petDao.find(pet.getId())));
    }

}

```
#### 去数据库验证结果
```json
> use learn
switched to db learn
> db.pet.find()
{ "_id" : NumberLong(1), "_class" : "com.wuwii.testmongodb.Pet", "pet_name" : "KronChan", "species" : "cat" }
```

### 多数据源的使用
未完成
