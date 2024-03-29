---
title: 解决Hash碰撞冲突方法总结
date: 2017-11-15 21:08:03
tags: [post]
categories: 学习笔记
---

### Hash碰撞冲突
我们知道，对象Hash的前提是实现equals()和hashCode()两个方法，那么HashCode()的作用就是保证对象返回唯一hash值，但当两个对象计算值一样时，这就发生了碰撞冲突。如下将介绍如何处理冲突，当然其前提是一致性hash。

<!--more-->

### 解决冲突
#### 开放地址法
开放地执法有一个公式:``Hi=(H(key)+di) MOD m i=1,2,…,k(k<=m-1)``
其中，m为哈希表的表长。di 是产生冲突的时候的增量序列。如果di值可能为1,2,3,…m-1，称线性探测再散列。
如果di取1，则每次冲突之后，向后移动1个位置.如果di取值可能为1,-1,2,-2,4,-4,9,-9,16,-16,…k*k,-k*k(k<=m/2)，称二次探测再散列。
如果di取值可能为伪随机数列。称伪随机探测再散列。
#### 再哈希法
当发生冲突时，使用第二个、第三个、哈希函数计算地址，直到无冲突时。缺点：计算时间增加。
比如上面第一次按照姓首字母进行哈希，如果产生冲突可以按照姓字母首字母第二位进行哈希，再冲突，第三位，直到不冲突为止
#### 链地址法（拉链法）
将所有关键字为同义词的记录存储在同一``线性链表``中。如下：

![image](http://img.blog.csdn.net/20160918154444663?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### 建立一个公共溢出区
假设哈希函数的值域为``[0,m-1]``,则设向量``HashTable[0..m-1]``为基本表，另外设立存储空间向量``OverTable[0..v]``用以存储发生冲突的记录。

#### 拉链法的优缺点
##### 优点
1. 拉链法处理冲突简单，且无堆积现象，即非同义词决不会发生冲突，因此平均查找长度较短；
2. 由于拉链法中各链表上的结点空间是动态申请的，故它更适合于造表前无法确定表长的情况；
3. 开放定址法为减少冲突，要求装填因子α较小，故当结点规模较大时会浪费很多空间。而拉链法中可取α≥1，且结点较大时，拉链法中增加的指针域可忽略不计，因此节省空间；
4. 在用拉链法构造的散列表中，删除结点的操作易于实现。只要简单地删去链表上相应的结点即可。而对开放地址法构造的散列表，删除结点不能简单地将被删结 点的空间置为空，否则将截断在它之后填人散列表的同义词结点的查找路径。这是因为各种开放地址法中，空地址单元(即开放地址)都是查找失败的条件。因此在 用开放地址法处理冲突的散列表上执行删除操作，只能在被删结点上做删除标记，而不能真正删除结点。

##### 缺点
指针需要额外的空间，故当结点规模较小时，开放定址法较为节省空间，而若将节省的指针空间用来扩大散列表的规模，可使装填因子变小，这又减少了开放定址法中的冲突，从而提高平均查找速度。

**文章转载 <a rel="external nofollow" target="_blank" href="http://blog.csdn.net/zeb_perfect/article/details/52574915">解决Hash碰撞冲突方法总结</a>**