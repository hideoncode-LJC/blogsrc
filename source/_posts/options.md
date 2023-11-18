---
title: options.md
date: 2023-11-08 20:33:41
tags: 语言
categories: Go
---

# Go语言设计模式
> 函数式选项模式

## 为什么需要Options

### 传统的面向对象模式

**传统的面向对象模式的结构体**

```go
// declare Struct

type DoSomeThingOptions struct {
    a int
    b float32
    c string
}

func NewDoSomeThingOptions(a int, b float32, c string) *DoSomeThingOptions {
    return &DoSomeThingOptions {
        a: a, 
        b: b, 
        c: c,
    }
}

func (d *DoSomeThingOptions) Do() {

}
```

**传统的面向对象模式的缺点**
1. 上述例子的参数只有三个，构造函数和参数的默认值设置都很简单，但是当参数十几个的时候就会非常复杂。
2. 如果结构体需要修改，则构造函数也需要修改，调用构造函数的位置也需要修改。

基于以上两个缺点，Go使用了函数式选项模式。

## 函数式选项模式
> funcational options pattern

允许你使用接受零个或多个函数作为参数的可变构造函数构建复杂结构。我们将这些函数称为选项，由此得名函数选项模式。

### 可选参数
Go语言是不支持默认值参数的，但是支持可变参数，可以利用这个特点来写构造函数。
可变函数参数应该满足三个特性
 
+ 不同的参数拥有相同的类型？
+ 指定函数参数能为特定的配置项赋值
+ 支持扩展新的配置项


**根据可选参数写构造函数**

```go
/* 
结构体如上图
假设a为必须输入的参数
*/

// 为什么使用指针，值传递和指针传递的区别

type DoSomeThingOption func(*DoSomeThingOptions)

func NewDoSomeThingOptions(a int, doSomeThingOption DoSomeThingOption...) {
    // 声明一个结构体
    // 先设置必须传递的参数
    o := DoSomeThingOptions {
        a: a,
    }
    // 然后设置可选参数
    for _, opt := range opts {
        opt(o)
    }
    return &o;
}

// 构造可选函数 标准函数名 With参数名
func WithB(b float32) DoSomeThingOption {
    return func(d *DoSomeThingOptions) {
        d.b = b
    }
}

func WithC(c string) DoSomeThingOption {
    return func(d *DoSomeThingOptions) {
        d.c = c
    }
}


// 构造一个结构体
func main() {
    dosomethingoptions := NewDoSomeThingOptions {
        a: 1,
        // 剩下的参数可写可不写
        WithB(12.3245),
        WithC("Hello World!"),
    }
    fmt.Printf("%+v\n", dosomethingoptions)
}

```

**根据可变参数构造含有默认值的构造函数**

```go
/*
设置默认值的原理
1. 先对参数设置默认值
2. 如果后续存在options，则会修改该值。
*/

const defaultBValue = 1.1

func NewDoSomeThingOptions(a int, opts DoSomeThingOption...) {
    o := DoSomeThingOptions{
        a: 1,
        b: defaultBValue,
    }
    for _, opt := range opts {
        opt(o)
    }
    return &o
}
```

## 接口类型的构造函数


```go
type doSomeThingOptions struct {
    a int
    b float32
    c string
}

type doSomeThingOption interface {
    apply(*doSomeThingOptions)
}

type adoSomeThingOption struct {
    f func(*)
}

type 


```
