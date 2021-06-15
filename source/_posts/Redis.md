---
title: Redis
date: 2021-06-01 19:18:37
tags:
    - nosql
    - 高并发
categories: nosql
---
# Redis 
## Redis 简介

Redis 全称 (REmote DIctionary Serer) 远程字典服务， 是一个键值对数据库。

与 NoSQL 相对应的 关系型数据库在当今环境存在的问题：
- 性能： 磁盘IO性能低
- 扩展： 数据之间的关系复杂，扩展性差，不便于大规模集群使用

NoSQL对其缺点进行了改进：
- 内存存储
- 不存数据关系，只存数据

## Redis 的特征

1. 数据之间没有必然联系
2. 采用单线程机制
3. 高性能。 
4. 多数据类型：
    - string
    - list
    - hash
    - set
    - sorted_set
- 持久化。数据灾难恢复。

## 应用场景

- 热点数据加速查询
- 任务队列
- 实时信息
- 时效信息
- 分布式数据功效。分布式集群中的session分离
- 消息队列
- 分布式锁

## Redis 的一些功能

- 缓存
    - 针对一些高并发业务的设计
    - 平台的高频率数据。
- 单服务器升级集群
- Session 管理
- Token

## Redis 命令

### 单值缓存

- 写 `set [key] [value]`
- 读 `get [key] ([value])` 。 如果取的值不存在，返回 `(nil)`

### 对象缓存

- SET user:1 value(json)
- MSET user:1:name xxx user:1:balance xxx

