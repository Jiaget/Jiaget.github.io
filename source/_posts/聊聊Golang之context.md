---
title: 聊聊Golang之context
date: 2021-06-05 12:51:06
tags:
    - golang
    - context
categories: golang
---
# context 解决的问题

在实际开发中，经常会出现一个goroutine创建多个goroutine的场景。
![goroutine.png](https://i.loli.net/2021/06/05/nrJzT1Si4PAHBxR.png)

context 用于解决多goroutine之前的通信问题，比如 G 可以控制它的子线程超时退出，取消，传递信息等。

当然，控制goroutine的方法有很多，最常见的就是通过 select + channel 实现超时与取消。

```
    done := make(chan struct{}, 1)
    go func() {
        // 发送请求等任务
        done <- struct{}{}
    }(ctx)

    select {
    case <-done:
        fmt.Println("call successfully!!!")
        return
    case <-time.After(time.Duration(800 * time.Millisecond)):
        fmt.Println("timeout!!!")
        return
    }
```

之所以还需要context,就是因为 简单的 select + channel 无法解决更复杂的场景：
正如图中协程G开启的协程g2，它也会开启g4, g5。 协程G希望给g2设置超时，取消机制，但如果g2没有及时将g4, g5 关闭，协程G将无法和g4, g5 建立联系，也就是说g4, g5会一直开启并占用系统资源。当然，只需要在g2中写好关闭它的子协程的业务逻辑，可以避免这个问题，但是如果协程更多，更复杂，那么需要额外添加的代码就更多，也更容易出错。

# Context 一个接口，四个实现，六个函数

一个接口

- Context

四个实现

- emptyCtx
- cancelCtx
- timerCtx
- valueCtx

六个函数

- Background
- TODO
- WithCancel
- WithDeadline
- WithTimeout
- WithValue

## context 接口

```
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}

```

## emptyCtx
`emptyCtx` 是一个整型变量， 它实现了 `context` 接口，但是什么也没做。`Background` 和 `TODO` 两个函数都会创建 `emptyCtx`。

`Background()` 用于初始化一个 ctx。它的接口类型是 `Context`, 动态类型是 `emptyCtx`， 值的本质是一个0。

`TODO()` 虽然底层实现和 `Background()` 一模一样，但官方希望它被用在，当我们不知道使用哪个context或者函数当前并不需要引入一个 context。换句话说，它被用在代码还没写完的部分，类似我们经常在注释里写的 TODO。


## cancelCtx

这是可取消的context。

```
type cancelCtx struct {
	Context

	mu       sync.Mutex            
	done     chan struct{}
	children map[canceler]struct{}
	err      error
}
```

- mu: 保证线程安全
- done：context取消通知
- children：记录以当前context为根节点的所有可取消的子context，便于级联取消
- err：存储取消时的错误信息

`WithCancel` 能将一个context包装成可取消的。
```
ctx := context.Background()
ctx1, cancel := context.WithCancel(ctx)

```

## timerCtx

```
timer *time.Timer // Under cancelCtx.mu.

deadline time.Time
```
`timerCtx` 也是一个可取消的 context。 它在 `cancelCtx` 的基础上又封装了一个计时器和一个截止时间。

它可以用 `WithDeadline` 和 `WithTimeout` 创建。
- WithDeadline: 指定一个时间点
- WithTimeout: 指定时间段

## valueCtx

valueCtx 能打包键值对
```
ctx := context.Background()
key1 := "key1"
ctx1 := context.WithValue(ctx, key1, "value1")
```

使用valueCtx打包键值对时会出现这样的问题：
当对同一个key赋值的两个context, 它们最终的键值对是相同的。
```
ctx := context.Background()
key1 := "key1"
key2 := "key2"

ctx1 := context.WithValue(ctx, key1, "value1")
ctx2 := context.WithValue(ctx1, key2, "value2")

fmt.Println("key1 ->", ctx2.Value(key1))
fmt.Println("key1 ->", ctx2.Value(key2))
----
key1 -> value2
key2 -> value2
```

这是因为 `Value()` 在查找前回去比对要 `查找的key` 和 `当前context中的key`。如果相同，直接返回当前context的key。

为了规避这个情况，我们可以将key的类型换成自定义类型，使 `Value()`中的 `c.key == key` 失效。
```
type keyType1 string
var key1 keyType1 = "key1"
```

另外，context 储存的键值对不支持修改。
