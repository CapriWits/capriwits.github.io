---
title: "Java 左值和右值的思考"
slug: "Thoughts-on-Java-lvalues-and-rvalues"
date: 2024-06-02T18:57:02+08:00
author: CapriWits
image: java-programming-cover.png
tags:
  - Java
  - ByteCode
categories:
  - Backend
hidden: false
math: true
comments: true
draft: false
---

<!-- TOC -->

* [Java 左值和右值的思考](#java-左值和右值的思考)
    * [前言](#前言)
    * [问题复现](#问题复现)
    * [字节码分析](#字节码分析)
    * [其他实验](#其他实验)
    * [Summary](#summary)
    * [🔗Reference](#reference)

<!-- TOC -->

# Java 左值和右值的思考

![JDK21](https://img.shields.io/badge/JDK-21-red)

## 前言

刷[算法题](https://leetcode-cn.com/problems/maximize-sum-of-array-after-k-negations/)，用到小根堆「PriorityQueue」，
其中一个操作让我困惑了很久

小根堆存储的是原数组为负值的下标，则小根堆堆顶为最小负数的下标

本意是循环中，让最小负数取到相反数 `k` 次，变成一个正数

`while (k-- > 0) nums[queue.peek()] = -nums[queue.poll()];`

这一操作让我疑惑了很久，根据赋值表达式的性质，应该是从右计算到左，但是如果按照这个逻辑，就会让 `poll()`
先进行，后面才 `peek()` ，
这样下标的计算就出错了, 即 `peek()` 实际取的下标值已经被 `poll()` 出堆了

## 问题复现

```java
public static void main(String[] args) {
    int[] nums = {2, 1, 0};
    PriorityQueue<Integer> pq = new PriorityQueue<>(List.of(0, 1, 2));
    nums[pq.peek()] = -nums[pq.poll()];
    System.out.println(Arrays.toString(nums));  // [-2, 1, 0]
}
```

小根存存储原数组下标，此时堆顶为 `0`

`nums[pq.peek()] = -nums[pq.poll()]` 实际执行运行时状态是先执行 `peek()`，再执行 `poll()`，即从左到右，与赋值运算符 `=`
右值赋值给左值不同

即 `nums[0] = -nums[0]` 取反

## 字节码分析

使用 `javap -c Solution.class` 反编译字节码分析

```
32: invokestatic  #15                 // InterfaceMethod java/util/List.of:(Ljava/lang/Object;Ljava/lang/Object;Ljava/lang/Object;)Ljava/util/List;
35: invokespecial #21                 // Method java/util/PriorityQueue."<init>":(Ljava/util/Collection;)V
38: astore_2
39: aload_1
40: aload_2
41: invokevirtual #24                 // Method java/util/PriorityQueue.peek:()Ljava/lang/Object;
44: checkcast     #10                 // class java/lang/Integer
47: invokevirtual #28                 // Method java/lang/Integer.intValue:()I
50: aload_1
51: aload_2
52: invokevirtual #32                 // Method java/util/PriorityQueue.poll:()Ljava/lang/Object;
55: checkcast     #10                 // class java/lang/Integer
58: invokevirtual #28                 // Method java/lang/Integer.intValue:()I
```

先是堆的初始化，然后先执行 `peek()` 再执行 `poll()`, 即从左到右的顺序编译

## 其他实验

```java
public static void main(String[] args) {
    int[] nums = {4, 5, 6};
    nums[getIndex()] = -nums[getValue()]; // Should call getIndex first, then getValue
    System.out.println(Arrays.toString(nums)); // [4, -6, 6]
}

public static int getIndex() {
    System.out.println("getIndex called");
    return 1;
}

public static int getValue() {
    System.out.println("getValue called");
    return 2;
}
```

> getIndex called  
> getValue called  
> [4, -6, 6]

```
18: invokestatic  #7                  // Method getIndex:()I
21: aload_1
22: invokestatic  #13                 // Method getValue:()I
```

同样，编译顺序也是从左至右

## Summary

- 不能按照赋值表达式 `=` 从右向左运行的思想思考 `nums[queue.peek()] = -nums[queue.poll()];`
- 最终的赋值的确会按照右向左执行「赋值操作」，完成 **置相反数** 的操作
- 但是从最终字节码的执行顺序来看，对于 **表达式** 的计算，会 **从左向右** 计算，将前序准备工作「取值」完成后，才进行写操作，赋值。

## 🔗Reference

[1005. K 次取反后最大化的数组和](https://leetcode-cn.com/problems/maximize-sum-of-array-after-k-negations/)
