---
title: 学习Spring Boot：（十二）Mybatis 中自定义枚举转换器
date: 2018-02-23 10:08:13
tags: [Spring Boot]
categories: 学习笔记
---

### 前言
在 Spring Boot 中使用 Mybatis 中遇到了字段为枚举类型，数据库存储的是枚举的值，发现它不能自动装载。

<!--more-->

### 解决
#### 内置枚举转换器
MyBatis内置了两个枚举转换器分别是：`org.apache.ibatis.type.EnumTypeHandler` 和 `org.apache.ibatis.type.EnumOrdinalTypeHandler`。

##### EnumTypeHandler
mybatis 中默认的枚举转换器，是获取枚举中的 `name` 属性。

##### EnumOrdinalTypeHandler
获取枚举中 `ordinal` 属性，就是例如索引一样的东西，不过是从 1 开始递增的。

因此上面提供的两种的转换器都不能满足我们的需求，我们需要自定义一个转换器。

#### 自定义枚举转换器
MyBatis提供了 `org.apache.ibatis.type.BaseTypeHandler` 类用于我们自己扩展类型转换器，上面的`EnumTypeHandler和EnumOrdinalTypeHandler` 也都实现了这个接口。

继承 `BaseTypeHandler<T>` 一共需要实现4个方法：  
1. `void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType)`  
    用于定义设置参数时，该如何把Java类型的参数转换为对应的数据库类型；
2. `T getNullableResult(ResultSet rs, String columnName)`  
    用于定义通过字段名称获取字段数据时，如何把数据库类型转换为对应的Java类型； 
3. `T getNullableResult(ResultSet rs, int columnIndex)`   
    用于定义通过字段索引获取字段数据时，如何把数据库类型转换为对应的Java类型； 
4. `T getNullableResult(CallableStatement cs, int columnIndex)`  
    用定义调用存储过程后，如何把数据库类型转换为对应的Java类型。

##### 定义一个枚举通用行为
定义一个枚举通用行为，规范枚举的实现。
```java
public interface BaseEnum<E extends Enum<?>, T> {
    /**
     * 获取枚举的值
     * @return 枚举的值
     */
    T getValue();
}
```

定义自己需要的枚举：
```java
public class SysConstant {

    /**
     * 人员状态
     */
    public enum SysUserStatus implements BaseEnum<SysUserStatus, String> {
        /**
         * 账户已经激活（默认）
         */
        ACTIVE("1"),
        /**
         * 账户锁定
         */
        LOCK("0");

        private String value;

        private SysUserStatus(String value) {
            this.value = value;
        }

        @Override
        public String getValue() {
            return value;
        }
    }

    /**
     * 人员类型
     */
    public enum SysUserType implements BaseEnum<SysUserType, String> {
        /**
         * 普通用户
         */
        USER("1"),
        /**
         * 系统管理员
         */
        ADMIN("0");

        private String value;

        private SysUserType(String value) {
            this.value = value;
        }

        @Override
        public String getValue() {
            return value;
        }
    }
}
```

