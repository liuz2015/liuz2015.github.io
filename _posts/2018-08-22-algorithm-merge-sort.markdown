---
layout:     post
title:      "搞懂归并排序"
subtitle:   ""
date:       2018-08-22
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 算法
    - 排序
    - 归并排序
---

## 目录
- [简介](#简介)
- [算法实现](#算法实现)
- [性能分析](#性能分析)
- [算法优化](#算法优化)
- [总结](#总结)
- [参考资料](#参考资料)

## 简介

归并排序（MERGE-SORT）是建立在**归并**操作上的一种有效的排序算法，该算法是采用分治法（Divide and Conquer）的一个非常典型的应用。归并排序的**运行时间为O(nlogn)**，而且是一种**稳定**的算法，不过，在排序过程中，归并排序需要使用的额外空间与n成正比。

## 算法实现

归并排序在算法实现上使用了分治思想。

基本做法是：将已经有序的子数组合并，就可以得到有序的完整数组，这个操作就称为**归并**。为了使得子数组有序，我们可以递归地进行同样的操作。

实现归并的办法是将两个有序数组归并到第三个数组中：先创建一个适当大小的辅助数组，然后将两个输入数组中的元素一个个从小到大的放入这个数组中。不过，如果在每次归并中都去创建这样一个数组，当数据量很大、归并次数很多时就会有问题，因此我们希望有一种方式，不需要在每次归并时额外创建数组；我们创建和原始数组一样大的辅助数组，使用下标来标志归并的范围，把归并范围内的元素放到辅助数组中，归并的结果再放回原数组中。

代码如下：

```java
public class MergeSort<T extends Comparable<T>> {

    protected void merge(T[] nums, int l, int m, int h) {

        int i = l, j = m + 1;

        for (int k = l; k <= h; k++) {
            aux[k] = nums[k]; // 将数据复制到辅助数组
        }

        for (int k = l; k <= h; k++) {
            if (i > m) {
                nums[k] = aux[j++];

            } else if (j > h) {
                nums[k] = aux[i++];

            } else if (aux[i].compareTo(nums[j]) <= 0) {
                nums[k] = aux[i++]; // 先进行这一步，保证稳定性

            } else {
                nums[k] = aux[j++];
            }
        }
    }
}
```

归并排序代码如下，这是自顶向下的实现方式：

```java
public class MergeSort<T extends Comparable<T>> {

    private T[] aux;//辅助数组

    public void sort(T[] nums) {
        aux = (T[]) new Comparable[nums.length];
        sort(nums, 0, nums.length - 1);
    }

    private void sort(T[] nums, int l, int h) {
        if (h <= l) {
            return;
        }
        int mid = l + (h - l) / 2;//数组对半分成两个子数组
        sort(nums, l, mid);//使左子数组有序
        sort(nums, mid + 1, h);//使右子数组有序
        merge(nums, l, mid, h);//归并
    }
}
```

前面的自顶向下的方式，是通过不断将大数组分成小数组；我们还有一种自底向上的实现方式，先排序小数组（最小的数组就是只有一个元素），然后成对地合并成大数组，这种方式的优点是没有递归的操作。

代码如下：

```java
public class MergeSortBU<T extends Comparable<T>> {
    
    private T[] aux;//辅助数组

    public void sort(T[] nums) {

        int N = nums.length;
        aux = (T[]) new Comparable[N];

        for (int sz = 1; sz < N; sz += sz) {//sz:子数组大小
            for (int lo = 0; lo < N - sz; lo += sz + sz) {
                merge(nums, lo, lo + sz - 1, Math.min(lo + sz + sz - 1, N - 1));
            }
        }
    }
}

```

## 性能分析

归并排序每次会将数组一分为二，所以运行时间的递归式为：T(n)=2T(n/2)+O(n)；递归的层数为lgn层，每层运行时间为O(n)，因此归并排序的**运行时间就为O(nlgn)**。

*注：递归层数计算如下：*

*假设层数为x层，每次数组都被切分为n/2和n/2两个子数组，当递归了x层后，n/2数组大小变为1，则停止递归，可以列出方程为：(n/2)<sup>x</sup>=1；解得层数x=lgn。*

## 算法优化

### 对小规模子数组改变排序算法

因为递归会使小规模问题中方法的调用过于频繁而导致性能问题，因此，对于小规模的数组，可以改用插入排序等。

### 测试数组是否已经有序

添加一个判断条件，如果a[m]小于等于a[m+1]，我们就认为数组已经有序，并跳过merge()方法。

### 不将元素复制到辅助数组

将元素从一个数组复制到另一个数组是耗时的操作，如果能够省去可以节约很多时间（但是辅助数组的空间是无法节约的）；我们可以在每次递归调用层交换辅助数组和原始数组的角色，即对原数组归并将结果存入到辅助数组，或者对辅助数组归并将结果存入原数组。

## 总结

归并排序提供了一种**运行时间为O(nlgn)的、稳定的**排序解决方案，不过其所需额外的空间的确是缺陷；不过，其中提到的**归并操作**其实并不只用在归并排序中，在其他一些问题中都可以借鉴。

## 参考资料
- 《算法》
- [归并排序_百度百科](https://baike.baidu.com/item/%E5%BD%92%E5%B9%B6%E6%8E%92%E5%BA%8F/1639015)

