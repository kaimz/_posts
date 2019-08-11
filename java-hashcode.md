---
title: Java 中 hashCode 的一些研究
date: 2018-05-15 22:08:03
tags: [java]
categories: 学习笔记
---

#### 提出问题
1. hashcode 是干什么用的？
2. 为什么要重写 hashcode 和 equals；
3. 怎么去重写 hashcode；

<!--more-->

#### hashcode 是什么

##### hash

> Hash，一般翻译做“散列”，也有直接音译为“哈希”的，就是把任意长度的输入（又叫做预映射， pre-image），通过散列算法，变换成固定长度的输出，该输出就是散列值。

##### Java 中的 hashCode

`hashCode`是 jdk 根据对象的地址算出来的一个 **int** 数字，即对象的哈希码值，代表了该对象在内存中的存储位置。

顶级父类 `Object` 提供获取 hashcode 的方法，调用的是本地的方法；

```java
public native int hashCode();
```

Java 中的 hash 值主要用来干什么的？

hash 值主要是用来在散列存储结构（HashMap、HashTable、HashSet 等等）中确定对象的存储地址的，提高对象的查询效率，

#### 为什么要重写 hashcode 和 equals

首先了解默认情况下的 hashcode 和 equals 方法是什么样：

* hashcode 根据内存地址换算出来一个值；
* equals 判断对象的内存地址是否一样；

但是大多数情况下，我们是需要判断它们的值是否是相等的情况。

`Object.hashCode`的通用约定（*摘自《Effective Java》第45页*）

1. 在一个应用程序执行期间，如果一个对象的equals方法做比较所用到的信息没有被修改的话，那么，对该对象调用hashCode方法多次，它必须始终如一地返回 同一个整数。在同一个应用程序的多次执行过程中，这个整数可以不同，即这个应用程序这次执行返回的整数与下一次执行返回的整数可以不一致。
2. 如果两个对象根据equals(Object)方法是相等的，那么调用这两个对象中任一个对象的hashCode方法必须产生同样的整数结果。
3. 如果两个对象根据equals(Object)方法是不相等的，那么调用这两个对象中任一个对象的hashCode方法，不要求必须产生不同的整数结果。然而，程序员应该意识到这样的事实，对于不相等的对象产生截然不同的整数结果，有可能提高散列表（hash table）的性能。

**如果只重写了equals方法而没有重写hashCode方法的话，则会违反约定的第二条：相等的对象必须具有相等的散列码（hashCode）**。

比如，我们经常用 String 类型作为 HashMap 的键，知道 HashMap 的键值存储的方式根据元素的 hashcode 求模来判断将元素放到哪个位置的。我们知道 hashMap 的特点是键值不能重复，这个重复就是先根据 hashcode 值判断在哪个位置，然后去链表中 用 equals 判断有没有相同。 如果不重写 hashcode 会造成一些意外的事件。

又引出例外一个问题了，比如我们用一个可变的对象作为 hashMap 的键值，并且重写了 hashcode 和 equals 方法，当我把一对键值（可变对象为键）装进 hashMap 后，又去改变了对象的某个属性（这个属性参与了 hashcode 的计算），然后就不能再用这个可变对象去操作已经插入到 hashMap 中的键值对了。

#### 怎么去重写 hashcode

从阅读 `String` 源码来分析比较简单：

```java
    public int hashCode() {
        int h = hash;// 主要是 String 对象是不可变的，可以使用一个变量存储起来，方便以后使用。
        if (h == 0 && value.length > 0) {
            char val[] = value;
// 计算每个字符的 ascii 参与到 hashcode 计算中，将前面计算的结果乘以 31 。
            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
```

提出问题，为什么要以 *31* 为权来计算 hashCode？

1. 因为 31 是素数，素数跟其他数相乘，更容易产生唯一性，所以 hash 冲突会小；

2. 相乘的时候，数字太大，结果也是越大，很容易造成超出 int 值上限，导致数据丢失的情况，那为什么不是 17 了，参考StackOverflow上最高票的答案[参考答案](https://stackoverflow.com/questions/299304/why-does-javas-hashcode-in-string-use-31-as-a-multiplier)

   > The value 31 was chosen because it is an odd prime. If it were even and the multiplication overflowed, information would be lost, as multiplication by 2 is equivalent to shifting. The advantage of using a prime is less clear, but it is traditional. A nice property of 31 is that the multiplication can be replaced by a shift and a subtraction for better performance: `31 * i == (i << 5) - i`. Modern VMs do this sort of optimization automatically.

   解释说，因为虚拟机已经对**移位和减法**进行了优化，并且代替了乘法，性能会更好，因此 hash 的计算表达式是 ：`31 * i == (i << 5) - i`。

另外非常有意思的是还可以查看 `Long.java` 的 `hashCode()` 方法，

```java
    public static int hashCode(long value) {
        return (int)(value ^ (value >>> 32));
    }
```

因为 Long 类型有 64 位，比 hash 的长度多了一倍，利用前 32 位 和后 32 位异或，尽可能的让更多的位置参与计算 hash 来保证唯一性。

最后可以参考 `IDEA` 默认自动生成的 hashCode 和 equals 方法：

```java
public class ObjectDemo {
    private String value1;
    private String value2;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true; 
        if (o == null || getClass() != o.getClass()) return false;

        ObjectDemo that = (ObjectDemo) o;

        if (value1 != null ? !value1.equals(that.value1) : that.value1 != null) return false; 
        return value2 != null ? value2.equals(that.value2) : that.value2 == null;
    }

    @Override
    public int hashCode() {
        int result = value1 != null ? value1.hashCode() : 0;
        result = 31 * result + (value2 != null ? value2.hashCode() : 0);
        return result;
    }
}
```

#### 参考文章

* [Why does Java's hashCode() in String use 31 as a multiplier?](https://stackoverflow.com/questions/299304/why-does-javas-hashcode-in-string-use-31-as-a-multiplier)

* [java为什么要重写hashCode和equals方法 ](https://blog.csdn.net/zknxx/article/details/53862572)

