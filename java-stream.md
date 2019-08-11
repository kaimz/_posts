---
title: Java8 新特性Stream 的学习和使用方法
date: 2017-10-25 22:18:03
tags: [java]
categories: 学习笔记
---

### 流（Stream）
流是java 8 中新引入的特性，用来处理集合中的数据，Stream 是一个来自数据源的元素队列并支持聚合操作。
* Java 中 Stream 不会存储元素。
* 数据源 流的来源。 可以是集合，数组，I/O channel， 产生器generator 等。
* 聚合操作 类似SQL语句一样的操作， 比如filter, map, reduce, find, match, sorted等。

<!--more-->

 Stream操作还有几个特征：
* **只遍历一次**。我们可以把流想象成一条流水线，流水线的源头是我们的数据源(一个集合)，数据源中的元素依次被输送到流水线上，我们可以在流水线上对元素进行各种操作。一旦元素走到了流水线的另一头，那么这些元素就被“消费掉了”，我们无法再对这个流进行操作。当然，我们可以从数据源那里再获得一个新的流重新遍历一遍。
* **Pipelining**: 中间操作都会返回流对象本身。 这样多个操作可以串联成一个管道， 如同流式风格（fluent style）。 这样做可以对操作进行优化， 比如延迟执行(laziness)和短路( short-circuiting)。
* **内部迭代**： 以前对集合遍历都是通过Iterator或者For-Each的方式, 显式的在集合外部进行迭代， 这叫做外部迭代。 Stream提供了内部迭代的方式， 通过访问者模式(Visitor)实现。

### 为什么需要 Stream

