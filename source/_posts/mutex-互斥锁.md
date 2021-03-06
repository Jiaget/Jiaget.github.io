---
title: mutex 互斥锁
date: 2021-06-07 23:16:47
tags: 
    - golang
    - 锁
    - mutex
categories: golang
---
# mutex
```
type Mutex struct {
    state int32
    sema uint32
}
```
- state:
    - 代表锁的状态
    - mutex的加锁与解锁都是通过 `atomic` 包提供的函数原子性地操作该字段实现的
- sema:
    - 一个信号量
    - 用作等待队列

## mutex 的 正常模式

一个goroutine 会先自旋几次，尝试获得锁，如果仍然没有获得锁，则通过信号量 `sema` 排队等待（进入等待队列）。

但是当锁被释放时，队首的goroutine也不会立即获得锁，而是与其他后来的自旋的goroutine竞争。一般情况下，处于自旋状态的goroutine会更先抢到锁，因为它们一直处于运行状态，而队列里出来的goroutine由于刚唤醒，还会因为需要分配cpu资源而需要消耗一点时间。这种情况下，为了保证先进先出，即使竞争失败了，仍然插入队列头部而不是尾部。

## mutex的饥饿模式

当一个goroutine本次加锁等待时间超过1ms后，它会将mutex从 `正常模式`切换成 `饥饿模式` 。

在饥饿模式下，锁一旦被释放，就会直接传递给等待队列的头部goroutine。而如果有一些后来的goroutine需要这个锁，它们不会像正常模式一样自旋，而是直接被加入到队列尾部。

`饥饿模式` 也可以再切换回正常模式：
- 当一个goroutine获得锁，且等待时间 `< 1ms`时。
- 当等待队列为空时 (即等待队列最后一个goroutine获得锁。)

## 两者对比

- 正常模式
    - 这是一个自旋和排队共存的模式
    - [优点] 由于自旋可以在无需唤醒的状态下直接获得锁，因此，正常模式可以保证Mutex有更好的。
    - [缺点] 但是, 这个模式可能导致队列尾端的goroutine一直抢不到锁（尾端延迟）

- 饥饿模式
    - 这个模式会强制所有goroutine排队
    - 它在牺牲一定性能的情况下， 解决正常模式的尾端延迟问题

两种模式之间的切换，既保证锁的效率，又能顾及公平性，不会导致某个goroutine过长时间的等待。