---
title: 初探Java 9 的的模块化
date: 2017-12-29 17:28:03
tags: [java]
categories: 学习笔记
---
Java 9中最重要的功能，毫无疑问就是模块化（Module），它将自己长期依赖JRE的结构，转变成以Module为基础的组件，当然这在使用Java 9 开发也和以前有着很大的不同。

<!--more-->

### Java8或更加早期的系统的问题
1. Jar文件，像rt.jar等jar文件太大的以至于不能使用在小设备和应用中。
2. 因为JDK是太大的，我们的应用或设备不能支持更好的平台.
3. 由于修饰符是public的缘故，每个人都可以通过此来进行访问，所以在当前Java系统的封闭性不是很强。
4. 由于JDK,Jre过于庞大，以至于很难进行测试和维护应用。
5. 由于public的关系，Java比较开放。不可避免的能访问象sun.， .internal.*等的一些内部不重要的APIs。

### Java9模块系统的特性 
1. 在Java SE 9中分离了JDK， JRE，jar等为更小的模。因此我们可以方便的使用任何我们想要的模块。因此缩减Java应用程序到小设备是非常容易的。
2. 更加容易的测试和维护。
3. 支持更好的平台。
4. public不再仅仅是public。现在已经支持非常强的封闭性(不用担心，后边我们会用几个例子来解释)。
5. 我们不能再访问内部非关键性APIs了。
6. 模块可以非常安全地帮助我们掩藏那些我们不想暴露的内部细节，我们可以得到更好的Security。
7. 应用会变的非常小，因为我们可以只使用我们要用的模块。
8. 组件间的松耦合变得非常容易。
9. 更容易支持唯一责任原则(SRP)。


### 目录结构
以前的jre中有一个很大的架包，jdk8 中rt.jar有62M，即便运行一个最简单的HelloWorld，都必须带上它。

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmxpyk5k96j20fo0gt40g.jpg)

jdk9 的目录
![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmxq22lqipj20fh06i0t1.jpg)

我们发现，jdk9 中没有Jre 文件，也没有rt.jar等这种很大的架包，但是它有了一个新的文件jmods，模块都是放在jmods文件夹中。

*目前共有98个模块。*

### Module的相关属性
在主目录的/main/java/下新建``module-info.java``文件，可以管理这个项目的module。
```java
module M {

}
```
#### 模块命名
又称模块描述文件
模块命名需要保证单一，可以使用反向域名模式，如``com.wuwii.xxx.xxx``，这个模块会导出包``com.wuwii``。

在JDK 9中， open, module, requires, transitive, exports, opens, to, uses, provides 和 with是受限关键字。只有当具体位置出现在模块声明中时，它们才具有特殊意义。 可以将它们用作程序中其他地方的标识符。 

例如：可以在程序中声明一个module变量。

#### 访问权限

模块之间的关系被称作readability（可读性），代表一个模块是否可以找到这个模块文件，并且读入系统中（注意：并非代表可以访问其中的类型）。在实际的代码，一个类型对于另外一个类型的调用，我们称之为可访问性(Accessible)，这意味着可以使用这个类型； 可访问性的前提是可读性，换句话说，现有模块可读，然后再进一步检测可访问性（安全）。
导出语句将模块的指定包导出到所有模块或编译时和运行时的命名模块列表。 它的两种形式如下：
```java
exports <package>;
exports <package> to <module1>, <module2>...;
```
以下是使用了导出语句的模块示例：
```java
module java.xml.ws {
……
    exports com.oracle.webservices.internal.api.databinding to
        jdk.xml.ws;
    exports com.sun.xml.internal.ws.addressing to
        jdk.xml.ws,
        java.xml.bind;
……
}
```
开放语句允许对所有模块的反射访问指定的包或运行时指定的模块列表。 其他模块可以使用反射访问指定包中的所有类型以及这些类型的所有成员（私有和公共）。 开放语句采用以下形式：
```java
opens <package>;
opens <package> to <module1>, <module2>...;
```
>Tips
对比导出和打开语句。 导出语句允许仅在编译时和运行时访问指定包的公共API，而打开语句允许在运行时使用反射访问指定包中的所有类型的公共和私有成员。

```java
module N {
    exports M;
    opens M;
}
```
阅读有关模块的时候会遇到三个短语：

* 模块M导出包P
* 模块M打开包Q
* 模块M包含包R

前两个短语对应于模块中导出语句和开放语句。 第三个短语意味着该模块包含的包R既不导出也不开放。 在模块系统的早期设计中，第三种情况被称为“模块M隐藏包R”。
#### 声明依赖关系
需要（require）语句声明当前模块与另一个模块的依赖关系。 一个名为M的模块中的“需要N”语句表示模块M取决于（或读取）模块N。语句有以下形式：
```java
requires <module>;
requires transitive <module>;
requires static <module>;
requires transitive static <module>;
```

