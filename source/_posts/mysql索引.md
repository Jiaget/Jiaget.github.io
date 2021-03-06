---
title: mysql索引
date: 2021-06-29 10:00:31
tags: 
    - Mysql
categories: MySQL
---

# 什么是索引

按 `用户指定` 的字段，对数据进行 `排序` 的一种数据结构

# 为什么需要索引

数据存储在磁盘上，磁盘本身不具备比较功能。假如我们需要查询某一行的数据，在没有索引的情况下，数据库会一行一行从磁盘中读取数据，判断是否是用户需要的。

为了优化读取效率， 才出现了索引。

针对某一列建立一个索引后，数据会根据该列进行（默认 asc 升序） 排列，对于一组有序数据，可以使用二分查找，这样，会大大加快查询数据的效率。

# 数据结构的对比

- 哈希表
    - 通过哈希函数建立哈希索引，可以快速读取某一个数据
    - 缺点是没有办法查询某一个范围内的数据，因为哈希表是无序的。

- B+ 树
    - 叶子节点是一个索引，能快速超找到目标数据。
    - 一个节点可以存储多个数据，相对AVL树，存储等量数据的树高会低一些，查询效率也会增加。