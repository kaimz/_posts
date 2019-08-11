---
title: 使用 Fork Join 框架
date: 2018-03-21 23:11:03
tags: [java]
categories: 学习笔记
---

### 前言
当需要执行大量的小任务的时候，我们需要将多个小任务进行拆分，类似 `快速排序` 的 `分而治之`。

`Fork` 将一个大任务进行递归，不断的把它切割成符合条件的小任务，然后这些子任务分配到不同 CPU 核心上去分别计算，提高效率，`Join` 是 获取到小任务的计算结果，最后合并返回。

它充分利用了现在多核 CPU 的性能。 



<!--more-->

### 正文
`Fork/Join`框架的核心类是`ForkJoinPool`，它能够接收一个`ForkJoinTask`，并得到计算结果。`ForkJoinTask`有两个子类，`RecursiveTask`（有返回值）和`RecursiveAction`（无返回结果），我们自己定义任务时，只需选择这两个类继承即可。

#### RecursiveAction
```java
// 没有返回值的 fork / join 任务框架
public class PrintTask extends RecursiveAction {
    private static final int THRESHOLD = 5;
    private int start;
    private int end;

    PrintTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    public static void main(String[] args) {
        PrintTask task = new PrintTask(0, 25);
        // 分配四个线程给它
        ForkJoinPool pool = new ForkJoinPool(4);
        pool.execute(task);
        pool.shutdown();
    }

    @Override
    protected void compute() {
        if (THRESHOLD >= (end - start)) {
            // 满足小任务条件，分配打印任务
            for (int i = start; i < end; i++) {
                System.out.println(Thread.currentThread().getName() + ": " + i);
            }
        } else {
            // 任务还能继续拆分
            int division = (start + end) >> 1;
            PrintTask task1 = new PrintTask(start, division);
            PrintTask task2 = new PrintTask(division, end);
            invokeAll(task1, task2);
        }
    }
}
```

#### RecursiveTask<T>
```java
// 有返回值的 fork / join 任务框架 RecursiveTask<T>
public class SumTask extends RecursiveTask<Integer> {
    private static final int THRESHOLD = 50;
    private int start;
    private int end;

    public SumTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    public static void main(String[] args) throws InterruptedException {
        final int works = 200;
        SumTask task = new SumTask(0, works);
        ForkJoinPool forkJoinPool = new ForkJoinPool(4);
        long beginTime = System.currentTimeMillis();
        int result = forkJoinPool.invoke(task);
        long consumeTime = System.currentTimeMillis() - beginTime;
        System.out.println("Fork Join calculated the result is " + result + ",consume " + consumeTime + "ms");
        forkJoinPool.shutdown();

        result = 0;
        beginTime = System.currentTimeMillis();
        for (int i = 0; i < works; i++) {
            TimeUnit.SECONDS.sleep(1);
            result += i;
        }
        consumeTime = System.currentTimeMillis() - beginTime;
        System.out.println("The correct result is " + result + ",consume " + consumeTime + "ms");
    }

    @Override
    protected Integer compute() {
        int sum = 0;
        if (THRESHOLD >= (end - start)) {
            for (int i = start; i < end; i++) {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                sum += i;
            }
            System.out.println(Thread.currentThread().getName() + " result is ：" + sum);
            return sum;
        } else {
            int division = (start + end) >> 1;
            SumTask task1 = new SumTask(start, division);
            SumTask task2 = new SumTask(division, end);
            invokeAll(task1, task2);
            int result1 = task1.join();
            int result2 = task2.join();
            return result1 + result2;
        }
    }
}
```
* 结果：
```java
ForkJoinPool-1-worker-1 result is ：1225
ForkJoinPool-1-worker-0 result is ：8725
ForkJoinPool-1-worker-3 result is ：3725
ForkJoinPool-1-worker-2 result is ：6225
Fork Join calculated the result is 19900,consume 50018ms
The correct result is 19900,consume 200053ms
```

### 总结
1. 我们虽然可以通过调整阈值 `THRESHOLD` 控制子任务的大小，从而控制了任务的数量，但是我们分配的 `Fork/Join Pool` 数量却是根据 CPU 性能而定的，所以，切割任务的大小和数量需要进行预先计算好，不是子任务越多就越好，而且合并结果集，需要消耗其他的计算性能。如果任务大小不能控制，可以设计可伸缩算法，动态来计算出合理的阈值，以符合要求。
2. 注意正确的 `Fork/Join` 框架的写法，通过廖老师的文章 [Java的Fork/Join任务，你写对了吗？](https://www.liaoxuefeng.com/article/001493522711597674607c7f4f346628a76145477e2ff82000) ，指出了错误的写法的问题所在。

[源码地址](https://github.com/kaimz/learning-code/tree/master/fork-join)
