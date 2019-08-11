---
title: 学习Spring Boot：（八）Mybatis使用分页插件PageHelper
date: 2018-02-23 10:08:03
tags: [Spring Boot]
categories: 学习笔记
[1075199251,张凯,18772383543,wuwii,追光者w,有梦想的咸鱼,一棵树站在原野上,Slience]
---

首先Mybqtis可以通过SQL 的方式实现分页很简单，只要在查询SQL 后面加上`limit #{currIndex} , #{pageSize}`就可以了。

本文主要介绍使用**拦截器**的方式实现分页。

<!--more-->

### 实现原理
拦截器实现了拦截所有查询需要分页的方法，并且利用获取到的分页相关参数统一在sql语句后面加上limit分页的相关语句，从而达到SQL 分页的目的，它的好处不用多说了，代码也写的很少，对SQL 的侵入较少，推荐使用。

### 步骤
#### 添加依赖
```xml
<pagehelper.version>1.2.3</pagehelper.version>

<dependency>
	<groupId>com.github.pagehelper</groupId>
	<artifactId>pagehelper-spring-boot-starter</artifactId>
	<version>${pagehelper.version}</version>
</dependency>
```

#### 配置文件
在系统配置文件中加入pagehelper的配置信息：
```yaml
pagehelper:
  helper-dialect: mysql
  reasonable: true
  supportMethodsArguments: true
  params: count=countSql
```
针对`pagehelper`的配置参数，查询了一下：
1. `helperDialect`：分页插件会自动检测当前的数据库链接，自动选择合适的分页方式。
    你可以配置`helperDialect`属性来指定分页插件使用哪种方言。配置时，可以使用下面的缩写值：  
    `oracle`,`mysql`,`mariadb`,`sqlite`,`hsqldb`,`postgresql`,`db2`,`sqlserver`,`informix`,`h2`,`sqlserver2012`,`derby`  
    <b>特别注意：</b>使用 SqlServer2012 数据库时，需要手动指定为 `sqlserver2012`，否则会使用 SqlServer2005 的方式进行分页。  
    你也可以实现 `AbstractHelperDialect`，然后配置该属性为实现类的全限定名称即可使用自定义的实现方法。

2. `offsetAsPageNum`：默认值为 `false`，该参数对使用 `RowBounds` 作为分页参数时有效。
    当该参数设置为 `true` 时，会将 `RowBounds` 中的 `offset` 参数当成 `pageNum` 使用，可以用页码和页面大小两个参数进行分页。

3. `rowBoundsWithCount`：默认值为`false`，该参数对使用 `RowBounds` 作为分页参数时有效。
    当该参数设置为`true`时，使用 `RowBounds` 分页会进行 count 查询。

4. `pageSizeZero`：默认值为 `false`，当该参数设置为 `true` 时，如果 `pageSize=0` 或者 `RowBounds.limit = 0` 就会查询出全部的结果（相当于没有执行分页查询，但是返回结果仍然是 `Page` 类型）。

5. `reasonable`：分页合理化参数，默认值为`false`。当该参数设置为 `true` 时，`pageNum<=0` 时会查询第一页，
    `pageNum>pages`（超过总数时），会查询最后一页。默认`false` 时，直接根据参数进行查询。

6. `params`：为了支持`startPage(Object params)`方法，增加了该参数来配置参数映射，用于从对象中根据属性名取值，
    可以配置 `pageNum,pageSize,count,pageSizeZero,reasonable`，不配置映射的用默认值，
    默认值为`pageNum=pageNum;pageSize=pageSize;count=countSql;reasonable=reasonable;pageSizeZero=pageSizeZero`。

7. `supportMethodsArguments`：支持通过 Mapper 接口参数来传递分页参数，默认值`false`，分页插件会从查询方法的参数值中，自动根据上面 `params` 配置的字段中取值，查找到合适的值时就会自动分页。
    使用方法可以参考测试代码中的 `com.github.pagehelper.test.basic` 包下的 `ArgumentsMapTest` 和 `ArgumentsObjTest`。

8. `autoRuntimeDialect`：默认值为 `false`。设置为 `true` 时，允许在运行时根据多数据源自动识别对应方言的分页
    （不支持自动选择`sqlserver2012`，只能使用`sqlserver`），用法和注意事项参考下面的**场景五**。

9. `closeConn`：默认值为 `true`。当使用运行时动态数据源或没有设置 `helperDialect` 属性自动获取数据库类型时，会自动获取一个数据库连接，
    通过该属性来设置是否关闭获取的这个连接，默认`true`关闭，设置为 `false` 后，不会关闭获取的连接，这个参数的设置要根据自己选择的数据源来决定。

**重要提示：**

当 `offsetAsPageNum=false` 的时候，由于 `PageNum` 问题，`RowBounds`查询的时候 `reasonable` 会强制为 `false`。使用 `PageHelper.startPage` 方法不受影响。
**注：** `PageRowBounds` 想要查询总数也需要配置该属性为 `true`。

#### 使用
在业务查询 的时候加上
```java
PageHelper.startPage(pageIndex, pageSize);
```
例如：
```java
@Override
    public List<SysUserEntity> query(SysUserEntity user) {
        // 查询第一页的两条数据
        PageHelper.startPage(1, 2);
        return sysUserDao.query(user);
    }
```
测试一下返回结果：

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/springboot-pagehelper/1.png)

我们来看下它执行的SQL :
```sql
Preparing: SELECT count(0) FROM sys_user 
Parameters: 
     Total: 1
Preparing: SELECT * FROM sys_user LIMIT ? 
Parameters: 2(Integer)
     Total: 2
```

### 补充
#### 使用方法
**使用的时候，需要仔细阅读作者的文章[
pagehelper/Mybatis-PageHelper](https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md)**  

主要是阅读使用方法，以及各个场景中使用什么方法解决问题，注意事项。

#### 分页排序
```java
PageHelper.startPage(pageIndex, pageSize, orderBy);
```

#### 支持返回PageInfo
```java
    @Override
    public PageInfo queryByPageInfo(SysUserEntity user) {
        return PageHelper.startPage(1,2).doSelectPageInfo(() -> sysUserDao.query(user));
    }
```
使用Swagger测试返回数据：
```json
{
  "pageNum": 1,
  "pageSize": 2,
  "size": 2,
  "startRow": 1,
  "endRow": 2,
  "total": 4,
  "pages": 2,
  "list": [
    {
      "id": 1,
      "username": "def",
      "password": "123",
      "mobile": null,
      "email": null,
      "createUserId": null,
      "createDate": null
    },
    {
      "id": 7,
      "username": "wuwii",
      "password": "123",
      "mobile": null,
      "email": null,
      "createUserId": null,
      "createDate": null
    }
  ],
  "prePage": 0,
  "nextPage": 2,
  "isFirstPage": true,
  "isLastPage": false,
  "hasPreviousPage": false,
  "hasNextPage": true,
  "navigatePages": 8,
  "navigatepageNums": [
    1,
    2
  ],
  "navigateFirstPage": 1,
  "navigateLastPage": 2,
  "firstPage": 1,
  "lastPage": 2
}
```
