---
title: "什么是单调队列（Monotonic Queue）"
date: 2021-02-23T11:02:48+08:00
author: oddcc
type: post
url: /2021/02/什么是单调队列（Monotonic Queue）/
categories:
  - Algorithm
---

## 什么是单调队列（单调栈）

简单的说，就是栈内的元素，满足某种单调性，比如栈内的元素是单调递增的，那么`pop()`的时候，得到的就一定是当前栈内最大的元素。

单调栈可以用`O(1)`的复杂度获得当前栈内元素的最大（最小）值。在有新元素入栈时，破坏了单调性的元素会被出栈，且不会再次入栈。

比如我们在遍历数组的时候维护一个单调递增栈，新的元素`x`小于栈顶元素`t`，那么我们就不停的让栈顶元素出栈，直到`x`大于当前栈顶元素，然后把`x`入栈。单调栈的维护是个`O(N)`的操作。

在遍历的过程中，数组中的每一个元素只会入栈`1`次，出栈`1`次，所以整个遍历操作保持了`O(N)`的时间复杂度。

## 解决什么问题

还是刚才那个例子，在维护单调栈的过程中，对于出栈元素`t`来说，它找到了离它最近的小于`t`的元素`x`；对于新元素`x`来说，找到了左侧离它最近的小于`x`的元素。

如果一个问题跟前后元素之间的大小关系有关，我们需要高效寻找一个元素前后大于或小于它的元素，就可以考虑使用单调栈。

例如[739. Daily Temperatures](https://leetcode-cn.com/problems/daily-temperatures/)

题目要求就是高效的寻找第一个大于`temperatures[i]`的元素位置，用单调栈来解就很合适

又比如[84. Largest Rectangle in Histogram](https://leetcode-cn.com/problems/largest-rectangle-in-histogram/)

![img](../../static/images/什么是单调队列（Monotonic%20Queue）/histogram_area.png)

要确定矩形的宽和高，问题可以转化为找`height[i]`左边小于它的值和找`height[i]`右边小于它的值，如果找到了，那么高为`height[i]`的矩形的宽度就能确定了，也适合用单调栈来解