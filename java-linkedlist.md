---
title: Java 8 中 LinkedList 源码阅读记录
date: 2018-5-11 22:08:03
tags: [java]
categories: 学习笔记
---

#### 思考

1. 认识LinkedList 数据结构；
2. 主要认识插入元素是怎么实现的；
3. 遍历 LinkedList 的方法，以及具体实现；
4. 与 ArrayList 对比。

#### 版本

* jdk1.8.0_161

<!--more-->

#### 结构

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

* 继承于`AbstractSequentialList`的双向链表。它也可以被当作堆栈、队列或双端队列进行操作。
* 实现 `List` 接口，能对它进行队列操作。
* 实现 `Deque` 接口，即能将LinkedList当作双端队列使用。
* 实现了`Cloneable`接口，即覆盖了函数`clone()`，能克隆。
* 实现`java.io.Serializable`接口，这意味着LinkedList支持序列化，能通过序列化去传输。
* 是非同步的，非线程安全。

#### 属性

```java
// 链表中有多少个节点，默认为 0    
transient int size = 0;
// 头节点
transient Node<E> first;
// 尾节点
transient Node<E> last;

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

**LinkedList 是一个双向链表**。内部类 `Node` 是 LinkedList 中的基本数据结构，包含当前节点值，上一个节点得引用，和下个节点的引用。

#### 构造方法

比较简单，默认无参构造，和一个 `Collection` 参数的构造（ *将里面元素按顺序前后连接，修改节点个数，并且操作次数 + 1* ）。

#### ADD

##### add(E e)

```java
    public boolean add(E e) {
        // 去为节点加
        linkLast(e);
        return true;
    }

    // 将指定的元素防止在链表的尾节点，以前的尾节点变成它前面的节点，如果上个尾节点为null，说明以前是的空链表。
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```

##### add(int index, E element)

```java
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

    /**
     * Returns the (non-null) Node at the specified element index.
     */
    Node<E> node(int index) {
        // assert isElementIndex(index);
		// 双链表可以分别从 头节点 或者尾节点开始遍历，计算它是在前面一半，还是在后面的位置，决定遍历方式。
		// 这也是LinkedList 为什么要使用双向链表，提升了使用游标操作链表的效率。
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }

    /**
     * Inserts element e before non-null Node succ.
     */
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```

1. 检查索引是否越界，虽然 ListedList 中没有索引概念；
2. 如果 index 和 size 相同，则在尾节点上加上元素； 
3. 不相同的话，先去遍历链表查找到索引位置的节点，然后在它的前面插入节点。

#### Get

##### get(int index)

```java
public E get(int index) {  
    checkElementIndex(index);  
    return node(index).item;  
}  
```

1. 检查索引越界；
2. 跟上面的一样，查找该索引位置的节点，然后获取它的元素。

##### 其他

其他的例如 `getFirst()`、`getLast()`。

#### Remove

##### remove()

```java
    public E remove() {
        return removeFirst();
    }

    // 移除头节点
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }
    // 参数 f 为头节点
	// 将头节点指向 next 节点，如果 next节点 为 null 则链表 为 null ，链表大小减 1 ，修改次数记录加 1.
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
```

##### remove（int index）

```java
public E remove(int index) {  
    checkElementIndex(index);  
    return unlink(node(index));  
}  

    /**
     * Unlinks non-null node x.
     */
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;
        // 如果本节点为头节点，头节点指向next
        if (prev == null) {
            first = next;
        } else {
            // 不是头节点，则将前节点和后节点连接起来，然后删掉本节点的引用 GC
            prev.next = next;
            x.prev = null;
        }
        // 如果是尾节点，则将尾节点指向前节点
        if (next == null) {
            last = prev;
        } else {
            // 连接，双向链表，双方都有引用，删除自身的引用GC
            next.prev = prev;
            x.next = null;
        }
        // 删除自身 GC
        x.item = null;
        size--;
        modCount++;
        return element;
    }

