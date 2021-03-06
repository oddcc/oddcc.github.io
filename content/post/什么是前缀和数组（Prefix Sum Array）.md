---
title: "什么是前缀和数组（Prefix Sum Array）"
date: 2021-02-22T17:48:15+08:00
author: oddcc
type: post
url: /2021/02/什么是前缀和数组（Prefix Sum Array）/
categories:
  - Algorithm
---

## 遇到的问题

如果有数组`arr`

```
       0  1  2  3  4  5  6
arr -> 2  5  4  9  7  1  0
```

我们需要频繁的求任意子数组的元素的和，应该如何求解？最容易想到的就是循环求和，但这是个`O(N)`的操作。有没有什么办法可以让我们可以用`O(1)`的时间复杂度来得到子数组的和？

## 什么是前缀和数组

如果对于数组arr，我们有如下的数组preSum，满足

> preSum[0] = arr[0]
>
> preSum[1] = arr[0] + arr[1]
>
> preSum[2] = arr[0] + arr[1] + arr[2]
>
> ...
>
> preSum[n] = arr[0] + ... + arr[n]

那么preSum就是一个前缀和数组，实际应用中，我们可以通过`preSum[0] = arr[0]`和`preSum[i] = preSum[i - 1] + arr[i]`来方便的得到前缀和数组。

## 用法

回到最初的问题，如果我们要求[2,4]的子数组的和是多少，我们可以利用`preSum[4] - preSum[2 - 1] == 27 - 7 == 20`来方便的求得，从而把一个`O(N)`的操作转为`O(1)`的操作，而构造前缀和数组是一个`O(N)`的操作

```
          0  1  2  3  4  5  6
arr    -> 2  5  4  9  7  1  0
preSum -> 2  7 11 20 27 28 28
```

