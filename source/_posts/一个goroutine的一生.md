---
title: 一个goroutine的一生
date: 2021-06-24 13:25:09
tags:   
    - golang
    - goroutine
categories: golang
---

一个函数被编译成可执行文件以后，会通过 `程序执行入口` 进入内存虚拟地址空间中的代码段。

对于一个 main 函数而言， 程序会以 `runtime.main` 作为入口， 创建一个 `main goroutine`, 之后再去调用 `main.main` 函数。

提到 Golang 的协程运行，就不得不提 `GMP` 模型, 在虚拟地址空间的 `数据段` 会记录这样的全局变量：

- g0 （主协程）
    - g0 会在主线程栈上分配栈空间
    - g0 持有 m0 指针

- m0
    - m0 即主线程所对应的m
    - m0 持有 g0 的指针，且m0上运行的第一个g就是g0

- allgs: 记录所有的g

- allm: 记录所有的m

- allp: 在程序初始化的过程中，调度器会进行初始化，根据环境变量 GOMAXPROCS 来决定创建 p 的个数。
    - 同样， 第一个被创建的p，会和m0相关联。

- sched: 调度器
    - 记录所有空闲的 m 和 p
    - runq

- runq：runq作为全局变量，记录在 `sched` 下
    - 这里的runq是全局runq
    - 当然还有一个本地runq， 记录在P中。

## 本地队列与全局队列

runq， 在 `runtime.p` 中记录的是本地队列，在 `runtime.schedt` 中记录的是全局队列。
 
当本地队列满了，G就会放在全局队列中。 m 优先获取本地队列的 g， 再从全局队列中获取，当 本地和全局都没有 g ， 它会从其他 m 中分担一些 g。

## main 函数

回到main函数的执行过程中来。

main函数执行过程中需要创建的 main goroutine， 会被加入到 allp[0] 的本地队列中， 之后会通过 mstart 函数开启调度循环(schedule()),调度器将队列中的 g 调度到m0上去执行。

main goroutine 的工作不仅仅是调用 main 函数， 它还会开启线程监控， 包初始化等工作， 准备工作结束后，执行完 main 函数后， goroutine 便退出。