```

##### remove(Object o)

```java
// 遍历 equals 找出 node，然后调用 unlink(Node<E> x)   
public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }
```

#### Set

```java
    public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }
```

1. 有索引，第一件事去检查索引是否越界；
2. 根据索引找出 node；
3. 替换 node 的元素，返回 该索引位置 Node 的旧元素的值。
4. 注意，Set 方法不增加LinkedList 的修改次数

#### clear()

```java
public void clear() {  
       // Clearing all of the links between nodes is "unnecessary", but:  
       // - helps a generational GC if the discarded nodes inhabit  
       //   more than one generation  
       // - is sure to free memory even if there is a reachable Iterator  
       for (Node<E> x = first; x != null; ) {  
           Node<E> next = x.next;  
           x.item = null;  
           x.next = null;  
           x.prev = null;  
           x = next;  
       }  
       first = last = null;  
       size = 0;  
       modCount++;  
   }  
```

1. 释放所有的元素，让他们直接无引用，垃圾回收器发现这些 node 元素是不可达的时候，释放内存。
2. 数据恢复默认；修改次数记录加一。

#### 实现 ListIterator

非线程安全，并发下会快速失败。

继承 `Iterator`，`Iterator` 只能向后遍历和删除操作，`ListIterator` 额外添加了几个方法，主要实现以下几个功能：

1. 允许我们向前（previous）、向后（next）两个方向遍历 List;
2. 使用迭代器遍历的时候，需要时刻知道游标索引的位置；
3. 使用迭代器的时候，需要修改游标处的值，`remove`、`set`、`add`   这几个方法。

```java
    private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned;// 记录当前游标的节点
        private Node<E> next;// 记录下一个节点，向前遍历的时候，一直和 lastReturned 相同
        private int nextIndex;// 记录当前游标
        private int expectedModCount = modCount;//记录修改记录，用于快速失败
        
        private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned; 
        private Node<E> next;
        private int nextIndex; 
        private int expectedModCount = modCount;
