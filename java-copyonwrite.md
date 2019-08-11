---
title: 并发容器CopyOnWriteArrayList
date: 2017-11-26 21:08:03
tags: [java,并发编程]
categories: 学习笔记
---

Copy-On-Write简称COW，是一种用于程序设计中的``优化策略``。其基本思路是，从一开始大家都在共享同一个内容，当某个人想要修改这个内容的时候，才会真正把内容Copy出去形成一个新的内容然后再改，这是一种延时懒惰策略。从JDK1.5开始Java并发包里提供了两个使用CopyOnWrite机制实现的并发容器,它们是CopyOnWriteArrayList和CopyOnWriteArraySet。CopyOnWrite容器非常有用，可以在非常多的并发场景中使用到。

### 什么是CopyOnWrite容器
CopyOnWrite容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。

<!--more-->

### CopyOnWriteArrayList如何做到线程安全的

CopyOnWriteArrayList使用了一种叫``写时复制``的方法，当有新元素添加到CopyOnWriteArrayList时，先从原有的数组中拷贝一份出来，然后在新的数组做写操作，写完之后，再将原来的数组引用指向到新数组。

```java
public boolean add(E e) {
    //1、先加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        //2、拷贝数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        //3、将元素加入到新数组中
        newElements[len] = e;
        //4、将array引用指向到新数组
        setArray(newElements);
        return true;
    } finally {
       //5、解锁
        lock.unlock();
    }
}
```
可以看出来CopyOnWriteArrayList的整个add操作都是在锁的保护下进行的。

当有新元素加入的时候，如下图，创建新数组，并往新数组中加入一个新元素,这个时候，array这个引用仍然是指向原数组的。

![image](http://img.blog.csdn.net/20170117145928110?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGluc29uZ2JpbjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
*图片来自水印*

当元素在新数组添加成功后，将array这个引用指向新数组。

![image](http://img.blog.csdn.net/20170117150336836?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGluc29uZ2JpbjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
*图片来自水印*

读取操作：
```java
public E get(int index) {
    return get(getArray(), index);
}
```

**但是它的读操作并没有同步，因此读取它的数据的时候不一定是最新的数据。**

### 总结
1. 注意内存的消耗，每次进行写入操作的时候，都会复制一个副本，如果这些对象占用的内存比较大，比如说200M左右，那么再写入100M数据进去，内存就会占用300M，那么这个时候很有可能造成频繁的Yong GC和Full GC。针对内存占用问题，可以通过压缩容器中的元素的方法来减少大对象的内存消耗，比如，如果元素全是10进制的数字，可以考虑把它压缩成36进制或64进制。或者不使用CopyOnWrite容器，而使用其他的并发容器，如ConcurrentHashMap。
2. 不能用于实时读的场景，像拷贝数组、新增元素都需要时间，所以调用一个set操作后，读取到数据可能还是旧的,虽然CopyOnWriteArrayList 能做到最终一致性,但是还是没法满足实时性要求；
3. CopyOnWriteArrayList 合适读多写少的场景，不过这类慎用 
因为谁也没法保证CopyOnWriteArrayList 到底要放置多少数据，万一数据稍微有点多，每次add/set都要重新复制数组，这个代价实在太高昂了。在高性能的互联网应用中，这种操作分分钟引起故障。
4. 设计思想：并发时候，可以开辟新的地址，来解决并发问题。

**参考文章：**
* <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/linsongbin1/article/details/54581787">线程安全的CopyOnWriteArrayList介绍</a>
* <a rel="external nofollow" target="_blank" href="https://www.cnblogs.com/dolphin0520/p/3938914.html">Java并发编程：并发容器之CopyOnWriteArrayList（转载）</a>
