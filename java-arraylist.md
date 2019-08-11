---
title: Java 8 中ArrayList源码分析
date: 2017-11-4 21:08:03
tags: [java]
categories: 学习笔记
---

这次简单看下ArrayList的实现过程，以及它拥有的操作方法。
在Java 8 中 ArrayList 的实现 较以前有很大的改变。

<!--more-->

#### ArrayList 拥有的属性
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    private static final long serialVersionUID = 8683452581122892189L;

    /**
     * Default initial capacity.
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     */
     //被用于空实例的共享空数组实例
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
     //被用于默认大小的空实例的共享数组实例。其与EMPTY_ELEMENTDATA的区别是：当我们向数组中添加第一个元素时，知道数组该扩充多少。
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;
    
```
1. 实现的接口看出，支持随机访问，克隆，序列化；
2. 默认大小``DEFAULT_CAPACITY`` 为 10 ；
3.  ``elementData``存储数组数据的，是 Object[] 类型的数组；
4.  ``size`` 为当前 ArrayList 的实际大小。
#### 构造函数
ArrayList 通过构造方法创建有三种方法：
```java
/**
     * 构造一个指定初始容量的空列表
     * @param  initialCapacity  ArrayList的初始容量
     * @throws IllegalArgumentException 如果给定的初始容量为负值
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    // 构造一个默认初始容量为10的空列表，但是还没分配大小。
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * 构造一个包含指定collection的元素的列表，这些元素按照该collection的迭代器返回的顺序排列的
     * @param c 包含用于去构造ArrayList的元素的collection
     * @throws NullPointerException 如果指定的collection为空
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray()可能不会正确地返回一个 Object[]数组，那么使用Arrays.copyOf()方法
            if (elementData.getClass() != Object[].class)
                //Arrays.copyOf()返回一个 Object[].class类型的，大小为size，元素为elementData[0,...,size-1]
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

#### 添加元素add
最简单的添加方法，在 ArrayList 尾部添加一个元素，需要去扩容，这个是ArrayList 最重要的一个特点；
``` java
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```
##### 扩容
下面是扩容的重要代码：
```java
    protected transient int modCount = 0;

    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
每当向数组中添加元素时，都要去检查添加元素后的个数是否会超出当前数组的长度，如果超出，数组将会进行扩容，都回去调用方法``ensureCapacityInternal(int minCapacity)``
在这个方法中看到，那个if语句判断就是，我们使用默认无参的构造函数创建的ArrayList 是在这里去 给大小的，如果第一次 add 的元素长度大于默认长度的话，就是用新的长度，否则给默认大小10；

给定大小后，就去调用``grow``方法，进行扩容。
看到``int newCapacity = oldCapacity + (oldCapacity >> 1);`` ArrayList 每次扩容的大小是当前容量的0.5倍，就是默认大小为10，下次扩容后大小为15，下次再扩容后为 *15 * 1.5*；所以ArrayList每次扩容的容量只会越来越大。

  ``modCount``用于记录ArrayList的结构性变化的次数，add()、remove()、addall()、removerange()及clear()方法都会让modCount增长。

##### 其余的add方法，addAll
```java
//将指定的元素(E e)添加到此列表的尾部
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

    //将指定的元素(E e)插入到列表的指定位置(index)
    public void add(int index, E element) {
        rangeCheckForAdd(index); //判断参数index是否IndexOutOfBoundsException

        ensureCapacityInternal(size + 1);  // Increments modCount!!  如果数组长度不足，将进行扩容
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index); //将源数组中从index位置开始后的size-index个元素统一后移一位
        elementData[index] = element;
        size++; //重新指定siez 大小
    }

    /**
     * 按照指定collection的迭代器所返回的元素顺序，将该collection中的所有元素添加到此列表的尾部
     * @throws NullPointerException if the specified collection is null
     */
    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        //将数组a[0,...,numNew-1]复制到数组elementData[size,...,size+numNew-1]
        System.arraycopy(a, 0, elementData, size, numNew); 
        size += numNew; //重新指定size 大小
        return numNew != 0;
    }

    /**
     * 从指定的位置开始，将指定collection中的所有元素插入到此列表中，新元素的顺序为指定collection的迭代器所返回的元素顺序
     * @throws IndexOutOfBoundsException {@inheritDoc}
     * @throws NullPointerException if the specified collection is null
     */
    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index); //判断参数index是否IndexOutOfBoundsException

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount

        int numMoved = size - index;
        if (numMoved > 0)
            //先将数组elementData[index,...,index+numMoved-1]复制到elementData[index+numMoved,...,index+2*numMoved-1]
            //即，将源数组中从index位置开始的后numMoved个元素统一后移numNew位
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);
        //再将数组a[0,...,numNew-1]复制到数组elementData[index,...,index+numNew-1]
        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }
