---
title: "关于LRU（Latest Recently Used）算法"
date: 2021-03-01T10:20:09+08:00
draft: false
author: oddcc
type: post
url: /2021/03/关于LRU（Latest Recently Used）算法/
categories:
  - Algorithm
  - Redis
---

## 解决什么问题

想象我们有一个缓存系统，里面以`key-value`的形式存储了很多缓存数据，但存储空间总是有限的，如果我们不断的接受请求，那么总有存满的一天。当存储空间存满了之后，我们就要通过**某种方式**去释放空间，不然我们的缓存系统就无法接受新的请求了。

那么作为一个缓存系统，我们其实希望清理的是不再用到的数据，对于有很多请求的数据（热点），我们要留在系统中，这才是缓存系统该做的事情。那么如何判断不再用到呢？其实我们没有办法判断，因为客户端用哪些数据不是缓存系统能决定的，我们只能尽可能达到这个目标。

容易想到的有几种方案

1. FIFO，First In First Out，先来先服务，如果存储空间满了，就把最早存储的数据清理掉，对于大多数场景来说，这个策略可能不太合理，因为哪些是热点数据，用缓存的时间来判断不太靠谱
2. LIFO，Last In First Out，甚至比FIFO更加不合理。
3. LRU，Latest Recently Used，顾名思义，如果存储空间满了，我们要清理的是**最久没被使用**到的数据，这基于一个假设，就是越长时间没被用到的数据，之后被用到的概率也不会很大。显然这个策略是符合直觉的，比如微博的热搜，如果过去2天都没有人看了，时间过的越久，被人想起来再看看的概率就越低。
4. LFU，Least Frequently Used，本质上跟LRU是基于同一个假设，只不过角度不是时间，而是频率。清理的是被访问频率最低的数据。

其他还有很多算法，想要解决的都是类似的缓存淘汰问题。

下面我们来看主角——LRU算法

## LRU算法应该如何实现

显然，如果我们需要清理最久没被使用的数据，首先我们要知道哪些是最久没被使用的数据。

一个朴素的想法是，记录数据被访问的时间，在需要清理的时候，按访问时间排序，然后清理时间最小的数据。这个算法是可行的，但是需要额外`O(N)`的存储空间，以及维护有序所需要的时间，另外还要考虑到当一个值后续被访问时，我们还要去更新它的被访问时间。总之这是一个逻辑上可行（堆、数组等等），但是实际成本非常大的算法。

其实我们需要的快速知道最小的数据是哪个，同时需要能快速把某数据放到最大的位置上（因为只要被访问了，访问时间就是当前时间），通常会用哈希表 + 双向链表来实现

这里引用[知乎文章](https://zhuanlan.zhihu.com/p/34133067)的图片（改了其中的错误）来说明，其中`s`表示`set`（即新的缓存值），`g`表示`get`（即对某缓存值的访问），哈希表的部分图中没表示出来，后面再进行说明。

下图是一个容量为`3`的LRU数据结构，是由双向链表构成，我们维护一个`head`结点，一个`tail`结点，其中`head.next`表示最近被访问的数据，而`tail.pre`表示最久没被访问的数据。LRU算法需要淘汰的就是`tail.pre`的数据。

重点从第5步开始可以看到，当新的请求进来后，发现容器满了，就要把新的元素放在`head.next`，并把`tail.pre`删除（并同步更新对应的哈希表），代码可以参考[这里](https://github.com/oddcc/leetcode-practice/blob/master/src/main/java/com/oddcc/leetcode/editor/cn/LruCache.java)。这种实现可以用`O(1)`的时间复杂度以及`O(N)`的空间复杂度来实现LRU算法，其中N是容器的最大容量。

![image-20210301110912192](../../static/images/关于LRU（Latest%20Recently%20Used）算法/image-20210301110912192.png)

## 现实场景中的应用

在现实场景中，由于LRU算法就是用于缓存的淘汰，对效率和资源的要求非常高，实际上都往往没有使用上面的精准实现，比如Redis，使用的就是近似的LRU算法。

如文档里提到的，Redis是直接在池子里随机捞一批key，然后淘汰其中最久没被访问的数据，重复这个过程，直到释放了足够的空间。虽然不是完全精准，但是非常高效。

> Redis LRU algorithm is not an exact implementation. This means that Redis is not able to pick the *best candidate* for eviction, that is, the access that was accessed the most in the past. Instead it will try to run an approximation of the LRU algorithm, by sampling a small number of keys, and evicting the one that is the best (with the oldest access time) among the sampled keys.

这也提醒我们，理论是理论，工业界更注重实践，一个产品的最终设计往往是多方面权衡妥协的结果.