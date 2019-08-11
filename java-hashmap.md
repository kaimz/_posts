---
title: Java 8 中HashMap源码分析
date: 2017-11-16 11:08:03
tags: [java]
categories: 学习笔记
---

### HashMap 文档
>　HashMap是基于哈希表的Map接口实现的,此实现提供所有可选的映射操作。存储的是``<key，value>``对的映射，允许多个null值和一个null键。但此类不保证映射的顺序，特别是它不保证该顺序恒久不变。
 　除了HashMap是非同步以及允许使用null外，HashMap 类与 Hashtable大致相同。
　 此实现假定哈希函数将元素适当地分布在各桶之间，可为基本操作（get 和 put）提供稳定的性能。迭代collection 视图所需的时间与 HashMap 实例的“容量”（桶的数量）及其大小（键-值映射关系数）成比例。所以，如果迭代性能很重要，则不要将初始容量设置得太高（或将加载因子设置得太低）。
>
>　　HashMap 的实例有两个参数影响其性能：``初始容量`` 和``加载因子``。容量 是哈希表中桶的数量，初始容量只是哈希表在创建时的容量。**加载因子 是哈希表在其容量自动增加之前可以达到多满的一种尺度。当哈希表中的条目数超出了加载因子与当前容量的乘积时，则要对该哈希表进行 rehash 操作（即重建内部数据结构），从而哈希表将具有大约两倍的桶数。**     
>     
>　　通常，默认``加载因子`` (0.75) 在时间和空间成本上寻求一种折衷。加载因子过高虽然减少了空间开销，但同时也增加了查询成本（在大多数 HashMap 类的操作中，包括 get 和 put 操作，都反映了这一点）。**在设置初始容量时应该考虑到映射中所需的条目数及其加载因子，以便最大限度地减少 rehash 操作次数。如果初始容量大于最大条目数除以加载因子，则不会发生 rehash 操作。**  
>　　注意，此实现``不是同步``的。 如果多个线程同时访问一个HashMap实例，而其中至少一个线程从结构上修改了列表，那么它必须保持外部同步。这通常是通过同步那些用来封装列表的 对象来实现的。但如果没有这样的对象存在，则应该使用{@link Collections#synchronizedMap Collections.synchronizedMap}来进行“包装”，该方法最好是在创建时完成，为了避免对映射进行意外的非同步操作。
```java
Map m = Collections.synchronizedMap(new HashMap(...));
```
>由所有此类的“collection 视图方法”所返回的迭代器都是快速失败的：在迭代器创建之后，如果从结构上对映射进行修改，除非通过迭代器本身的remove 方法，其他任何时间任何方式的修改，迭代器都将抛出 ConcurrentModificationException。因此，面对并发的修改，迭代器很快就会完全失败，而不会在将来不确定的时间发生任意不确定行为的风险。
>
>注意，迭代器的``快速失败``行为不能得到保证，一般来说，存在非同步的并发修改时，不可能作出任何坚决的保证。快速失败迭代器尽最大努力抛出 ``ConcurrentModificationException``。因此，编写依赖于此异常的程序的做法是错误的，正确做法是：迭代器的快速失败行为应该仅用于检测程序错误。

<!--more-->

**jdk版本：jdk1.8.0_144**

### HashMap的数据结构
HashMap实际上是一个“链表的数组”的数据结构，每个元素存放链表头结点的数组，即数组（散列桶）中的每一个元素都是链表。
![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwxdigp23j20qy0kedhk.jpg)

#### 解决Hash冲突
 HashMap就是使用哈希表来存储的。哈希表为解决冲突，可以采用开放地址法和链地址法等来解决问题，Java中HashMap采用了``链地址法（拉链法）``。链地址法，简单来说，就是数组加链表的结合。在每个数组元素上都一个链表结构，当数据被Hash后，得到数组下标，把数据放在对应下标元素的链表上。
 有时候计算Hash值的时候，会出现相同的情况，这样两个key就存储到相同的位置上了，这个时候会出现``Hash碰撞``。
### HashMap的属性
#### 实现的接口和继承的类
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```
实际上HashMap没有从AbstractMap父亲中继承任何属性，从实现的接口上看，HashMap拥有克隆和序列化的属性。

#### 属性

```java
//默认初始容量16，必须为2的幂
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    //最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;

    //默认加载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
    //使用红黑树而不是链表的阈值
    static final int TREEIFY_THRESHOLD = 8;
    
    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    //table是一个Node<K,V>[]数组类型，而Node<K,V>实际上就是一个元素值为<key,value>对的单向链表。
    //哈希表的"key-value键值对"都是存储在Node<K,V>数组中的。 
    transient Node<K,V>[] table;

    //用来指向entrySet()返回的set集合
    transient Set<Map.Entry<K,V>> entrySet;


    //HashMap的大小,即保存的键值对的数量
    transient int size;

    //用来实现fail-fast机制的，记录HashMap结构化修改的次数
    transient int modCount;

    //下次需扩容的临界值，size>=threshold就会扩容
    //如果table数组没有被分配，则该值为初始容量值16；或若该值为0，也表明该值为初始容量值
    int threshold;

    //加载因子
    final float loadFactor;
```
##### table
table是一个Node[]数组类型，而Node实际上就是一个单向链表，哈希桶数组。哈希表的"key-value键值对"都是存储在Node数组中的。
```java
//实现Map.Entry<K,V>接口
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash; //hash码
        final K key;
        V value;
        Node<K,V> next; //指向链表中下一个实例

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        //返回此映射项的哈希值:key值的哈希码与value值的哈希码按位异或的结果
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        //用指定值替换对应于此项的值,并返回旧值
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        //比较指定对象与此项的相等性
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```
在HashMap中，哈希桶数组table的长度length大小必须为2的n次方，而当链表长度太长（默认超过8）时，链表就转换为红黑树，利用红黑树快速增删改查的特点提高HashMap的性能，其中会用到红黑树的插入、删除、查找等算法。
##### loadFactor加载因子
HashMap的初始化大小length为16（默认值），默认加载因子0.75，threshold是HashMap所能容纳的最大数据量的Node(键值对)个数。threshold = length * Load factor。也就是说，在数组定义好长度之后，负载因子越大，所能容纳的键值对个数越多。

threshold就是在此Load factor和length(数组长度)对应下允许的最大元素数目，超过这个数目就重新resize(扩容)，扩容后的HashMap容量是之前容量的两倍。默认的负载因子0.75是对空间和时间效率的一个平衡选择，建议大家不要修改，除非在时间和空间比较特殊的情况下，如果内存空间很多而又对时间效率要求很高，可以降低负载因子Load factor的值；相反，如果内存空间紧张而对时间效率要求不高，可以增加负载因子loadFactor的值，这个值可以大于1。

##### size大小
HashMap中实际存在的键值对数量。

### 方法
#### 构造函数
HashMap提供了四种方式的构造器，可以构造一个带指定初始容量和加载因子的空HashMap，构造一个带指定初始容量和默认加载因子(0.75)的空 HashMap，构造一个默认初始容量为16和默认加载因子为0.75的空HashMap，以及构造一个包含指定Map的元素的HashMap，容量与指定Map容量相同，加载因子为默认的0.75。
```java
//找出“大于Capacity”的最小的2的幂,使Hash表的容量保持为2的次方倍
    //算法的思想：通过使用逻辑运算来替代取余，这里有一个规律，就是当N为2的次方（Power of two），那么X％N==X&(N-1)。
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1; //>>> 无符号右移,高位补0
        n |= n >>> 2; //a|=b的意思就是把a和b按位或然后赋值给a
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

    //构造一个带指定初始容量和加载因子的空HashMap
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

    //构造一个带指定初始容量和默认加载因子(0.75)的空 HashMap
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    //构造一个具有默认初始容量 (16)和默认加载因子 (0.75)的空 HashMap
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    //构造一个映射关系与指定 Map相同的新 HashMap,容量与指定Map容量相同，加载因子为默认的0.75
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```
#### 确定哈希桶数组索引位置
不管增加、删除、查找键值对，定位到哈希桶数组的位置都是很关键的第一步。前面说过HashMap的数据结构是数组和链表的结合，所以我们当然希望这个HashMap里面的元素位置尽量分布均匀些，尽量使得每个位置上的元素数量只有一个，那么当我们用hash算法求得这个位置的时候，马上就可以知道对应位置的元素就是我们要的，不用遍历链表，大大优化了查询的效率。HashMap定位数组索引位置，直接决定了hash方法的离散性能。先看看源码的实现(方法一+方法二):
```java
方法一：
static final int hash(Object key) {   //jdk1.8 & jdk1.7
     int h;
     // h = key.hashCode() 为第一步 取hashCode值
     // h ^ (h >>> 16)  为第二步 高位参与运算
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
方法二：
static int indexFor(int h, int length) {  //jdk1.7的源码，jdk1.8没有这个方法，但是实现原理一样的
     return h & (length-1);  //第三步 取模运算
}
```
这里的Hash算法本质上就是三步：取key的hashCode值、高位运算、取模运算。

对于任意给定的对象，只要它的hashCode()返回值相同，那么程序调用方法一所计算得到的Hash码值总是相同的。我们首先想到的就是把hash值对数组长度取模运算，这样一来，元素的分布相对来说是比较均匀的。但是，模运算的消耗还是比较大的，在HashMap中是这样做的：调用方法二来计算该对象应该保存在table数组的哪个索引处。

这个方法非常巧妙，它通过h & (table.length -1)来得到该对象的保存位，而HashMap底层数组的长度总是2的n次方，这是HashMap在速度上的优化。当length总是2的n次方时，h& (length-1)运算等价于对length取模，也就是h%length，但是&比%具有更高的效率。

在JDK1.8的实现中，优化了高位运算的算法，通过hashCode()的高16位异或低16位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。
![image](http://tech.meituan.com/img/java-hashmap/hashMap%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE.png)

#### HashMap的put方法
HashMap提供了put(K key, V value)、putAll(Map<? extends K, ? extends V> m)这些添加键值对的方法。
HashMap的put方法执行过程可以通过下图来理解，
![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwxdibl3ij21bo11s7cj.jpg)
1. 判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；

2. 根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向⑥，如果table[i]不为空，转向③；

3. 判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向④，这里的相同指的是hashCode以及equals；

4. 判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向⑤；

5. 遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；

6. 插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

##### put方法源码
```java
/**
     * 在此映射中关联指定值与指定键。如果该映射以前包含了一个该键的映射关系，则旧值被替换。
     * 
     * @param key 指定值将要关联的键
     * @param value 指定键将要关联的值
     * @return 与 key关联的旧值；如果 key没有任何映射关系，则返回 null。（返回 null 还可能表示该映射之前将null与 key关联。）
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * 用于实现 Map.put()和相关的方法
     *
     * @param hash 键的hash码
     * @param key 键
     * @param value 值
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict evict=false：表明该hash表处于初始化创建的过程中
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //步骤 1 ：tab为空则创建  
        //此处分两种情况：1.当table为null时，用默认容量16初始化table数组；2.当table非空时
        if ((tab = table) == null || (n = tab.length) == 0) //旧hash表为null或旧hash表长度为0
            n = (tab = resize()).length;  //初始化hash表的长度（16）
        //步骤 2
        //此处又分为两种情况：1.key的hash值对应的那个节点为空；2.key的hash值对应的那个节点不为空
        if ((p = tab[i = (n - 1) & hash]) == null) //该key的hash值对应的那个节点为空，即表示还没有元素被散列至此
            tab[i] = newNode(hash, key, value, null); //则创建一个新的new Node<>(hash, key, value, next);
        else {
             //该key的hash值对应的那个节点不为空，先与链表上的第一个节点p比较
            Node<K,V> e; K k;
            // 步骤 3：节点key存在，直接覆盖value  
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
                // 步骤 4：判断该链为红黑树  
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            // 步骤 5：该链为链表 的情况下进行遍历table
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {  
                        p.next = newNode(hash, key, value, null);
                        //链表长度大于8转换为红黑树进行处理 TREEIFY_THRESHOLD = 8  
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // key已经存在的话
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;  //向后查找
                }
            }
            //若该key对应的value已经存在，则用新的value取代旧的value
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        // 步骤 6：如果加入该键值对后超过最大阀值，则进行resize操作 ，就扩容  threshold：
        //单词解释--阈(yu)值,不念阀(fa)值！顺便学下语文咯。  
        if (++size > threshold)  
            resize();
        afterNodeInsertion(evict);
        return null;
    }

    //将指定映射的所有映射关系复制到此映射中，这些映射关系将替换此映射目前针对指定映射中所有键的所有映射关系。
    public void putAll(Map<? extends K, ? extends V> m) {
        putMapEntries(m, true);
    }

    //用于帮助实现Map.putAll()方法 和Map构造器，当evict=false时表示构造初始HashMap。
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size(); //得到指定Map的大小
        if (s > 0) {
            if (table == null) { // pre-size
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);  //得到按指定Map大小计算出的HashMap所需的容量
                if (t > threshold)  //如果容量大于阈值
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)  //指定Map的大小>扩容临界值,扩容  
                resize();
            //通过迭代器，将“m”中的元素逐个添加到HashMap中
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
```
#### HashMap的扩容机制resize
在HashMap的四种构造函数中并没有对其成员变量Node<K,V>[] table进行任何初始化的工作，那么HashMap是如何构造一个默认初始容量为16的空表的？该初始化的诱发条件是在向HashMap中添加第一对<key,value>时，通过``put(K key, V value) -> putVal(hash(key), key, value, false, true) -> resize()``方法。故HashMap中尤其重要的resize()方法主要实现了两个功能：
1. 在table数组为null时，对其进行初始化，默认容量为16；
2. 当tables数组非空，但需要调整HashMap的容量时，将hash表容量翻倍。

jdk1.8中的resize：
```java
//resize()方法作用有两种：1.初始化hash表的容量，为16； 2.将hash表容量翻倍
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;  //旧hash表
        int oldCap = (oldTab == null) ? 0 : oldTab.length; //旧hash表容量
        int oldThr = threshold; //旧hash表阈值
        int newCap, newThr = 0;  //新hash表容量与扩容临界值
        //2.旧hash表非空，则表容量翻倍
        if (oldCap > 0) { 
            //如果当前的hash表长度已经达到最大值，则不在进行调整
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }  //更新新hash表容量：翻倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //更新扩容临界值
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr; 
        //1. 初始化hash表容量，设为默认值16，并且计算临界值。
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //设置下次扩容的临界值
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        //创建一个初始容量为新hash表长度的newTab数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //如果旧hash表非空，则按序将旧hash表中的元素重定向到新hash表
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;  //e按序指向oldTab数组中的元素，即每个链表中的头结点
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null) //如果链表只有一个头节点
                        newTab[e.hash & (newCap - 1)] = e;
                    // 如果节点是红黑树
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //对链表进行秩序维护：因为我们使用的是两倍扩容的方法，所以每个桶里面的元素必须要么待在原来的
                    //索引所对应的位置，要么在新的桶中位置偏移两倍
                    else { 
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
扩容是使用2次幂的扩展(指长度扩为原来2倍)，所以，
**经过rehash之后，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置**。

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwxdhq3ooj219c0ce761.jpg)

元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwxdi5angj20tk05mgmn.jpg)

因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图：

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fmwxghads4j20z80kadlq.jpg)

这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。这一块就是JDK1.8新增的优化点。有一点注意区别，JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，但是从上图可以看出，JDK1.8不会倒置。

#### 查找
HashMap提供了get(Object key)、containsKey(Object key)、containsValue(Object value)这些查找键值对的方法。
```java
//返回指定key所映射的value；如果对于该键来说，此映射不包含任何映射关系，则返回 null
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

   final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) { //key的哈希值为数组下标
            if (first.hash == hash && //检查第一个节点
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first; 
            //如果第一个节点不对，则向后检查
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

    //如果此映射包含对于指定键的映射关系，则返回 true。
    public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }

    //如果此映射将一个或多个键映射到指定值，则返回 true。
    public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            //外层循环搜索数组
            for (int i = 0; i < tab.length; ++i) {
                //内层循环搜索链表
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }
```

#### 清空与删除
HashMap提供了remove(Object key)删除键值对、clear()清除所有键值对的方法。

```java
//从此映射中移除指定键的映射关系（如果存在）
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

    /**
     * 用于实现 Map.remove()方法和其他相关的方法
     *
     * @param hash 键的hash值
     * @param key 键
     * @param value the value to match if matchValue, else ignored
     * @param matchValue if true only remove if value is equal
     * @param movable if false do not move other nodes while removing
     * @return the node, or null if none
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        //table数组非空，键的hash值所指向的数组中的元素非空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;  //node指向最终的结果结点，e为链表中的遍历指针
            if (p.hash == hash &&   //检查第一个节点，如果匹配成功
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            //如果第一个节点匹配不成功，则向后遍历链表查找
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)  //删除node结点
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }

    //从此映射中移除所有映射关系
    public void clear() {
        Node<K,V>[] tab;
        modCount++;
        if ((tab = table) != null && size > 0) {
            size = 0;
            for (int i = 0; i < tab.length; ++i)
                tab[i] = null;
        }
    }
```
### 总结
1. Java 8 中HashMap是数组+链表+红黑树；

2. 哈希桶数组table的长度length大小必须为2的n次方，也就是我想要创建一个长度为19的HashMap，那么它需要创建的大小为32；HashMap 的 bucket 数组大小一定是2的幂，如果 new 的时候指定了容量且不是2的幂，实际容量会是最接近(大于)指定容量的2的幂，比如 new HashMap<>(19)，比19大且最接近的2的幂是32，实际容量就是32。

3. 没有特殊要求，负载因子使用默认值0.75，并且它可以大于1；加载因子是表示Hsah表中元素的填满的程度。若加载因子越大，填满的元素越多，好处是，空间利用率高了，但冲突的机会加大了。反之，加载因子越小，填满的元素越少，好处是，冲突的机会减小了，但空间浪费多了。冲突的机会越大，则查找的成本越高；反之,查找的成本越小，因而,查找时间就越小.

4. HashMap是线程不安全的，不要在并发的环境中同时操作HashMap，建议使用ConcurrentHashMap，HashTable的并发性不如ConcurrentHashMap；

5. 扩容特别消耗性能，初始化的时候，尽量控制好HashMap的大小，避免频繁扩容；

6. HashMap 在 new 后并不会立即分配哈希桶数组，而是第一次 put 时初始化，类似 ArrayList 在第一次 add 时分配空间。

7. HashMap 在 put 的元素后，如果数量大于 ``Capacity * LoadFactor``（默认``16 * 0.75``） 之后会进行扩容。

8. Java 8在哈希碰撞的链表长度达到TREEIFY_THRESHOLD（默认8)后，会把该链表转变成树结构，提高了性能。

9. Java 8在 resize() 的时候，通过巧妙的设计，减少了 rehash 的性能消耗，主要是体现在算法上判断该值需不要移动位置。主要体现在下面一段代码上，判断 hash 的最高位的结果为真的话，就是用原来的索引，否则使用 `原索引+oldCap` :

   ```
   （e.hash & oldCap) == 0
   ```

   省去了重新计算hash值的时间。1.7 之前链表元素会进行倒置，1.8的时候没有这种。

10. 为什么table长度必须为二次幂，主要是体现在当我们需要 put 元素的时候，使用 hashCode 去判断该元素放在哪一个 table 里，例如，我们初始化一个 MAP，它的大小默认是16，但是 hashCode 是个很长 的 int，并不能确定 key 值放在哪个位置 ，源码中提供了取模运算 `(n - 1) & hash` ，(16-1) 的二进制 `1111`，不管和谁做与运算，最后的值都是 `0 -15` 之间的，就可以当作索引，放进相应的 table 中。加入我们将map 的容量设置成 15 ，它的取模运算就是 `14 & hash`，但是 14 的二进制是 `1110` ，当和我们的 hash 值做与运算的时候， hash 的最后以为不管是 0 还是 1 ，最终的结果都是一样的，这样就会造成 Hash 碰撞。所以为二次幂可以降低 hash 碰撞，提高搜索效率，而且计算机中都是采用二进制，这样就可以使用位运算符，提高计算的效率。

11.  hashMap 为了减少 hash 碰撞，重新对 key 进行了hash，让 hashCode 的高十六位和低十六做与运算。

    ```java
    static final int hash(Object key) {
            int h;
            return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
        }
    ```

**参考文章**

<a rel="external nofollow" target="_blank" href="http://blog.csdn.net/qq_27093465/article/details/52207135">Java 8系列之重新认识HashMap</a>  
<a rel="external nofollow" target="_blank" href="http://www.cnblogs.com/CherishFX/p/4739712.html">jdk1.8.0_45源码解读——HashMap的实现</a>  
<a rel="external nofollow" target="_blank" href="https://www.cnblogs.com/rogerluo1986/p/5851300.html">HashMap数据结构</a>  
<a rel="external nofollow" target="_blank" href="http://blog.csdn.net/u011411283/article/details/48024999">HashMap的性能提升从之链表到二叉树</a>
