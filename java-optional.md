---
title: Java 8 Optional类的分析与使用
date: 2017-11-12 21:08:03
tags: [java]
categories: 学习笔记
---

``Optional`` 类 是``jdk 1.8``后新添加的特性，阿里巴巴的代码规范也明确说明了使用 Optional 来防止NPE。

>Optional 类是一个可以为null的容器对象。如果值存在则isPresent()方法会返回true，调用get()方法会返回该对象。
Optional 是个容器：它可以保存类型T的值，或者仅仅保存null。Optional提供很多有用的方法，这样我们就不用显式进行空值检测。  
Optional 类的引入很好的解决空指针异常。

<!--more-->

它拥有的方法：
![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwxaeo6upj20f60g5ab9.jpg)

### 属性    
```java
// @since 1.8
public final class Optional<T> {
    /**
     * Common instance for {@code empty()}.
     */
    private static final Optional<?> EMPTY = new Optional<>();

    /**
     * If non-null, the value; if null, indicates no value is present
     */
    private final T value;
}
```
value 属性就是存储数据的地方。如果为null，表示没有值的存在，取值的时候如果没有默认值，会抛出空指针。


### 方法
#### 构造方法
它拥有两个构造方法：

```java
    /**
     * Constructs an empty instance.
     *
     * @implNote Generally only one empty instance, {@link Optional#EMPTY},
     * should exist per VM.
     */
    private Optional() {
        this.value = null;
    }

    /**
     * Constructs an instance with the value present.
     *
     * @param value the non-null value to be present
     * @throws NullPointerException if value is null
     */
    private Optional(T value) {
        this.value = Objects.requireNonNull(value);
    }
```
* 第一个构造一个空的Optional；
* 第二个构造一个值为value的Optional，值为null会抛出NPE；

#### of
>为非null的值创建一个Optional。

of方法通过工厂方法创建Optional类。需要注意的是，创建对象时传入的参数不能为null。如果传入参数为null，则抛出NullPointerException。
源码：
```java
public static <T> Optional<T> of(T value) {
        return new Optional<>(value);
}
```
可以看出它最后调用的是第二个有参的构造函数，所以它传入的参数也为null会抛出空指针。

```java
        Optional<String> spring = Optional.of("SPRING");
        Optional<String> emptyStr = Optional.of("");
        Optional<String> nullValue = Optional.of(null);
```
最后个创建Optional实例会抛出空指针异常。
#### ofNullable

```java
    /**
     * Returns an {@code Optional} describing the specified value, if non-null,
     * otherwise returns an empty {@code Optional}.
     *
     * @param <T> the class of the value
     * @param value the possibly-null value to describe
     * @return an {@code Optional} with a present value if the specified value
     * is non-null, otherwise an empty {@code Optional}
     */
    public static <T> Optional<T> ofNullable(T value) {
        return value == null ? empty() : of(value);
    }
```
为指定的值创建一个Optional，如果指定的值为null，则返回一个空的Optional。
相比较of 方法，能够接受 null 参数。
#### isPresent
判断值是否存在。值不为null，返回true。
```java
public boolean isPresent() {
        return value != null;
}
```
#### get
取出存在Optional 中的值，为Null 抛出``NoSuchElementException``异常
```java
    public T get() {
        if (value == null) {
            throw new NoSuchElementException("No value present");
        }
        return value;
}
```
#### ifPresent
```java
    /**
     * If a value is present, invoke the specified consumer with the value,
     * otherwise do nothing.
     *
     * @param consumer block to be executed if a value is present
     * @throws NullPointerException if value is present and {@code consumer} is
     * null
     */
    public void ifPresent(Consumer<? super T> consumer) {
        if (value != null)
            consumer.accept(value);
    }
```
对传入的值使用``Consumer``接口的accept方法进行处理，
```java
    /**
     * Performs this operation on the given argument.
     *
     * @param t the input argument
     */
    void accept(T t);
```

实际上就是可以使用函数式编程了，使用lambda表达式方法了，前提是
```
spring.ifPresent(a -> System.out.println(a.indexOf("I")));
```
结果为 ``3``。

#### orElse

如果有值则将其返回，否则返回指定的其它值。

```java
System.out.printf("有值的Optional: %s，没值的Optional：%s%n",
                spring.orElse("summer"), nullValue.orElse("summer"));
```
打印结果：
```
有值的Optional: SPRING，没值的Optional：summer
```

