---
title: Java 8中 lambda 表达式和 function包的函数式接口
date: 2017-10-20 17:18:03
tags: [java,lambda]
categories: 学习笔记
---

### lambda 表达式

java 中``lambda``表达式 实在 java 8 版本后新加入的特性，Lambda 允许把函数作为一个方法的参数（函数作为参数传递进方法中）。

**特征:**
* **可选类型声明**：不需要声明参数类型，编译器可以统一识别参数值。
* **可选的参数圆括号**：一个参数无需定义圆括号，但多个参数需要定义圆括号。
* **可选的大括号**：如果主体包含了一个语句，就不需要使用大括号。
* **可选的返回关键字**：如果主体只有一个表达式返回值则编译器会自动返回值，大括号需要指定明表达式返回了一个数值。
<!--more-->
总结语法就是：

```java
(params) -> expression
(params) -> statement
(params) -> { statements }
```

#### 使用lambda表达式替换匿名类
以 Runnable 为例
```java
        //before java8
        new Thread(new Runnable() {

            /**
             * When an object implementing interface <code>Runnable</code> is used
             * to create a thread, starting the thread causes the object's
             * <code>run</code> method to be called in that separately executing
             * thread.
             * <p>
             * The general contract of the method <code>run</code> is that it may
             * take any action whatsoever.
             *
             * @see Thread#run()
             */
            public void run() {
                System.out.println("before jdk 1.8;");
            }
        }).start();

        // after jdk 1.8
        new Thread(() -> System.out.println("after jdk 1.8;")).start();
```
执行结果是：
```
before jdk 1.8;
after jdk 1.8;
```
#### 使用lambda表达式 迭代
以 forEach 为例，迭代所有对象
```java
List<String> list1 = Arrays.asList("spring", "summer", "autumn", "winter");
        //before java8
        for (String s : list1) {
            System.out.println("before: " + s);
        }
        //after
        list1.forEach(n -> System.out.println("after: " + n));
        //list1.forEach(System.out::println); //可以打印，方法引用由::双冒号操作符标示，
```

打印结果：
```
before: spring
before: summer
before: autumn
before: winter
after: spring
after: summer
after: autumn
after: winter

```
#### 使用lambda表达式和函数式接口Predicate
除了在语言层面支持函数式编程风格，Java 8也添加了一个包，叫做 java.util.function。它包含了很多类，用来支持Java的函数式编程。其中一个便是Predicate，使用 java.util.function.Predicate 函数式接口以及lambda表达式，可以向API方法添加逻辑，用更少的代码支持更多的动态行为。下面是Java 8 Predicate 的例子，展示了过滤集合数据的多种常用方法。Predicate接口非常适用于做过滤。
```java
List<String> list1 = Arrays.asList("spring", "summer", "autumn", "winter");
        System.out.println("Print which end with n: ");
        filter(list1, str -> (str + "").endsWith("n"));

        System.out.println("Print which start with s: ");
        filter(list1, str -> (str + "").startsWith("s"));

        System.out.println("Print whose length greater than 6: ");
        filter(list1, str -> (str + "").length() > 6);

        System.out.println("Print all:");
        filter(list1, str -> true);

        System.out.println("Print none:");
        filter(list1, str -> false);
        
        
 public static void filter (List list, Predicate condition) {
        list.stream().
                filter(s -> condition.test(s)).
                forEach(s -> System.out.println(s));
    }
```
打印结果：
```
Print which end with n: 
autumn
Print which start with s: 
spring
summer
Print whose length greater than 6: 
Print all:
spring
summer
autumn
winter
Print none:


```

例外 filter 还提供逻辑操作符AND和OR的方法，名字叫做and()、or()和xor()，用于将传入 filter() 方法的条件合并起来。
```java
List<String> list1 = Arrays.asList("spring", "summer", "autumn", "winter");
Predicate<String> startWithS = s -> s.startsWith("s");
        Predicate<String> endWithG = g -> g.endsWith("g");
        list1.stream()
                .filter(startWithS.and(endWithG))
                .forEach(System.out::println);
```
打印结果：
```
spring
```