```

#### Spliterator

jdk 1.8 中新增分割的迭代器，主要是 Jdk 1.8 中增加的并行处理的流运算，用到了可分割的迭代器。

LinkedList 主要只是实现默认的方法，进行分割，

```java
    static final class LLSpliterator<E> implements Spliterator<E> {
        static final int BATCH_UNIT = 1 << 10;  // batch array size increment
        static final int MAX_BATCH = 1 << 25;  // max batch array size;
        final LinkedList<E> list; // null OK unless traversed
        Node<E> current;      // current node; null until initialized
        int est;              // size estimate; -1 until first needed
        int expectedModCount; // initialized when est set
        int batch;            // batch size for splits
```



1. 使用` trySplit()` 对等分割，返回索引小的那部分 `Spliterator`;

   ```java
           public Spliterator<E> trySplit() {
               Node<E> p;
               int s = getEst();
               // 还能继续分割
               if (s > 1 && (p = current) != null) {
                   // 头节点不为 null
                   // batch 默认为0 ，
                   // BATCH_UNIT 默认为 1024
                   // 所以每一个后面的批处理，都会比前面多
                   int n = batch + BATCH_UNIT;
                   if (n > s)
                       n = s;
                   // 太大了执行最大批量处理
                   if (n > MAX_BATCH)// 33554432
                       n = MAX_BATCH;
                   Object[] a = new Object[n];
                   int j = 0;
                   // 将 list 的元素复制到数组中
                   do { a[j++] = p.item; } while ((p = p.next) != null && j < n);
                   // 如果初始的 list 大小大于 1024 ，将记录这些值，用于下次切割的时候使用
                   current = p;
                   batch = j;
                   est = s - j;
                   return Spliterators.spliterator(a, 0, j, Spliterator.ORDERED);
               }
               return null;
           }
           // 进来的时候 est = -1 ，表示没有分割
           // 其实就是返回 list 的大小
           final int getEst() {
               int s; // force initialization
               final LinkedList<E> lst;
               if ((s = est) < 0) { // -1
                   if ((lst = list) == null)
                       s = est = 0;
                   else {// list 不为 Null 我需要对自己的几个属性进行初始化值
                       expectedModCount = lst.modCount;
                       current = lst.first;
                       s = est = lst.size;
                   }
               }
               return s;
           }
   ```

   我初始化进来的时候，LLSpliterator 中的属性都是注释中的默认值，流程分析在上面中文注释中。

   **所以当大小 小于 1024 的时候，分割没效果，还浪费了计算的性能。**

   **还发现虽然它可以分割的，但是依然是非线程安全。**

   2. ``forEachRemaining(Consumer<? super E> action)`` 批量消费，不能重复消费。
   3. `tryAdvance`，单个消费，从第一个开始消费，消费完移动节点，以后无法消费这个节点，成功消费返回 `true`。

#### 其他方法

1. `indexof(Object)`,`lastindexof(Object)`

   （查找属性）找元素的索引，遍历查找，效率取决于离尾节点的位置；

2. `peek`，（队列属性）查看头节点的元素；`element`方法也是一个意思，感觉有点重复的意思。

3. `poll` ，（队列属性）删除头节点，并返回该节点的元素。

4. `offer`，（队列属性），在尾节点添加，同 `offerFirst`；

5. `removeFirstOccurrence(Object o) `，`removeLastOccurrence(Object o) ` 删除第一次或者最后一次出现该元素的节点。

#### 实现线程安全

1. `Collections.synchronizedList(LinkedList);`
2. 锁

#### 为什么 LinkedList 查找速度比 ArrayList 慢

1. `ArrayList`  实现 了 `RandomAccess ` 接口，并且它的数据结构是数组，空间连续的，可以使用索引下标直接访问元素；
2. `LinkedList` 使用的是`get(int)` 方法，它需要从头节点或者尾节点开始遍历，如果要查找的元素的离头节点或者尾节点很远的时候，速度自然而然的慢下来了。


#### 为什么 LinkedList 要使用迭代器而不使用 for 循环

1. 迭代器的源码很好理解，就是一直使用游标获取下一个节点的值。
2. 使用 for 循环的时候，我们肯定使用的是 `get(index)`这个方法，查看这个方法的源码发现，每次都是从头到尾遍历一遍，因为 LinkedLIst 是无序的，没有索引这个概念，只能够重新查找，当容量足够大后，这个性能损失得很明显。

这样对比起来差距就很明显的。

最后还需要注意一点：**使用迭代器循环 LinkedList 不比 for 循环的 ArrayList 的慢。**这个可以下去做个测试，最好 10 万条。

测试代码：

```java
public class Test {

    private enum Target {
        ARRAYLIST,
        LINKEDLIST;
    }

    public static void main(String[] args) {
        int size = 100000;
        List<Integer> aList = new ArrayList<>(size);
        List<Integer> lList = new LinkedList<>();
        for (int i = 0; i < size; i++) {
            aList.add(i);
            lList.add(i);
        }
        String msg;
        Target target;
        msg = "使用 for 循环 %s 十万次的时间 %s ms %n";
        target = Target.ARRAYLIST;
        long begin = System.currentTimeMillis();
        for (int i = 0; i < aList.size(); i++) {
            aList.get(i);
        }
        long end = System.currentTimeMillis();
        System.out.printf(msg, target, end - begin);

        target = Target.LINKEDLIST;
        begin = System.currentTimeMillis();
        for (int i = 0; i < lList.size(); i++) {
            lList.get(i);
        }
        end = System.currentTimeMillis();
        System.out.printf(msg, target, end - begin);

        msg = "使用 foreach 循环 %s 十万次的时间 %s ms%n";
        target = Target.ARRAYLIST;
        begin = System.currentTimeMillis();
        for (Integer anAList : aList) {
        }
        end = System.currentTimeMillis();
        System.out.printf(msg, target, end - begin);

        target = Target.LINKEDLIST;
        begin = System.currentTimeMillis();
        for (Integer aLList : lList) {
        }
        end = System.currentTimeMillis();
        System.out.printf(msg, target, end - begin);

        msg = "使用 jdk stream api 循环 %s 十万次的时间 %s ms%n";
        target = Target.ARRAYLIST;
        begin = System.currentTimeMillis();
        aList.stream().forEach(l -> {});
        end = System.currentTimeMillis();
        System.out.printf(msg, target, end - begin);

        target = Target.LINKEDLIST;
        begin = System.currentTimeMillis();
        lList.stream().forEach(l -> {});
        end = System.currentTimeMillis();
        System.out.printf(msg, target, end - begin);

        msg = "使用 jdk parallelStream api 循环 %s 十万次的时间 %s ms%n";
        target = Target.ARRAYLIST;
        begin = System.currentTimeMillis();
        aList.parallelStream().forEach(l -> {});
        end = System.currentTimeMillis();
        System.out.printf(msg, target, end - begin);

        target = Target.LINKEDLIST;
        begin = System.currentTimeMillis();
        lList.parallelStream().forEach(l -> {});
        end = System.currentTimeMillis();
        System.out.printf(msg, target, end - begin);
    }
}
```

测试结果 ：

```java
使用 for 循环 ARRAYLIST 十万次的时间 3 ms 
使用 for 循环 LINKEDLIST 十万次的时间 5397 ms 
使用 foreach 循环 ARRAYLIST 十万次的时间 3 ms
使用 foreach 循环 LINKEDLIST 十万次的时间 3 ms
使用 jdk stream api 循环 ARRAYLIST 十万次的时间 41 ms
使用 jdk stream api 循环 LINKEDLIST 十万次的时间 3 ms
使用 jdk parallelStream api 循环 ARRAYLIST 十万次的时间 7 ms
使用 jdk parallelStream api 循环 LINKEDLIST 十万次的时间 4 ms
```

所以建议都是用 foreach （迭代器）进行没有性能损失，主要是有快速失败机制。jdk 1.8 的 stream 的表达式使用起来方便，稍微有些性能损失，按照使用环境考虑得失。

#### LinkedList 插入速度一定比 ArrayList 快吗？

不一定。

**首先我们需要说下 LinkedList 的插入步骤：**

1. 构建一个 Node 对象（需要浪费性能）；
2. 寻址，找到对象要插入的位置（查找慢，如果插入的位置离头节点或者尾节点远，消耗的消费更多的性能）；
3. 更改对象的引用地址，插入 node 对象。

**再看下 ArrayList 的插入操作：**

1. 寻址，找到需要插入的位置；
2. 如果需要扩容就扩容，如果不是插入到最后的位置，空出一个位置提供给插入的元素，需要批量往后移动数组的位置；
3. 插入数据。

**然后根据它们的特性：**

1. LinkedList 插入的时候 为什么快，因为快在值需要改变前后 node 的引用地址就可以；
2. ArrayList 插入操作的时候为什么慢，因为慢在移动数组的位置上，移动的越多越慢。

**问题就清楚了：**

1. 如果你确定了 ArrayList 不会扩容的情况下，又比较少移动元素（插入或者删除操作在比较后的位置），ArrayList 的插入或者删除操作效率其实比 LinkedList 还高。
2. 如果 删除的元素在数据结构的非常靠前的位置，毫无疑问使用 LinkedList，
3. 如果它们的操做位置都是**在尾端进行插入操作**，而且需要插入的**元素的数量已知**的情况下，用 ArrayList 效率会更高些，不需要构建 Node 对象，就少消耗了很多引用内存。
4. 综合实际情况，还涉及到 ArrayList 扩容，插入元素是否在尾部等复杂因素。
