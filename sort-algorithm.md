---
title: 学习几种简单排序算法
date: 2018-2-23 22:30:03
tags: [数据结构和算法]
categories: 学习笔记
---

分类：
* 插入排序（直接插入排序、希尔排序）
* 交换排序（冒泡排序、快速排序）
* 选择排序（直接选择排序、堆排序）
* 归并排序
* 分配排序（基数排序）

所需辅助空间最多：归并排序
所需辅助空间最少：堆排序
平均速度最快：快速排序
不稳定：快速排序，希尔排序，堆排序。

![image](https://ws1.sinaimg.cn/large/ece1c17dgy1fn4y9x6vkkj20ko0g174y.jpg)

<!--more-->

### 直接插入排序
#### 基本思想
将数组中的所有元素依次跟前面已经排好的元素相比较，如果选择的元素比已排序的元素小，则交换，直到全部元素都比较过为止。
#### 代码实现
```java
    /**
     * 插入排序
     * <p>
     * 1. 从第一个元素开始，该元素可以认为已经被排序
     * 2. 取出下一个元素，在已经排序的元素序列中从后向前扫描
     * 3. 如果该元素（已排序）大于新元素，将该元素移到下一位置
     * 4. 重复步骤3，直到找到已排序的元素小于或者等于新元素的位置
     * 5. 将新元素插入到该位置后
     * 6. 重复步骤2~5
     *
     * @param array 待排序数组
     * @return int[]
     */
    public static int[] insertSort(int[] array) {
        for (int i = 1; i < array.length; i++) {
            int temp = array[i];
            int j = i - 1;
            for (; j >= 0 && array[j] > temp; j--) {
                array[j + 1] = array[j];
            }
            array[j + 1] = temp;
        }
        return array;
    }
```
#### 复杂度

平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度
---|---|---|---
O(n²) |	O(n²)|	O(n²)|	O(1)

由于直接插入排序每次只移动一个元素的位， 并不会改变值相同的元素之间的排序， 因此它是一种稳定排序。

### 希尔排序
#### 基本思想
该方法的基本思想是：先将整个待排元素序列分割成若干个子序列（由相隔某个“增量”的元素组成的）分别进行直接插入排序，然后依次缩减增量再进行排序，待整个序列中的元素基本有序（增量足够小）时，再对全体元素进行一次直接插入排序。因为直接插入排序在元素基本有序的情况下（接近最好情况），效率是很高的。
#### 代码实现
```java
    /**
     * shell sort
     * <p>
     * 1. 选择一个增量序列t1，t2，…，tk，其中ti>tj，tk=1；（一般初次取数组半长，之后每次再减半，直到增量为1）
     * 2. 按增量序列个数k，对序列进行k 趟排序；
     * 3. 每趟排序，根据对应的增量ti，将待排序列分割成若干长度为m 的子序列，分别对各子表进行直接插入排序。
     * 仅增量因子为1 时，整个序列作为一个表来处理，表长度即为整个序列的长度。
     *
     * @param array array
     * @return int[] 排序后的array
     */
    public static int[] shellSort(int[] array) {
        int len = array.length;
        if (len < 2) {
            return array;
        }
        // 分组
        for (int gap = len / 2; gap != 0; gap /= 2) {
            for (int i = 0; i < gap; i++) {
                // 对组进行插入排序
                for (int j = i + gap; j < len; j += gap) {
                    // 组第一个元素
                    int index = j - gap;
                    // 找到当前组的例外一个元素
                    int temp = array[j];
                    // 进行比较
                    for (; index >= 0 && temp < array[index]; index -= gap) {
                        // 交换
                        array[index + gap] = array[index];
                    }
                    array[index + gap] = temp;
                }
            }
        }
        return array;
    }
```
#### 复杂度
平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度
---|---|---|---
$O(nlog2 n)$ |	$O(nlog2 n)$ |	$O(nlog2 n)$ |	$O(1)$

由于交换位置了，希尔排序是一种不稳定排序。

### 简单选择排序
#### 基本思想
1. 在要排序的一组数中，选出最小的一个数与第一个位置的数交换；
2. 然后在剩下的数当中再找最小的与第二个位置的数交换
3. 回到步骤1，如此循环直到倒数第二个数和最后一个数比较为止。

#### 代码实现
```java
    /**
     * 选择排序，
     * 直接选择排序
     *
     * 1. 在要排序的一组数中，选出最小的一个数与第一个位置的数交换；
     * 2. 然后在剩下的数当中再找最小的与第二个位置的数交换
     * 3. 回到步骤1，如此循环直到倒数第二个数和最后一个数比较为止。
     *
     * @param array 需要排序的数组
     * @return int[] 排序好的数组
     */
    public static int[] selectionSort(int[] array) {
        int len = array.length;
        for (int i = 0; i < len; i++) {
            // 记下最终交换的索引
            int position = i;
            // 记下当前轮次最小值
            int temp = array[i];
            for (int j = i + 1; j < len; j++) {
                if (temp > array[j]) {
                    temp = array[j];
                    position = j;
                }
            }
            if (position > i) {
                // 交换
                array[position] = array[i];
                array[i] = temp;
            }
        }
        return array;
    }
```
#### 复杂度
平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度
---|---|---|---
O(n²)|	O(n²)|	O(n²)|	O(1)

无论是哪种情况，哪怕原数组已排序完成，它也将花费将近n²/2次遍历来确认一遍。并且它是一种不稳定排序。

### 堆排序
#### 基本思想
堆排序是一种树形选择排序，是对直接选择排序的有效改进。
此处以大顶堆为例，堆排序的过程就是将待排序的序列构造成一个堆，选出堆中最大的移走，再把剩余的元素调整成堆，找出最大的再移走，重复直至有序。

![](https://ws1.sinaimg.cn/large/ece1c17dgy1fn7zcyqz9jg20f70a4u0x.jpg)

1. 建堆；
2. 堆顶与堆的最后一个元素交换位置，并且把最大值从堆删除；
3. 重复到最后一个节点，删除最大一个，结束

#### 代码实现
```java
    /**
     * 堆排序
     *
     * @param array array
     * @return int[]
     */
    public static int[] heapSort(int[] array) {
        for (int i = array.length; i > 0; i--) {
            //  每次交换后重新构造堆
            maxHeapify(i, array);
            // 将堆顶和堆的最后一个数值进行交换，
            swap(0, i - 1, array);
        }
        return array;
    }

    /**
     * 初始化大顶堆，使得子节点永远小于父节点，并且记录最大值，然后将堆顶和堆的最后一个数值进行交换，
     * 对于堆节点的访问：
     *      父节点i的左子节点在位置：(2*i+1);
     *      父节点i的右子节点在位置：(2*i+2);
     *      子节点i的父节点在位置：floor((i-1)/2);
     *
     * @param index index
     * @param array array
     */
    private static void maxHeapify(int index, @NotNull int[] array) {
        int parentIdx = index >> 1;
        for (; parentIdx >= 0; parentIdx--) {
            if (parentIdx << 1 >= index) {
                continue;
            }
            //左子节点索引
            int left = parentIdx * 2;
            //右子节点索引，如果没有右节点，默认为左节点索引
            int right = (left + 1) >= index ? left : (left + 1);
            // 找到较大的子节点
            int maxChildId = array[left] >= array[right] ? left : right;
            //如果子节点中的较大值比父节点大，交换父节点与左右子节点中的最大值
            if (array[maxChildId] > array[parentIdx]) {
                swap(maxChildId, parentIdx, array);
            }
        }
    }
    
    /**
     * 交换
     *
     * @param i     索引1
     * @param j     索引2
     * @param array array
     */
    private static void swap(int i, int j, @NotNull int[] array) {
        int temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }
```

#### 复杂度
平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度
---|---|---|---
$O(n \log_{2}n)$	|	$O(n \log_{2}n)$	|	$O(n \log_{2}n)$	|	$O(1)$

顺序打乱，它是不稳定排序。

### 冒泡排序
#### 基本思想
重复地走访过要排序的数列，一次比较两个元素，如果他们的顺序错误就把他们交换过来。走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。这个算法的名字由来是因为越小的元素会经由交换慢慢“浮”到数列的顶端。
#### 代码实现
```java
    /**
     * 冒泡排序
     * 将较小的数一直往前面冒
     * 
     * @param array array
     * @return int[]
     */
    public static int[] bubbleSort(@NotNull int[] array) {
        int len = array.length;
        for (int i = 0; i < len - 1; i++) {
            for (int j = i + 1; j < len; j++) {
                if (array[i] > array[j]) {
                    swap(i, j, array);
                }
            }
        }
        return array;
    }
```
#### 复杂度
平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度
---|---|---|---
O(n²) |	O(n)|	O(n²)|	O(1)

不会改变相同元素直接的位置，所以它是稳定的排序算法。
### 快速排序
#### 基本思想
快速排序的基本思想：**挖坑填数+分治法**。  
快速排序采用“分而治之、各个击破”的观念，

#### 代码实现
```java
    /**
     * 快速排序
     *
     * @param array array
     * @return int[]
     */
    public static int[] quickSort(@NotNull int[] array) {
        quickSort(array, 0, array.length - 1);
        return array;
    }

    /**
     * 快速排序 》递归
     * 递归满足条件：
     *      low < high
     * @param array array
     * @param low 最小边界
     * @param high 最大边界
     */
    private static void quickSort(@NotNull int[] array, int low, int high) {
        if (low >= high) {
            return;
        }
        int pivot = getPivot(array, low, high);
        quickSort(array, low, pivot - 1);
        quickSort(array, pivot + 1, high);
    }

    /**
     * 返回基准的索引
     * 1. i = low;L = low; R = high 将基准数挖出形成第一个坑a[i]。
     * 2. R-- 向前找到比基准a[i]小的值，找到后挖出此数填前一个坑a[i]中。
     * 3. L++ 向后找比基准大的数，找到后也挖出此数填到前一个坑a[j]中。
     * 4. 重复 2 、 3 步操作，知道 L == R ，返回基准的索引。
     * 
     * @param array array
     * @param low 最小边界
     * @param high 最大边界
     * @return 索引
     */
    private static int getPivot(@NotNull int[] array, int low, int high) {
        // 定义一个基准
        int pivotValue = array[low];
        while (low < high) {
            // 找到比基准小的放在左边
            while (low < high && array[high] >= pivotValue) {
                high--;
            }
            array[low] = array[high];
            // 找到比基准大的放在右边
            while (low < high && array[low] <= pivotValue) {
                low++;
            }
            array[high] = array[low];
        }
        // 找到基准的索引位置，并且返回
        array[low] = pivotValue;
        return low;
    }
```
#### 复杂度
平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度
---|---|---|---
$O(n \log_{2}n)$	|	$O(n \log_{2}n)$	|	$O(n^2)$	|	$O(1)$

>快速排序是通常被认为在同数量级（$O(n \log_{2}n)$）的排序方法中平均性能最好的。但若初始序列按关键码有序或基本有序时，快排序反而蜕化为冒泡排序。为改进之，通常以“三者取中法”来选取基准记录，即将排序区间的两个端点与中点三个记录关键码居中的调整为支点记录。快速排序是一个不稳定的排序方法。

>快速排序排序效率非常高。 虽然它运行最糟糕时将达到O(n²)的时间复杂度, 但通常平均来看, 它的时间复杂为O(nlogn), 比同样为O(nlogn)时间复杂度的归并排序还要快. 快速排序似乎更偏爱乱序的数列, 越是乱序的数列, 它相比其他排序而言, 相对效率更高.

### 归并排序
#### 基本思想
归并排序算法是将两个（或两个以上）有序表合并成一个新的有序表，即把待排序序列分为若干个子序列，每个子序列是有序的。然后再把有序子序列合并为整体有序序列。
#### 代码实现
```java
    /**
     * 归并排序
     *
     * 1. 将序列每相邻两个数字进行归并操作，形成 floor(n/2)个序列，排序后每个序列包含两个元素；
     * 2. 将上述序列再次归并，形成 floor(n/4)个序列，每个序列包含四个元素；
     * 3. 重复步骤2，直到所有元素排序完毕。
     *
     * @param array array
     * @return int[]
     */
    public static int[] mergingSort(int[] array) {
        // 拆分到最小单元的时候终止递归操作。
        if (array.length <= 1) {
            return array;
        }
        // 每次拆分的长度
        int segmentLen = array.length >> 1;
        int[] leftArr = Arrays.copyOfRange(array, 0, segmentLen);
        int[] rightArr = Arrays.copyOfRange(array, segmentLen, array.length);
        // 拆分成最小单元排序后，合并
        return merge(mergingSort(leftArr), mergingSort(rightArr));
    }

    /**
     * 合并左右两个数组
     *
     * @param arr1 数组1
     * @param arr2 数组2
     * @return int[]
     */
    private static int[] merge(int[] arr1, int[] arr2) {
        int i = 0, j = 0, k = 0;
        // 创建一个新数组
        int[] result = new int[arr1.length + arr2.length];
        // 将两个数据中的元素进行一一比较，较小的放入新的数组中
        while (i < arr1.length && j < arr2.length) {
            if (arr1[i] <= arr2[j]) {
                result[k++] = arr1[i++];
            } else {
                result[k++] = arr2[j++];
            }
        }
        // 经过上一步骤后，可能有一个数组的数没有放完，继续在后面添加上
        while (i < arr1.length) {
            result[k++] = arr1[i++];
        }
        while (j < arr2.length) {
            result[k++] = arr2[j++];
        }
        return result;
    }
```
#### 复杂度
平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度
---|---|---|---
O(nlog₂n)|	O(nlog₂n)|	O(nlog₂n)|	O(n)

>和选择排序一样，归并排序的性能不受输入数据的影响，但表现比选择排序好的多，因为始终都是O(n log n）的时间复杂度。代价是需要额外的内存空间。
 度为n的数组, 最终会调用mergeSort函数2n-1次。通过自上而下的递归实现的归并排序, 将存在堆栈溢出的风险。

### 基数排序
#### 基本思想
它是这样实现的：将所有待比较数值（正整数）统一为同样的数位长度，数位较短的数前面补零。然后，从最低位开始，依次进行一次排序。这样从最低位排序一直到最高位排序完成以后，数列就变成一个有序序列。

基数排序按照优先从高位或低位来排序有两种实现方案：

`MSD`[^1] : 从最左侧高位开始进行排序。先按k1排序分组, 同一组中记录, 关键码k1相等, 再对各组按k2排序分成子组, 之后, 对后面的关键码继续这样的排序分组, 直到按最次位关键码kd对各子组排序后. 再将各组连接起来, 便得到一个有序序列。MSD方式适用于位数多的序列。

`LSD`[^2] :从最右侧低位开始进行排序。先从kd开始排序，再对kd-1进行排序，依次重复，直到对k1排序后便得到一个有序序列。LSD方式适用于位数少的序列。

#### 代码实现
```java
    /**
     * 基数排序（LSD 从低位开始）
     * <p>
     * 基数排序适用于：
     * (1)数据范围较小，建议在小于1000
     * (2)每个数值都要大于等于0
     * <p>
     * ①. 取得数组中的最大数，并取得位数；
     * ②. arr为原始数组，从最低位开始取每个位组成radix数组；
     * ③. 对radix进行计数排序（利用计数排序适用于小范围数的特点）；
     *
     * @param arr 待排序数组
     * @return int[] 排序好的数组
     */
    public static int[] radixSort(@NotNull int[] arr) {
        int len = arr.length;
        if (len <= 1) {
            return arr;
        }
        //取得数组中的最大数，并取得位数
        int max = 0;
        for (int anArr : arr) {
            if (max < anArr) {
                max = anArr;
            }
        }
        int maxDigit = 1;
        while (max / 10 > 0) {
            maxDigit++;
            max = max / 10;
        }
        // System.out.println("maxDigit: " + maxDigit);
        //申请一个桶空间
        int[][] buckets = new int[10][arr.length];
        int base = 10;
        //从低位到高位，对每一位遍历，将所有元素分配到桶中
        for (int i = 0; i < maxDigit; i++) {
            //存储各个桶中存储元素的数量
            int[] bktLen = new int[10];
            //分配：将所有元素分配到桶中
            for (int j = 0; j < arr.length; j++) {
                int whichBucket = (arr[j] % base) / (base / 10);
                buckets[whichBucket][bktLen[whichBucket]] = arr[j];
                bktLen[whichBucket]++;
            }
            //收集：将不同桶里数据挨个捞出来,为下一轮高位排序做准备,由于靠近桶底的元素排名靠前,因此从桶底先捞
            int k = 0;
            for (int b = 0; b < buckets.length; b++) {
                for (int p = 0; p < bktLen[b]; p++) {
                    arr[k++] = buckets[b][p];
                }
            }
            base *= 10;
        }
        return arr;
    }
```
#### 复杂度
平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度
---|---|---|---
O(d*(n+r))|	O(d*(n+r))|	O(d*(n+r))|	O(n+r)

其中，d 为位数，r 为基数，n 为原数组个数。在基数排序中，因为没有比较操作，所以在复杂上，最好的情况与最坏的情况在时间上是一致的，均为 `O(d*(n + r))`。

基数排序更适合用于对时间, 字符串等这些整体权值未知的数据进行排序。

Tips: 基数排序不改变相同元素之间的相对顺序，因此它是稳定的排序算法。

### 总结
* 当原表有序或基本有序时，直接插入排序和冒泡排序将大大减少比较次数和移动记录的次数，时间复杂度可降至O（n）；
* 而快速排序则相反，当原表基本有序时，将蜕化为冒泡排序，时间复杂度提高为O（n2）；
* 原表是否有序，对简单选择排序、堆排序、归并排序和基数排序的时间复杂度影响不大。

![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/sort/常用排序算法.png)

### 参考博客
* [八大排序算法总结与java实现](https://itimetraveler.github.io/2017/07/18/%E5%85%AB%E5%A4%A7%E6%8E%92%E5%BA%8F%E7%AE%97%E6%B3%95%E6%80%BB%E7%BB%93%E4%B8%8Ejava%E5%AE%9E%E7%8E%B0/)

[^1]: Most significant digital
[^2]: Least significant digital