> Stream 作为 Java 8 的一大亮点，它与 java.io 包里的 InputStream 和 OutputStream 是完全不同的概念。它也不同于 StAX 对 XML 解析的 Stream，也不是 Amazon Kinesis 对大数据实时处理的 Stream。Java 8 中的 Stream 是对集合（Collection）对象功能的增强，它专注于对集合对象进行各种非常便利、高效的聚合操作（aggregate operation），或者大批量数据操作 (bulk data operation)。Stream API 借助于同样新出现的 Lambda 表达式，极大的提高编程效率和程序可读性。同时它提供串行和并行两种模式进行汇聚操作，并发模式能够充分利用多核处理器的优势，使用 fork/join 并行方式来拆分任务和加速处理过程。通常编写并行代码很难而且容易出错, 但使用 Stream API 无需编写一行多线程的代码，就可以很方便地写出高性能的并发程序。所以说，Java 8 中首次出现的 java.util.stream 是一个函数式语言+多核时代综合影响的产物。
>
> [Java 8 中的 Streams API 详解](https://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/index.html)

### 流的使用
流的使用过程有三步：
* 获取流；
* 中间操作，得到一个新的流；
* 最终操作，获取结果。

#### 获取流
流有两种：
* stream() ： 创建串行流。
* parallelStream()  ： 创建并行流。

并行流的特点就是将一个大任务切分成多个小任务，无序一起执行，当然如果我们需要顺序输出的话可以使用``forEachOrdered``，速度会比串行流快一些。它通过默认的ForkJoinPool,可能提高你的多线程任务的速度。

##### 从集合获取流

```java
List<FarmEntity> list = service.getBySql(sql1);
Stream<FarmEntity> stream = list.stream();
```
##### 从数组获取流

```java
String[] arrays = {"你", "我", "她"};
Stream<String> stream = Arrays.stream(arrays);
```

##### 从值获取流
```java
Stream<String> stream = Stream.of("你", "我", "她");
```
##### 从文件获取流
```java
try {
    Stream<String> file =Files.lines(Paths.get("D:\\zhangkai\\WorkSpace\\Git\\hexo\\_config.yml"));
    file.forEach(System.out::println);
} catch (Exception e) {

}
```
使用NIO获取流，可以打印出文本文件的内容。
#### 流的操作
##### filter 过滤
filter函数接收一个Lambda表达式作为参数，该表达式返回boolean，在执行过程中，流将元素逐一输送给filter，并筛选出执行结果为true的元素。
```java
String[] strings = {"珊瑚", "阳光", "细腻", "冷暖", "阳光"};
Arrays.stream(strings).filter(n -> n.startsWith("冷")).forEach(System.out::print);
```

##### distinct 去重

```java
Arrays.stream(strings).distinct().forEach(System.out::print);
```


##### limit 截取
截取前面两个单位：
```java
Arrays.stream(strings).limit(2).forEach(System.out::print);
```


##### skip 跳过
和上面的limit 相反，跳过前面两个
```java

```


##### map 映射 
map 方法用于映射每个元素到对应的结果。
给每个词语后面加个 “兮”
```java
        Arrays.stream(strings).map(s -> s + "兮").forEach(System.out::println);
```
输出：
```
珊瑚兮
阳光兮
细腻兮
冷暖兮
阳光兮
```


##### sorted 排序 

```java
//Arrays.stream(strings).sorted((x, y) -> x.compareTo(y)).forEach(System.out::println);
Arrays.stream(strings).sorted(String::compareTo).forEach(System.out::println);
```
输出：
```
冷暖
珊瑚
细腻
阳光
阳光
```

java8 以前排序：
```java
 // Before Java 8 sorted
        System.out.println("java8以前排序：");
        List<String> list1 = Arrays.asList(strings);
        list1.sort(new Comparator<String>() {
            @Override
            public int compare(String o1, String o2) {
                return o1.compareTo(o2);
            }
        });
        System.out.printf("java8 以前的排序：%s%n", list1);
```
输出：
```
java8以前排序：
java8 以前的排序：[冷暖, 珊瑚, 细腻, 阳光, 阳光]
```

HashMap根据value值排序key：
```java
Map<String, Integer> map = new HashMap<>();
map.put("spring", 1);
map.put("summer", 2);
map.put("autumn", 3);
map.put("winter", 4);
map.entrySet().stream()
    .sorted((a, b) -> b.getValue().compareTo(a.getValue()))
    .forEach(a -> System.out.println(a.getKey()));
```
输出结果：
```java
winter
autumn
summer
spring
```

##### 统计

```java
//统计
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

##### match 匹配

* ``anyMatch``用于判断流中是否存在至少一个元素满足指定的条件，这个判断条件通过Lambda表达式传递给anyMatch，执行结果为``boolean``类型。 
* ``noneMatch``与allMatch恰恰相反，它用于判断流中的所有元素是否都不满足指定条件
* ``findAny``能够从流中随便选一个元素出来，它返回一个Optional类型的元素。
```java
 Boolean result1 = Arrays.stream(strings).allMatch(s -> s.equals("java"));
        System.out.println(result1);

        Boolean reslut2 = Arrays.stream(strings).noneMatch(s -> s.equals("java"));
        System.out.println(reslut2);
        //随机读取一个
        Optional<String> getResult = Arrays.stream(strings).findAny();
        System.out.println(getResult);
        System.out.printf("获取Optional中的值：%s%n", getResult.get());
```
运行结果：
```
false
true
Optional[冷暖]
获取Optional中的值：冷暖
```
>Optional是Java8新加入的一个容器，这个容器只存1个或0个元素，它用于防止出现NullpointException，它提供如下方法：
* ``isPresent()`` 判断容器中是否有值。
* ``ifPresent(Consume lambda) `` 容器若不为空则执行括号中的Lambda表达式。
* ``T get()``  获取容器中的元素，若容器为空则抛出NoSuchElement异常。
* ``T orElse(T other)``  获取容器中的元素，若容器为空则返回括号中的默认值。

##### reduce 归约
求和：
```java
//归约
        //第一种方法求和
        String connectStrings = Arrays.stream(strings).reduce("", (x, y) -> x + y);
        System.out.println(connectStrings);

        // 第二种方法求和
        String connectStrings1 = Arrays.stream(strings).reduce("", TestStream::getConnectStrings);
        System.out.println(connectStrings1);
```

getConnectStrings方法：
```java
/**
* Connect Strings
* @param s1 参数1
* @param s2 参数2
* @return java.lang.String
*/
private static String getConnectStrings(String s1, String s2) {
    return s1 + s2;
}
```
``reduce``中第一个参数是初始值，第二个参数是方法引用。

##### 数据流 
StreamAPI提供了三种数值流：IntStream、DoubleStream、LongStream，也提供了将普通流转换成数值流的三种方法：mapToInt、mapToDouble、mapToLong。 

每种数值流都提供了数值计算函数，如max、min、sum等。

下面使用 mapToInt 为例：

```java
        String[] numberStrings = {"1", "2", "3"};
        // mapToInt参数： 需要转换成相应的类型方法
        IntStream intStream = Arrays.stream(numberStrings).mapToInt(Integer::valueOf);
        //使用对应的 Optional 接收
        OptionalInt optionalNumber = intStream.max();
        // 取值，给默认值 0，为空结果为0
        System.out.printf("numberStrings's max number is: %s%n", optionalNumber.orElse(0));

```
打印结果：
```
numberStrings's max number is: 3
```

由于数值流可能为空，并且给空的数值流计算最大值是没有意义的，因此max函数返回OptionalInt，它是Optional的一个子类，能够判断流是否为空，并对流为空的情况作相应的处理。 所以可以直接使用 ``OptionalInt.getAsInt()``获取容器的值。  
为空的话捕捉异常：
```
java.util.NoSuchElementException: No value present
	at java.util.OptionalInt.getAsInt(OptionalInt.java:118)
	at com.wuwii.test.TestStream.main(TestStream.java:105)
```

此外，mapToInt、mapToDouble、mapToLong进行数值操作后的返回结果分别为：OptionalInt、OptionalDouble、OptionalLong。


##### Collectors 集合归约
将流转换成集合和聚合元素。
```java
 //Collectors 集合归约
        //  toList
        List<String> list2 = Arrays.stream(strings).collect(Collectors.toList());
        // Get String by connected
        String connectStrings2 = Arrays.stream(strings).collect(Collectors.joining(","));
        System.out.printf("Collectors toList: %s , Conlletors Join Strings: %s%n", list2, connectStrings2);
```
打印结果：
```
Collectors toList: [冷暖, 珊瑚, 细腻, 阳光, 阳光] , Conlletors Join Strings: 冷暖,珊瑚,细腻,阳光,阳光
```
后面补充: Collectors中还有一个groupingBy() 方法，比较实用，例子来源网上<a rel="external nofollow" target="_blank" href="https://nkcoder.github.io/2016/01/24/java-8-stream-api/">使用Java 8中的Stream</a>
1. **``groupingBy()``表示根据某一个字段或条件进行分组，返回一个``Map``，其中key为分组的字段或条件，value默认为list，``groupingByConcurrent()``是其并发版本**：
```java
Map<String, List<Locale>> countryToLocaleList = Stream.of(Locale.getAvailableLocales())
    .collect(Collectors.groupingBy(l -> l.getDisplayCountry()));
```
2. **如果``groupingBy()``分组的依据是一个bool条件，则key的值为true/false，此时与partitioningBy()等价，且``partitioningBy()``的效率更高**：
```java
// predicate
Map<Boolean, List<Locale>> englishAndOtherLocales = Stream.of(Locale.getAvailableLocales())
    .collect(Collectors.groupingBy(l -> l.getDisplayLanguage().equalsIgnoreCase("English")));

// partitioningBy
Map<Boolean, List<Locale>> englishAndOtherLocales2 = Stream.of(Locale.getAvailableLocales())
    .collect(Collectors.partitioningBy(l -> l.getDisplayLanguage().equalsIgnoreCase("English")));
```

3. **``groupingBy()``提供第二个参数，表示downstream，即对分组后的value作进一步的处理**：

```java
// 返回set，而不是list：
Map<String, Set<Locale>> countryToLocaleSet = Stream.of(Locale.getAvailableLocales())
          .collect(Collectors.groupingBy(l -> l.getDisplayCountry(), Collectors.toSet()));

// 返回value集合中元素的数量：
Map<String, Long> countryToLocaleCounts = Stream.of(Locale.getAvailableLocales())
          .collect(Collectors.groupingBy(l -> l.getDisplayCountry(), Collectors.counting()));

// 对value集合中的元素求和：
Map<String, Integer> cityToPopulationSum = Stream.of(cities)
        .collect(Collectors.groupingBy(City::getName, Collectors.summingInt(City::getPopulation)));
		
// 对value的某一个字段求最大值，注意value是Optional的：
Map<String, Optional<City>> cityToPopulationMax = Stream.of(cities)
        .collect(Collectors.groupingBy(City::getName,
            Collectors.maxBy(Comparator.comparing(City::getPopulation))));
			

// 使用mapping对value的字段进行map处理：
Map<String, Optional<String>> stateToNameMax = Stream.of(cities)
    .collect(Collectors.groupingBy(City::getState, Collectors.mapping(City::getName,
        Collectors.maxBy(Comparator.comparing(String::length)))));

Map<String, Set<String>> stateToNameSet = Stream.of(cities)
.collect(Collectors.groupingBy(City::getState,
    Collectors.mapping(City::getName, Collectors.toSet())));
	
// 通过summarizingXXX获取统计结果：
Map<String, IntSummaryStatistics> stateToPopulationSummary = Stream.of(cities)
    .collect(Collectors.groupingBy(City::getState, Collectors.summarizingInt(City::getPopulation)));
reducing()

// 可以对结果作更复杂的处理，但是reducing()却并不常用：
Map<String, String> stateToNameJoining = Stream.of(cities)
    .collect(Collectors.groupingBy(City::getState, Collectors.reducing("", City::getName,
        (s, t) -> s.length() == 0 ? t : s + ", " + t)));

// 比如上例可以通过mapping达到同样的效果：
Map<String, String> stateToNameJoining2 = Stream.of(cities)
        .collect(Collectors.groupingBy(City::getState,
            Collectors.mapping(City::getName, Collectors.joining(", ")
        )));
```


### 完整代码
```java
package com.wuwii.test;

import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.IntStream;
import java.util.stream.Stream;

/**
 * Learn Java 8 Stream
 *
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/10/25 22:16</pre>
 */
public class TestStream {

    public static void main(String[] args) {
        // Get Stream from file
        System.out.println("读取文件：");
        try {
            Stream<String> file = Files.lines(Paths.get("D:\\zhangkai\\WorkSpace\\Git\\hexo\\_config.yml"));
            file.forEach(System.out::println);
        } catch (Exception e) {

        }

        // Get Stream by Filter
        String[] strings = {"珊瑚", "阳光", "细腻", "冷暖", "阳光"};
        Arrays.stream(strings).filter(n -> n.startsWith("冷")).forEach(System.out::print);

        // Get Stream by Distinct
        System.out.println("去重:");
        Arrays.stream(strings).distinct().forEach(System.out::print);

        // Get Stream by Limit
        System.out.println("截取:");
        Arrays.stream(strings).limit(2).forEach(System.out::print);

        // Get Stream by Skip
        System.out.println("跳过:");
        Arrays.stream(strings).skip(2).forEach(System.out::print);

        // Java 8 sorted
        System.out.println("排序：");
        //Arrays.stream(strings).sorted((x, y) -> x.compareTo(y)).forEach(System.out::println);
		Arrays.stream(strings).sorted(String::compareTo).forEach(System.out::println);

        // Before Java 8 sorted
        System.out.println("java8以前排序：");
        List<String> list1 = Arrays.asList(strings);
        list1.sort(new Comparator<String>() {
            @Override
            public int compare(String o1, String o2) {
                return o1.compareTo(o2);
            }
        });
        System.out.printf("java8 以前的排序：%s%n", list1);

        //Handle map
        System.out.println("map 映射：");
        Arrays.stream(strings).map(s -> s + "兮").forEach(System.out::println);

        //Match
        Boolean result1 = Arrays.stream(strings).allMatch(s -> s.equals("java"));
        System.out.println(result1);

        Boolean reslut2 = Arrays.stream(strings).noneMatch(s -> s.equals("java"));
        System.out.println(reslut2);
        //findAny to find anyone
        Optional<String> getResult = Arrays.stream(strings).findAny();
        System.out.println(getResult);
        System.out.printf("获取Optional中的值：%s%n", getResult.get());

        //统计
        List<Integer> list4 = Arrays.asList(1, 2, 3, 4, 1);
        IntSummaryStatistics stats = list4.stream().mapToInt((x) -> x).summaryStatistics();
        System.out.println("Highest number in List : " + stats.getMax());
        System.out.println("Lowest  number in List : " + stats.getMin());
        System.out.println("Sum of all numbers : " + stats.getSum());
        System.out.println("Average of all numbers : " + stats.getAverage());

        //归约
        //第一种方法求和
        String connectStrings = Arrays.stream(strings).reduce("", (x, y) -> x + y);
        System.out.println(connectStrings);

        // 第二种方法求和
        String connectStrings1 = Arrays.stream(strings).reduce("", TestStream::getConnectStrings);
        System.out.println(connectStrings1);

        //Collectors 集合归约
        //  toList
        List<String> list2 = Arrays.stream(strings).collect(Collectors.toList());
        // Get String by connected
        String connectStrings2 = Arrays.stream(strings).collect(Collectors.joining(","));
        System.out.printf("Collectors toList: %s , Conlletors Join Strings: %s%n", list2, connectStrings2);

        String[] numberStrings = {"1", "2", "3"};
        // mapToInt参数： 需要转换成相应的类型方法
        IntStream intStream = Arrays.stream(numberStrings).mapToInt(Integer::valueOf);
        //使用对应的 Optional 接收
        OptionalInt optionalNumber = intStream.max();
        // 取值，给默认值 0，为空结果为0
        System.out.printf("numberStrings's max number is: %s%n", optionalNumber.orElse(0));
    }

    /**
     * 拼接字符串
     *
     * @param s1 参数1
     * @param s2 参数2
     * @return java.lang.String
     */
    private static String getConnectStrings(String s1, String s2) {
        return s1 + s2;
    }

}
```
