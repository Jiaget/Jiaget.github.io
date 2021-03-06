---
title: GC(二)
date: 2021-06-18 19:12:55
tags:
    - gc
categories: golang
---

之前，主要讲了golang的三色标记法以及混合写屏障机制。这里简略介绍一下一些GC机制。

## 简述三色标记法

三色标记法是一种追踪式回收机制：
- 垃圾回收开始时：所有数据标记为白色
- 将可以追踪到的root节点标记为灰色（灰色表示由当前节点展开的追踪还未完成）
- 当该节点追踪任务完成，该节点标记为黑色，并无需再次基于它展开追踪
- 由黑色节点追踪到的节点会被标记为灰色。
- 当所有的灰色节点都被标记为黑色时，标记工作完成
- 白色节点均为垃圾

## 标记清除带来的问题

内存碎片化，回收的内存很难保证能成片聚在一起。而这些小块内存碎片会导致找出合适的内存空间进行分配的工作将付出更高的代价。甚至一些小块内存永远无法使用

### Bibop

这是一种管理碎片化内存的机制，它将大小规格相同的内存碎片统一管理。从而快速匹配内存。

当然这个机制无法解决小内存无法使用的问题

### 移动数据

在标记清除算法中的标记阶段，会移动非垃圾数据，使它们尽可能紧凑地放在内存中。
![move.png](https://i.loli.net/2021/06/18/kUBnZb9fpEO1veG.png)

缺点在于移动数据会带来不小的开销


### 复制式回收

将内存划分为两个相等的空间 `From`, `To`。 程序执行使用 `From`空间，在垃圾回收扫描时，将能追踪的数据复制到 `To` 空间。所有空间复制完成后，将`From` 和 `To` 空间交换，并清除原`From`空间的数据。

这种方法会占用一般的内存空间，为了提高内存使用率，只会在一部分的内存空间使用，并搭配其他的回收机制。

### 分代回收

### 弱分代假说

大部分对象都在年轻时死亡。

我们将只存活1， 2次回收的对象称为新生代对象。而经历过多次垃圾回收的对象为老年代对象。实践证明，新生代对象成为垃圾的概率要高于老年代对象。

我们可以将数据分成新生代和老年代，降低老年代数据的垃圾回收扫描频率，这将提升垃圾回收效率。

## 引用计数式垃圾回收

以上均为追踪式垃圾回收，引用计数式垃圾回收有所不同。

该垃圾回收在执行过程中，会更新对象的引用计数，当引用计数更新为0时，说明该数据不再有用，可以进行垃圾回收。这种回收机制可以将之前的追踪代价分摊到每次对数据的操作中。

但是，引用计数的均摊思路，如果在高频率开辟内存的场景下，仍然带来不小的开销。


# STW

每种垃圾回收策略都无法完美解决各式各样的缺陷。同样的，以上的垃圾回收策略均无可避免的需要 `Stop the World`， 这将会暂停用户程序，为了避免长时间 STW 带给用户影响，我们可以将 STW 分成多次执行，将一次垃圾回收拆分成多次，和用户程序交替执行。这叫做 `增量式回收`。

这可能带来的问题是，在某段垃圾回收标记了一个黑色对象，用户程序立马修改它，导致垃圾回收机制的误判，这里就得使用 `读写屏障` 机制，详见上一篇文章。

## 读屏障

补充一个读屏障。

在 `FROM` `TO` 复制回收机制中。

![fromti.png](https://i.loli.net/2021/06/18/LP5Fzi1HGnc96AB.png)

当A已经复制到 `to` 空间上后，用户程序读取了 `From` 空间的旧数据，B复制到了 A,由于B建立的指针指向了旧的A，当`FROM`空间被销毁，B指向的旧A也不复存在，此时再去访问旧A，就会出错。

此时需要引入一个读屏障：
- 当我们读取的数据有新副本时，直接读取 `To` 空间的新副本。

# 多核


多线程并行执行垃圾回收的场景，被称为 `并行垃圾回收`。

并发GC的问题在于，很容易导致有的协程的GC忙碌，而有的线程空闲。如果使用负载均衡，这也会带来同步的开销。

同时，并行GC还有的问题是，`FROM-TO`复制回收还需要避免数据对象被重复复制。


- 用户程序和GC并行执行的垃圾回收被称为 `并发垃圾回收`。

- 在GC的某个阶段进行STW，被称为 `主体并发式垃圾回收`

- 再增加增量垃圾回收机制，又会出现 `主题并发增量式垃圾回收`

![gc-type.png](https://i.loli.net/2021/06/18/GNYLQxF534jwTaU.png)


# Golang采用了哪些？

Golang 采用了 三色标记法 的 `标记-清除` 算法 + 主体并发增量式回收 + 插入，删除的混合写屏障
