---
title: "关于并查集（Disjoint Set Union或Union Find）"
date: 2021-03-02T15:40:14+08:00
draft: false
author: oddcc
type: post
url: /2021/03/关于并查集（Disjoint Set Union或Union Find）/
categories:
  - Algorithm
---

## 什么是并查集

并查集说的是这样一种数据结构，对外提供三个（一个属于可选）接口

- `add(x): void` （可选）表示添加一个元素`x`，这个元素`x`属于自己一个单独的子集

- `find(x): element` 表示在整个数据结构中找一个元素`x`，返回`x`所属子集的根（`root`，这里假设了子集是个树，有根节点，实际上这里也可以是子集的编号等等，总之是可以标识一个集合的信息）

- `union(x,y): void` 表示把`x`和`y`各自所在的子集进行合并

另外还可以方便的知道，整个集合中有几个子集。

这也就是`Disjoint Set Union`名称的由来，这种数据结构可以方便的处理不相交集（`Disjoint Set`，一系列没有重复元素的集合）的合并和查询的问题。

## 解决什么问题

上面说的有点抽象了，下面来看具体可以解决什么问题，比如现在有`[1...n]`共`n`个点，给你一些类似`(1,2)`这样的数据，表示`点1`和`点2`是相互连接的，属于同一个子集。要查询总共有多少个子集，或者随便给个点`m`，要查`m`属于哪一个子集。

这种问题，我们就可以考虑使用并查集来解决。

比如上面的例子，我们遍历所有的数据，先使用`add(x)`把所有的点建立一个集合，再遍历所有条件，使用`union(1,2)`来把`点1`和`点2`连接起来，最后只要调用`find(x)`就可以方便的得到结果。

## 并查集的实现

这里给出一个参考实现，使用一个数组来表示并查集

```java
private class UnionFind {
    private int[] parent; // 记录了属于哪个集合，如果有parent[i] == i，那么i就是一个子集的根
    private int count; // 记录了包含子集数量

    // 没实现添加操作，而是在构造方法中直接生成集合
    public UnionFind(int count) {
        this.count = count;
        int[] parent = new int[count];
        for (int i = 0; i < count; i++) parent[i] = i;
        this.parent = parent;
    }

    public int getCount() {
        return count;
    }

    public int find(int x) {
        while (x != parent[x]) {
            parent[x] = parent[parent[x]]; // 路径压缩（path compression），后面细说
            x = parent[x];
        }
        return x;
    }

    public void union(int x, int y) {
        int rootX = find(x);
        int rootY = find(y);
        if (rootX == rootY) return;
        if (rootX > rootY) {
            parent[rootX] = rootY;
        }
        else {
            parent[rootY] = rootX;
        }
        // 合并后子集数量要减1
        count--;
    }
}
```

以上是个简单的并查集实现，代码逻辑不难，重点说说路径压缩（path compression）。因为我们在`int[] parent`中记录的是父节点的信息，所以实际上查询的时候，实际上是从树的叶子节点往根节点查，类似于在遍历一个链表，当树的高度很大，或链表很长时，查询效率是很低的（`O(n)`）。

一个简单的办法就是，如果发现我的父节点不是根节点，那么就让父节点的父节点直接变成我的父节点，这样一来，层级就少了一层。最后的结果就是，除了根节点之外，一个子集中的所有节点都直接连接到根节点上，查询变成了`O(1)`的复杂度。这是并查集能高效进行查询的关键。

