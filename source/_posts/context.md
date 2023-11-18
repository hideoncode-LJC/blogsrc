---
title: context.md
date: 2023-11-08 20:33:41
tags: 语言
categories: Go
---

# Context 理解

## 源码理解

**接口定义**

```go
type context interface {
    Deadline() (deadlint time.Time, ok bool)
    Done() <-chan struct {}
    Err() error
    Value(key any) any
}
```

+ `Deadline()`方法返回了该`Context`的过期时间，`bool` ?
+ `Done`方法返回了一个空结构体类型的通道,当通道存在信号时,表示该`Context`方法被取消。
+ `Err()`方法返回`Context`被取消的原因
+ `Value()`传递值

**错误类型定义**

```go
var 
```