#### 使用lambda表达式的Map和Reduce
给list 中 每个数据 增加 50% 
```java
 List<Integer> list2 = Arrays.asList(100, 200, 300, 400);
        for (Integer num : list2) {
            Double result = num + num * 0.5;
            System.out.println(result);
        }

        list2.stream()
                .map(num -> num + num * 0.5)
                .forEach(System.out::println);
```

打印结果：
```
150.0
300.0
450.0
600.0
150.0
300.0
450.0
600.0
```
计算一个list 每个值加上 50%后的和
```java
 List<Integer> list2 = Arrays.asList(100, 200, 300, 400);
double total = 0;
        for (Integer num : list2) {
            Double result = num + num * 0.5;
            total += result;
            System.out.println(total);
        }

        total = list2.stream()
                .map(num -> num + num * 0.5)
                .reduce((sum, result) -> sum + result).get();
        System.out.println(total);
```
打印结果：
```
1500.0
1500.0
```
map将集合类（例如列表）元素进行转换的。还有一个 reduce() 函数可以将所有值合并成一个。Map和Reduce操作是函数式编程的核心操作，因为其功能，reduce 又被称为折叠操作。

#### 通过过滤创建一个String列表
 通过过滤创建一个新的字符串列表，每个字符串长度大于2
```java
List<String> list3 = Arrays.asList("abc", "def", "hi", "hello");
        // 创建一个字符串列表，每个字符串长度大于2
        List<String> filtered = list3.stream().filter(x -> x.length()> 2).collect(Collectors.toList());
        System.out.printf("Original List : %s, filtered list : %s %n", list3, filtered);
```

#### 对列表的每个元素应用函数
对list3 的每个元素转换成大写，并用逗号连接起来。
```java
List<String> list3 = Arrays.asList("abc", "def", "hi", "hello");
String string = list3.stream().map(s -> s.toUpperCase()).collect(Collectors.joining(","));
        System.out.printf("Original List : %s, After String : %s %n", list3, string);
```
运行结果：
```
Original List : [abc, def, hi, hello], After String : ABC,DEF,HI,HELLO 
```
#### 复制不同的值，创建一个子列表
如何利用流的 distinct() 方法来对集合进行去重。
```java
List<Integer> list4 = Arrays.asList(1, 2, 3, 4, 1);
        List<Integer> distinctList = list4.stream().map( i -> i * i).distinct().collect(Collectors.toList());
        System.out.printf("Original List : %s,  Square Reslut : %s %n", list4, distinctList);
```
运行结果：
```
Original List : [1, 2, 3, 4, 1],  Square Reslut : [1, 4, 9, 16] 
```
#### 计算集合元素的最大值、最小值、总和以及平均值
IntStream、LongStream 和 DoubleStream 等流的类中，有个非常有用的方法叫做 summaryStatistics() 。可以返回 IntSummaryStatistics、LongSummaryStatistics 或者 DoubleSummaryStatistic s，描述流中元素的各种摘要数据。

我们用这个方法来计算列表的最大值和最小值。它也有 getSum() 和 getAverage() 方法来获得列表的所有元素的总和及平均值。
```java
List<Integer> list4 = Arrays.asList(1, 2, 3, 4, 1);
        IntSummaryStatistics stats = list4.stream().mapToInt((x) -> x).summaryStatistics();
        System.out.println("Highest number in List : " + stats.getMax());
        System.out.println("Lowest  number in List : " + stats.getMin());
        System.out.println("Sum of all numbers : " + stats.getSum());
        System.out.println("Average of all numbers : " + stats.getAverage());
```
运行结果：
```
Highest number in List : 4
Lowest  number in List : 1
Sum of all numbers : 11
Average of all numbers : 2.2
```

###  function 包下的函数式接口

JDK 1.8 API包含了很多内建的函数式接口，在老 Java 中常用到的比如 `Comparator`或者 `Runnable`接口，这些接口都增加了 `@FunctionalInterface` 注解以便能用在lambda上。
Java 8 API同样还提供了很多全新的函数式接口来让工作更加方便。

