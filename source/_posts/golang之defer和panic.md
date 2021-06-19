---
title: golang之defer和panic
date: 2021-06-16 18:12:34
tags:
    - golang
categories: golang
---

# panic 和 defer 的一般执行流程

Golang的每一个goroutine上都会有 `*_defer`, `*_panic` 的头指针，每次发生新的panic， 都会在链表头上加入panic。defer与之类似。

![case1.png](https://i.loli.net/2021/06/16/Y84sWgIQvpVmxwG.png)

在这个场景下，当程序执行到panic时，goroutine的defer链表已经储存了两个defer函数。此时，程序会先将panic储存到panic链表中，再从defer链表中取出defer函数执行。

![link.png](https://i.loli.net/2021/06/16/QYjncWOBK65s4gx.png)

由panic触发的defer，在执行中有所不同。先看下defer的结构：

![_defer.png](https://i.loli.net/2021/06/16/NT9a4d5yDlSpOqh.png)

- 先将 `started` 置 `true`
- 将 `_panic` 指向当前panic

这种设计，是为了应对defer函数的执行失败。比如：

![case2.png](https://i.loli.net/2021/06/16/5CMwiJmxZHaELfS.png)

我们让A1也发生panic。当程序执行到 `A1` 的 `panic` 时, `goroutine` 的 `_panic`链表又会增加一个 `panic` 。

此时，新的 `panic` 也会触发执行 `_defer` 链表中的函数，并发现第一个defer函数上标记的 `panic` 不是自己，此时，程序会将 `_panic` 链表中对应的 `panic` 中止。

![panic-shutdown.png](https://i.loli.net/2021/06/16/gHTuS8jRka1L4eh.png)

此时，需要展示一下 `panic` 的结构体。

![panic-struct.png](https://i.loli.net/2021/06/16/EedOavxV5HyzwNk.png)

最后， 在没有 `recover` 的情况下，`panic` 打印会从链表尾部，打印至头部。

## 小结1

无 recover 的 panic 执行流程的要点在于：
- `panic` 执行 `defer` 函数会先标记defer函数，目的是嵌套的 `panic` 可以中止之前的 `panic`
- 所有的panic 都会输出，且顺序输出。

# recover

`recover` 只会把 `panic` 结构体上的 `recovered` 字段置 `true`。