##### 实现自定义转换器
自定义一个基本的枚举转换器工具，如果有其他需求可以在这个基类上自定义。
```java
package com.wuwii.common.util;

import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;

import java.sql.CallableStatement;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.Objects;

/**
 * 参考 http://blog.csdn.net/fighterandknight/article/details/51520595 
 * 进行对本地项目的优化
 * <p>
 * 解决 Mybatis 中枚举的问题，
 * 获取 ResultSet 的值都是获取字符串的，然后比较字符串，以便通用。
 *
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2018/2/9 17:26</pre>
 */
public abstract class BaseEnumTypeHandler<E extends Enum<E> & BaseEnum> extends BaseTypeHandler<E> {
    /**
     * 枚举的class
     */
    private Class<E> type;
    /**
     * 枚举的每个子类枚
     */
    private E[] enums;

    /**
     * 一定要有默认的构造函数，不然抛出 not found method 异常
     */
    public BaseEnumTypeHandler() {
    }

    /**
     * 设置配置文件设置的转换类以及枚举类内容，供其他方法更便捷高效的实现
     *
     * @param type 配置文件中设置的转换类
     */
    public BaseEnumTypeHandler(Class<E> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.type = type;
        this.enums = type.getEnumConstants();
        if (this.enums == null) {
            throw new IllegalArgumentException(type.getSimpleName()
                    + " does not represent an enum type.");
        }
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, E parameter,
                                    JdbcType jdbcType) throws SQLException {
        /*
         * BaseTypeHandler已经帮我们做了parameter的null判断
         * 数据库存储的是枚举的值，所以我们这里使用 value ， 如果需要存储 name，可以自定义修改
         */
        if (jdbcType == null) {
            ps.setString(i, Objects.toString(parameter.getValue()));
        } else {
            ps.setObject(i, parameter.getValue(), jdbcType.TYPE_CODE);
        }
    }

    @Override
    public E getNullableResult(ResultSet rs, String columnName)
            throws SQLException {
        String i = rs.getString(columnName);
        if (rs.wasNull()) {
            return null;
        } else {
            return locateEnumStatus(i);
        }
    }

    @Override
    public E getNullableResult(ResultSet rs, int columnIndex)
            throws SQLException {
        String i = rs.getString(columnIndex);
        if (rs.wasNull()) {
            return null;
        } else {
            return locateEnumStatus(i);
        }
    }

    @Override
    public E getNullableResult(CallableStatement cs, int columnIndex)
            throws SQLException {
        String i = cs.getString(columnIndex);
        if (cs.wasNull()) {
            return null;
        } else {
            return locateEnumStatus(i);
        }
    }

    /**
     * 枚举类型转换，由于构造函数获取了枚举的子类 enums，让遍历更加高效快捷，
     * <p>
     * 我将取出来的值 全部转换成字符串 进行比较，
     *
     * @param value 数据库中存储的自定义value属性
     * @return value 对应的枚举类
     */
    private E locateEnumStatus(String value) {
        for (E e : enums) {
            if (Objects.toString(e.getValue()).equals(value)) {
                return e;
            }
        }
        throw new IllegalArgumentException("未知的枚举类型：" + value + ",请核对"
                + type.getSimpleName());
    }
}

```
##### 配置转换器
将枚举转换器编写完成后，我们需要定义它对哪些枚举进行转换。
可以在Mybatis 配置文件配置
```xml
<typeHandlers>
    <typeHandler handler="com.example.typeHandler.CodeEnumTypeHandler" javaType="com.example.entity.enums.ComputerState"/>
</typeHandlers>

```
###### 优化
在MyBatis中添加typeHandler用于指定哪些类使用我们自定义的转换器，**一旦系统中的枚举类多了起来，MyBatis的配置文件维护起来会变得非常麻烦，也容易出错**。

**方法一**：  
定义一个 `EnumTypeHandler` 去继承我们的 `BaseEnumTypeHandler`。然后使用注解 `@MappedTypes` 类型转换。
```java
package com.wuwii.module.sys.common.handle;

import com.wuwii.common.handle.BaseEnumTypeHandler;
import com.wuwii.common.util.BaseEnum;
import com.wuwii.module.sys.common.util.SysConstant;
import org.apache.ibatis.type.MappedTypes;

/**
 * 枚举转换的公共模块
 *
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2018/2/9 18:12</pre>
 */
@MappedTypes(value = {SysConstant.SysUserStatus.class, SysConstant.SysUserType.class})
public class SysEnumTypeHandler<E extends Enum<E> & BaseEnum> extends BaseEnumTypeHandler<E> {
    /**
     * 设置配置文件设置的转换类以及枚举类内容，供其他方法更便捷高效的实现
     *
     * @param type 配置文件中设置的转换类
     */
    public SysEnumTypeHandler(Class<E> type) {
        super(type);
    }
}
```
需要在系统配置文件中配置
```yml
# 多个模块的的多个 包配置可以使用逗号分开。
mybatis:
    typeHandlersPackage: com.wuwii.module.sys.common.handle,com.wuwii.module.task.common.handle
```

**方法二**：
如果你的项目中自定义了 `SqlSessionFactory`，推荐使用下面这种方式，一次使用无需多次配置。

<a rel="external nofollow" target="_blank" href="https://segmentfault.com/a/1190000010755321"> 如何在MyBatis中优雅的使用枚举 </a> 

在 =》 方案 6. 优化

**方法三**：  
还有个人修改源码实现的，有兴趣的可以看看：  

<a rel="external nofollow" target="_blank" href="http://blog.csdn.net/fighterandknight/article/details/51600997"> 修改MyBatis源码实现扫描注册枚举-具体实现  </a> 

### 参考文章
* <a rel="external nofollow" target="_blank" href="https://segmentfault.com/a/1190000010755321"> 如何在MyBatis中优雅的使用枚举 </a> 
* <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/u014044812/article/details/78258730"> SpringBoot Mybatis EnumTypeHandler自定义统一处理器</a> 