学习例子：

```java
public class Demo {
    private String demoName;

    public Demo() {
    }

    public Demo(String demoName) {
        this.demoName = demoName;
    }

    public String getDemoName() {
        return demoName;
    }

    @Override
    public String toString() {
        return "Demo{" +
                "demoName='" + demoName + '\'' +
                '}';
    }

    public static void main(String[] args) {
        // Predicate 接口只有一个参数，返回boolean类型。
        // 该接口包含多种默认方法来将Predicate组合成其他复杂的逻辑（比如：与，或，非）：
        Predicate<String> predicate = (s) -> s.length() > 0;
        predicate.test("foo");              // true
        predicate.negate().test("foo");     // false
        Predicate<String> isEmpty = String::isEmpty;
        Predicate<String> isNotEmpty = isEmpty.negate();

        //Function 接口
        //Function 接口有一个参数并且返回一个结果，并附带了一些可以和其他函数组合的默认方法（compose, andThen）
        // 类似的 有操作的函数式接口 UnaryOperator ，一元参数和返回类型规则相同
        Function<String, Integer> toInteger = Integer::valueOf;
        Function<Integer, Demo> int2Demo = integer -> new Demo(integer.toString());
        Function<String, Demo> str2Demo = toInteger.andThen(int2Demo);
        System.out.printf("Function toInteger is %d%n", toInteger.apply("123"));
        System.out.printf("Function toString is %s%n", str2Demo.apply("123"));
        // Function backToString is Demo{demoName='123'}

        //Supplier 接口
        //Supplier 接口返回一个任意范型的值，和Function接口不同的是该接口没有任何参数
        Supplier<Demo> supplier = Demo::new;
        // Get a result.
        System.out.println(supplier.get()); // Demo{demoName='null'}

        // Consumer 接口
        // Consumer 接口只是实现操作没有任何返回值
        Consumer<Integer> consumer = System.out::println;
        consumer.accept(2); // 2

        // 基础多参数的 Bi 前缀的函数接口 BiConsumer，BiFunction，BiPredicate……
        // 举个栗子 BiFunction()
        BiFunction<Integer, String, Demo> biFunction = (i, s) -> {
            String demoName = s + i;
            return new Demo(demoName);
        };
        System.out.println(biFunction.apply(1, "KronChan")); //  Demo{demoName='KronChan1'}

        // BinaryOperator 继承 BiFunction，表示二元参数 和 返回类型一样
        BinaryOperator<Integer> integerBinaryOperator = (l, r) -> l + r;
        System.out.println(integerBinaryOperator.apply(4, 5));

        //制作比较规则，输出大的 Demo
        ToIntFunction<Demo> toIntFunction = d -> {
            String value = d.getDemoName();
            return Integer.valueOf(value);
        };
        BinaryOperator<Demo> compareBinaryOperator = BinaryOperator.maxBy(Comparator.comparingInt(toIntFunction));
        System.out.println(compareBinaryOperator.apply(new Demo("1"), new Demo("2")));
        // Demo{demoName='2'}

        // LongBinaryOperator 继承自 BinaryOperator 实现 两个 long 参数方法，返回 long 类型，此外还有 double int……
        LongBinaryOperator longBinaryOperator = (l, r) -> l + r;
        System.out.println(longBinaryOperator.applyAsLong(1, 2)); // 3
    }
}
```



### 总结

1. lambda 表达式只能引用 ``final`` 或 final 局部变量，这就是说不能在 lambda 内部``修改``定义在域外的变量，否则会编译错误。
2. Lambda表达式在Java中又称为闭包或匿名函数，
3. lambda内部可以使用静态、非静态和局部变量，这称为lambda内的变量捕获。
4. 如果在 lambda 表达式 内部不能调用参数方法的引用，需要声明参数类型。
5. 合理使用 Function 函数式接口可以方便的构建符合自己的需求的函数。