```
#### remove 删除元素
```java
/**
     * 移除此列表中指定位置上的元素
     * @param index 需被移除的元素的索引
     * @return the element 被移除的元素值
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E remove(int index) {
        rangeCheck(index);  //判断index是否 <= size

        modCount++;
        E oldValue = elementData(index);
        //将数组elementData中index位置之后的所有元素向前移一位
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; //将原数组最后一个位置置为null，由GC清理

        return oldValue;
    }

    //移除ArrayList中首次出现的指定元素(如果存在)，ArrayList中允许存放重复的元素
    public boolean remove(Object o) {
        // 由于ArrayList中允许存放null，因此下面通过两种情况来分别处理。
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index); //私有的移除方法，跳过index参数的边界检查以及不返回任何值
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    //私有的删除指定位置元素的方法，跳过index参数的边界检查以及不返回任何值
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }

    //清空ArrayList，将全部的元素设为null
    public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }

    //删除ArrayList中从fromIndex（包含）到toIndex（不包含）之间所有的元素，共移除了toIndex-fromIndex个元素
    protected void removeRange(int fromIndex, int toIndex) {
        modCount++;
        int numMoved = size - toIndex;  //需向前移动的元素的个数
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

        // clear to let GC do its work
        int newSize = size - (toIndex-fromIndex);
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }

    //删除ArrayList中包含在指定容器c中的所有元素
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);  //检查指定的对象c是否为空
        return batchRemove(c, false);
    }

    //移除ArrayList中不包含在指定容器c中的所有元素，与removeAll(Collection<?> c)正好相反
    public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c); 
        return batchRemove(c, true);
    }

	//批量删除
    //complement为true 表示不同的删除，
    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;  //读写双指针
        boolean modified = false;
        try {
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement) //判断指定容器c中是否含有elementData[r]元素
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```
#### 修改元素 set
```java
//将指定索引上的值替换为新值，并返回旧值
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```

#### 查询
```java
//判断ArrayList中是否包含Object(o)
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    //正向查找，返回ArrayList中元素Object o第一次出现的位置，如果元素不存在，则返回-1
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)                 
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

    //逆向查找，返回ArrayList中元素Object o最后一次出现的位置，如果元素不存在，则返回-1
    public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

    @SuppressWarnings("unchecked")
    E elementData(int index) {
        return (E) elementData[index];
    }

    //返回指定索引处的值
    public E get(int index) {
        rangeCheck(index);

        return elementData(index); //实质上return (E) elementData[index]
    }
```
#### 其他方法
```java
//将底层数组的容量调整为当前列表保存的实际元素的大小的功能
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }

    //返回ArrayList的大小（元素个数）
    public int size() {
        return size;
    }

   //判断ArrayList是否为空
    public boolean isEmpty() {
        return size == 0;
    }

    //返回此 ArrayList实例的浅拷贝
    public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }
    
    //返回一个包含ArrayList中所有元素的数组
    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }

    //如果给定的参数数组长度足够，则将ArrayList中所有元素按序存放于参数数组中，并返回
    //如果给定的参数数组长度小于ArrayList的长度，则返回一个新分配的、长度等于ArrayList长度的、包含ArrayList中所有元素的新数组
    @SuppressWarnings("unchecked")
    public <T> T[] toArray(T[] a) {
        if (a.length < size)
            // Make a new array of a's runtime type, but my contents:
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }
```

#### 遍历方法
```java
package com.wuwii.test;

import java.util.*;

/**
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/11/4 16:53</pre>
 */
public class TestArrayList {
    public static void main(String[] args) {
        List<String> seasons = new ArrayList();// 创建默认大小ArrayList
        seasons.add("spring"); //第一次赋值，才有大小
        seasons.addAll(Arrays.asList("summer", "autumn", "winter"));
        //使用迭代器 Iterator
        Iterator iterator = seasons.iterator();
        while (iterator.hasNext()) {
            System.out.println(iterator.next());
        }

        //使用迭代器 ListIterator
        System.out.println("使用迭代器 ListIterator");
        ListIterator listIterator = seasons.listIterator();
        while (listIterator.hasNext()) {
            System.out.println(listIterator.next());
        }
        System.out.println("使用迭代器 ListIterator，逆向访问");
        while (listIterator.hasPrevious()) {
            System.out.println(listIterator.nextIndex() + " : " + listIterator.previous());
        }

        //通过索引 ，随机访问
        System.out.println("通过索引 ，随机访问");
        for (int i = 0, len = seasons.size(); i < len; i++) System.out.println(seasons.get(i));

        // 使用foreach 遍历
        System.out.println("使用foreach 遍历");
        for (String season : seasons) System.out.println(season);
        //第二种写法
		//@since 1.8 
		//@see Iterable
        seasons.forEach(System.out::println);
    }
}

```
Iterator与ListIterator的区别：
1. Iterator可以应用于所有的集合，Set、List和Map和这些集合的子类型。而ListIterator只能用于List及其子类型；
2. Iterator只能实现顺序向后遍历，ListIterator可实现顺序向后遍历和逆向（顺序向前）遍历；
3. Iterator只能实现remove操作，ListIterator可以实现remove操作，add操作，set操作。
#### 多线程中使用ArrayList
当多个线程同时修改一个ArrayList对象的时候，必须要保持外部同步操作，但是ArrayList不是同步的，非线程安全，有一种办法就是可以使用``Collections.synchronizedList``进行包装：
```java
List list = Collections.synchronizedList(new ArrayList(...));
```
但是在平时开发中，多线程开发中多选择使用``Vector``或者``CopyOnWriteArrayList``。

#### 补充：toArray 方法

##### toArray()

返回的是java.lang.Object的数组，不建议使用类型强转，可能会存在空指针异常。

##### toArray(**T[]**)

源码：

```java
    public <T> T[] toArray(T[] var1) {
        if (var1.length < this.size) {
            // 如果参数的长度比 list.size()小的话，将数组拷贝到新的数据中，且指定类型强制转换，不会存在空指针异常。
            return (Object[])Arrays.copyOf(this.elementData, this.size, var1.getClass());
        } else {
            // 如果长度不小于的话，将list中的元素拷贝到传进来参数 var1 的数据中，起点都为0，长度为size，
            System.arraycopy(this.elementData, 0, var1, 0, this.size);
            // 如果长度大于size 则将副本数组中的索引为 size 的元素设置为Null，
            if (var1.length > this.size) {
                var1[this.size] = null;
            }
            return var1;
        }
    }
```

所以上面就会出现一个现象，索引为 size 的元素，一直会为null ，可以检验下的。