#### orElseGet
orElseGet与orElse方法类似，区别在于得到的默认值。orElse方法将传入的字符串作为默认值，orElseGet方法可以接受Supplier接口的实现用来生成默认值，由于参数是接口形式，直接使用lambda表达式，更方便。
能接收函数式返回处理的数据。
```java
System.out.printf("有值的Optional: %s，没值的Optional：%s%n",
                spring.orElseGet(() -> "summer"),
                nullValue.orElseGet(() -> "summer"));
```
输出：
```
有值的Optional: SPRING，没值的Optional：summer
```
#### orElseThrow
如果有值则将其返回，否则抛出supplier接口创建的异常。
在上面的 orElseGet 方法中，传入的是Supplier接口的实现，在orElseThrow中传入一个Throwable ，如果值不存在来抛出传入的指定类型异常，源码：
```java
    public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {
        if (value != null) {
            return value;
        } else {
            throw exceptionSupplier.get();
        }
    }
```
使用方法：
```java
        //orElseThrow
        System.out.printf("orElseThrow有值的Optional: %s，没值的Optional：%s%n",
                spring.orElseThrow(OptionalThrowable::new),
                nullValue.orElseThrow(OptionalThrowable::new));

/**
 * 自定义的Optional异常类
 */
class OptionalThrowable extends Throwable {
    public OptionalThrowable() {
    }

    public OptionalThrowable(String message) {
        super(message);
    }

    @Override
    public String getMessage() {
        //return super.getMessage();
        return "这个Optional 是空值";
    }
}


```
结果符合预期错误抛出结果：
```
有值的Optional: SPRING，没值的Optional：summer
Exception in thread "main" com.wuwii.utils.OptionalThrowable: 这个Optional 是空值
	at java.util.Optional.orElseThrow(Optional.java:290)
	at com.wuwii.utils.OptionalTest.main(OptionalTest.java:28)
```
#### map
>如果有值，则对其执行调用mapping函数得到返回值。如果返回值不为null，则创建包含mapping返回值的Optional作为map方法返回值，否则返回空Optional。

map就是stream中的方法一样的，是用来操作的，用来对Optional实例的值执行一系列操作，所以我们可以灵活的使用Function包的方法和lamdba表达式。

```java
//map
        Optional<String> castedOptional = spring.map(String::toLowerCase);
        System.out.printf("转换过后的值：%s%n", castedOptional.orElseGet(null));
```
输出结果：
```
转换过后的值：spring
```
可以看出转换成小写的了。
#### flatMap
>如果有值，为其执行mapping函数返回Optional类型返回值，否则返回空Optional。flatMap与map（Funtion）方法类似，区别在于flatMap中的mapper返回值必须是Optional。调用结束时，flatMap不会对结果用Optional封装。

```java
//flatMap
        Optional<String> upperOptional = castedOptional.flatMap(a -> Optional.of(a.toUpperCase()));
        System.out.printf("将上面小写的castedOptional 转换成大写：%s%n", upperOptional.orElseGet(null));
```
输出结果：
```
将上面小写的castedOptional 转换成大写：SPRING
```

#### filter
>如果有值并且满足断言条件返回包含该值的Optional，否则返回空Optional。

过滤，对于filter函数我们应该传入实现了Predicate接口的lambda表达式。

```java
/filter
        //过滤掉长度不大于10的，SPRING长度小于10，故此被过滤了
        Optional<String> filterOptional = upperOptional.filter(a -> a.length() > 10);
        System.out.printf("过滤掉长度不大于10的 ：%s%n", filterOptional.orElse("Default value"));
    }
```
输出结果：
```
过滤掉长度不大于10的结果 ：Default value
```

### 学习的所有代码
```java
package com.wuwii.utils;

import java.util.Optional;

/**
 * 学习Optional
 * @author: Zhang Kai
 * @since : 2017/11/12 20:40
 */
public class OptionalTest {
    public static void main(String[] args) {
        //of
        Optional<String> spring = Optional.of("SPRING");
        Optional<String> emptyStr = Optional.of("");
		//会抛出异常NPE
        //Optional<String> nullValue1 = Optional.of(null); 
		//不会抛异常，做了判断
        Optional<String> nullValue = Optional.ofNullable(null);
		
        //ifPresent
        spring.ifPresent(a -> System.out.println(a.indexOf("I")));
		
        //orElse
        System.out.printf("orElse有值的Optional: %s，没值的Optional：%s%n",
                spring.orElse("summer"), nullValue.orElse("summer"));
				
        //orElseGet
        System.out.printf("orElseGet有值的Optional: %s，没值的Optional：%s%n",
                spring.orElseGet(() -> "summer"),
                nullValue.orElseGet(() -> "summer"));
				
        //orElseThrow
		// 这段代码会抛出异常，为了下面能运行，先注释。
        /*try {
            System.out.printf("orElseThrow有值的Optional: %s，没值的Optional：%s%n",
                    spring.orElseThrow(OptionalThrowable::new),
                    nullValue.orElseThrow(OptionalThrowable::new));
        } catch (OptionalThrowable optionalThrowable) {
            optionalThrowable.printStackTrace();
        }*/

        //map
        Optional<String> castedOptional = spring.map(String::toLowerCase);
        System.out.printf("转换成小写的值：%s%n", castedOptional.orElseGet(null));

        //flatMap
        Optional<String> upperOptional = castedOptional.flatMap(a -> Optional.of(a.toUpperCase()));
        System.out.printf("将上面小写的castedOptional 转换成大写：%s%n", upperOptional.orElseGet(null));

        //filter
        //过滤掉长度不大于10的，SPRING长度小于10，故此被过滤了
        Optional<String> filterOptional = upperOptional.filter(a -> a.length() > 10);
        System.out.printf("过滤掉长度不大于10的结果 ：%s%n", filterOptional.orElse("Default value"));
    }
}

/**
 * 自定义的Optional异常类
 */
class OptionalThrowable extends Throwable {
    public OptionalThrowable() {
    }

    public OptionalThrowable(String message) {
        super(message);
    }

    @Override
    public String getMessage() {
        //return super.getMessage();
        return "这个Optional 是空值";
    }
}

```