* require语句中的静态修饰符表示在编译时的依赖是强制的，但在运行时是可选的。
* requires static N语句意味着模块M取决于模块N，模块N必须在编译时出现才能编译模块M，而在运行时存在模块N是可选的。
* require语句中的transitive修饰符会导致依赖于当前模块的其他模块具有隐式依赖性。
* 假设有三个模块P，Q和R，假设模块Q包含requires transitive R语句，如果如果模块P包含包含requires Q语句，这意味着模块P隐含地取决于模块R。

#### 配置服务
Java允许使用服务提供者和服务使用者分离的服务提供者机制。 JDK 9允许使用语句（uses statement）和提供语句（provides statement）实现其服务。

使用语句可以指定服务接口的名字，当前模块就会发现它，使用 java.util.ServiceLoader类进行加载。格式如下：
```java
uses <service-interface>;
```
使用语句的实例如下：
```java
module M {
    uses com.jdojo.prime.PrimeChecker;
}
```
com.jdojo.PrimeChecker是一个服务接口，其实现类将由其他模块提供。 模块M将使用java.util.ServiceLoader类来发现和加载此接口的实现。

提供语句指定服务接口的一个或多个服务提供程序实现类。 它采取以下形式：
```java
provides <service-interface>
    with <service-impl-class1>, <service-impl-class2>...;
```
相同的模块可以提供服务实现，可以发现和加载服务。 模块还可以发现和加载一种服务，并为另一种服务提供实现。 以下是例子：
```java
module P {
    uses com.jdojo.CsvParser;
    provides com.jdojo.CsvParser
        with com.jdojo.CsvParserImpl;
    provides com.jdojo.prime.PrimeChecker
        with com.jdojo.prime.generic.FasterPrimeChecker;
}
```

### 总结

需要注意的是，不只是jdk中内置的98种模块，引用maven的第三方架包，也需要module，
如用的比较多的日志
```
    /**
     * logger
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(LearnSoap.class);
```
配置
需要在``module-info.java``配置：
```
requires slf4j.api;
```

Java中的包已被用作类型的容器。 应用程序由放置在类路径上的几个JAR组成。 软件包作为类型的容器，不强制执行任何可访问性边界。 类型的可访问性内置在使用修饰符的类型声明中。 如果包中包含内部实现，则无法阻止程序的其他部分访问内部实现。 类路径机制在使用类型时线性搜索类型。 这导致在部署的JAR中缺少类型时，在运行时接收错误的另一个问题 —— 有时在部署应用程序后很长时间。 这些问题可以分为两种类型：封装和配置。

JDK 9引入了模块系统。 它提供了一种组织Java程序的方法。 它有两个主要目标：强大的封装和可靠的配置。 使用模块系统，应用程序由模块组成，这些模块被命名为代码和数据的集合。 模块通过其声明来控制模块的其他模块可以访问的部分。 访问另一个模块的部分的模块必须声明对第二个模块的依赖。 控制访问和声明依赖的是达成强封装的基础。 在应用程序启动时解决了一个模块的依赖关系。 在JDK 9中，如果一个模块依赖于另一个模块，并且运行应用程序时第二个模块丢失，则在启动时将会收到一个错误，而不是应用程序运行后的某个时间。 这是一个可靠的基础配置。

使用模块声明定义模块。 模块的源代码通常存储在名为module-info.java的文件中。 一个模块被编译成一个类文件，通常命名为module-info.class。 编译后的模块声明称为模块描述符。 模块声明不允许指定模块版本。 但诸如将模块打包到JAR中的jar工具的可以将模块版本添加到模块描述符中。

使用module关键字声明模块，后跟模块名称。 模块声明可以使用五种类型的模块语句：exports，opens，require，uses和provide。 导出语句将模块的指定包导出到所有模块或编译时和运行时的命名模块列表。 开放语句允许对所有模块的反射访问指定的包或运行时指定的模块列表， 其他模块可以使用反射访问指定包中的所有类型以及这些类型的所有成员（私有和公共）。 使用语句和提供模块语句用于配置模块以发现服务实现并提供特定服务接口的服务实现。

从JDK 9开始，open， module， requires， transitive, exports，opens，to，uses，provides和with都是受限关键字。 只有当具体位置出现在模块声明中时，它们才具有特殊意义。

### 参考文章
* <a rel="external nofollow" target="_blank" href="http://www.cnblogs.com/IcanFixIt/p/6947763.html">Java 9 揭秘（2. 模块化系统）</a>
* <a rel="external nofollow" target="_blank" href="https://www.jianshu.com/p/053a5ca89bbb">Java 9 新特性来临——模块化</a>

