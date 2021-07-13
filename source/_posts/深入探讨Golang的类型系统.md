---
title: 深入探讨Golang的类型系统
date: 2021-06-09 19:33:42
tags:
	- golang
	- 类型
categories: golang
---

# 先讲一讲Golang的方法

```
type T struct {
    name string
}
func (t T) F1() {
    fmt.Println(t.name)
}
func main(){
    t := T{name: "a"}
    t.F1()
}
```

像上面这段代码，我们定义了一个结构体，并给这个结构体关联了一个方法， 并使用 `t.F1()`语句调用了这个方法。

方法的本质是一个函数，只是这个函数在被调用时，接收者会作为第一个参数传入。（`t.F1()` 的本质是 `T.F1(t)`。 前者的写法只是语法糖）

在编译阶段，我们可以获取变量，类型，方法等参数。

到了执行阶段，像 `反射`， `接口`， `类型断言` 这类语言特性/ 机制，我们也需要动态获取数据类型信息。

# 我们先看看Golang有哪些类型？
## 内置类型(build-in)

```
int8
int16
int32
int64
int
byte
string
slice
func
map
```

## 自定义类型

```
type typ int

type T struct {
    name string
}

type I interface{
    Name() string
}
```

在 Golang 官方定义中， 内置类型不能定义方法，接口类型是方法的无效接收者。

换句话说，我们只能给 `type T int` 和 `type T struct{}` 定义方法。

## 类型元数据

无论是内置类型还是自定义类型，每种类型都有全局唯一的 `类型元数据`。

类型元数据记录了如下信息:(rutime/type.go)

```
type _type struct {
    size        uintptr
	hash        uint32
	tflag       tflag
	align       uint8
	fieldAlign  uint8
	kind        uint8
    ....
}
```

有些类型元数据可能还需要对信息进行扩展，比如切片类型

```
type slicetype struct {
    typ     _type
    elem    *_type
}
```

因为切片类型可以是整型切片，也可以是字符串切片，因此，它还需要一个elem字段来描述储存元素的类型。比如整形切片，它的类型元数据的elem字段需要指向整形的类型元数据。

对于自定义类型，它们可能还需要更多的描述，这些描述被储存在 `uncommontype` 结构体中。

```
type uncommontype struct {
	pkgpath nameOff
	mcount  uint16 // 方法数
	xcount  uint16 // exported方法数
	moff    uint32 // 相对于方法元数据数据的偏移值
	_       uint32 // unused
}
```

## 类型的两种写法区别

- `type MyType1 = int32`
    - `Mytype1` 和 `int32` 会被关联到同一个类型元数据，比如 `rune` 和 `int32` 就是这样的关系。
- `type MyType2 int32`
    - `MyType2` 和 `int32` 会各自拥有自己的类型元数据。


# 接口类型

## 空接口类型
空接口类型可以接收任意类型的数据，也就是说它需要记录接收数据的类型和数据的地址。

```
type eface struct {
	_type *_type            // 动态类型
	data  unsafe.Pointer    // 动态值
}
```
下图展示了一个空接口变量赋值后，其元数据的变化。

![interface.png](https://i.loli.net/2021/06/09/93lIxa4JU6GHAZj.png)

## 非空接口类型
非空接口类型即拥有一个方法列表的类型。一个变量要想赋值给一个非空接口类型，它需要实现这个接口所有的方法。

```
type iface struct {
	tab  *itab
	data unsafe.Pointer // 指向接口的动态值
}
```

![non-nil-interface.png](https://i.loli.net/2021/06/09/NtDmKZUFSEWP2i3.png)

非空接口的 
- `tab.inter` 指向了接口类型的元数据，其中的 `mhdr` 是接口的方法列表。
- `_type` 是接口的动态类型
- `hash` 是类型哈希值，用于接口类型的快速比较
- `fun` 是方法地址数组

itab 结构体中的值，在它被确定后就不会发生改变，所以它是可复用的。

![itab.png](https://i.loli.net/2021/06/09/7HebRa1sKAwzqyk.png)

Golang 会将接口类型和动态类型作为一个组合key， itab结构体指针为value, 形成一张轻量级的哈希表。当我们需要一个itab时，会先在哈希表中查找，如果找到了，直接复用，否则会创建一个新的itab，添加到这张哈希表中。

# 类型断言

像上面提到的 `空接口` 和 `非空接口` ，被叫做 `抽象类型`; 与之对应的 int, string, slice 等类型就是具体类型。

类型断言是作用在抽象类型之上的， 而断言的目标类型，可以是具体类型，也可以是非空接口。两两组合，可以得到4种断言。

- 空接口.(具体类型)
- 非空接口.(具体类型)

## 空接口.(具体类型)

`r, ok := interface.(*type)`

对于这种类型断言，我们只需要检查 interface 元数据中的 `_type` 字段是否指向 断言的`*type`， 是则 Ok 为 true, 否则 ok 为 false，且 r 为 nil

## 非空接口.(具体类型)

与空接口不同的是，这种断言方法只需检查 itab 是否指向目标类型结构体。

## 空接口.(非空接口)

在我们检查接口类型元数据时， 我们需要比对 空接口的方法元数据数组和目标类型的方法列表是否相同。当然我们不通过空接口的类型元数据直接比对，还是通过itab缓存。但是通过itab缓存时，虽然查到了目标itab，但还是需要比对itab中的方法 `f[0]` 是否为 0。 这是因为，在某次类型断言失败时，itab缓存还是会添加itab到哈希表中，只是会将其方法列表置0，代表断言失败。

## 非空接口.(非空接口)

同样，从itab缓存查itab结构体指针，并去查f[0]是否为0.

# 小结

Golang 的类型断言的关键在于明确接口动态类型，与对应类型实现了哪些方法。

这些明确内容的关键点在于 `类型元数据` `空接口` `非空接口`