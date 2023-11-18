---
title: GMP模型
date: 2023-11-09 20:33:41
tags: 语言
categories: Go
---

# GMP 模型

## Golang调度器的由来

**进程调度的缺点**

1. 需要消耗大量的资源。
2. 如果参与调度的进程太多，CPU大量资源被用于进程调度。

**内核级线程**

1. 是操作系统的最小调度单元，最小的资源的分配单元还是进程。
2. 创建、销毁、调度均由内核完成，CPU需要完成和内核态的切换。
3. 可充分类用多核，实行并行。

### 




# GoRoutines

**G**


```go
type g struct {
    // ...
    m *m // 当前g正在哪个m上执行
    // ...
    sched gobuf

}

type gobuf struct {
    sp uintptr // 保存CPU的rsp寄存器的值，指向函数调用栈栈顶
    pc uintptr // 保存CPU的rip寄存器的值，指向程序下一条执行指令的地址 
    ret uintptr // 保存系统调用的返回值
    bp uintptr // 保存CPU的rbp寄存器的值，存储函数栈帧的起始位置
}
```

**g的声明周期**

```go
const (
    _Gidle = iota // GoRoutines刚开始被创建，初始化还没完成
    _Grunnable // GoRoutines初始化完成，等待被执行
    _Grunning // GoRoutines正在被执行，同一时刻对一个Processor来说，只能由一个GoRoutines位于该状态
    _Gsyscall
    _Gwaiting // 挂起态，等待被唤醒 GC channel通信、IO、锁
    _Gsys

)
```

**m**

```go
type m struct {
    /* 
    goroutine with scheduling stack 
    一类特殊的调度协程，不用于执行用户函数，负责执行g之间的切换调度，与m的关系为1:1
    */
    g0 *g
    /*
    tls thread-local storage 
    线程本地存储，存储内容只对当前线程可见，线程本地存储的是m.tls的地址，m.tls[0]存储的是当前运行的g，因此线程可以通过g找到当前的m、p、g0等信息
    */
    tls [tlsSlots]uintptr
}
```

**P**

```go
type p struct {
    // ...
    runqhead uint32 // 队列头部
    runqtail uint32 // 队列尾部
    /*
    本地Goroutine队列，最大长度为256
    */
    runq [256]gintptr
    runnext guintptr // 下一个可以执行的Goroutine
    //...
}
```

**Schedt**

```go
type schedt struct {
    // ...
    /*
    操作全局队列时使用的锁
    */
    lock mutex
    // ...
    /*
    全局队列
    */
    runq gQueue
    /*
    全局队列的大小
    */
    runqsize int32
    //...
}
```


# 调度流程

## `Schedule`函数

+ 根据策略找到下一个可以执行的goroutine。`findRunnable`函数
+ 执行该goroutine。`execute`函数

### `findRunnable`函数

**根据策略判断是否从全局队列中获取一个g**

1. 判断是否是61次调度
2. 如果是，则从全局队列获取一个g 直接执行？
3. 额外从全局队列中获取一个g‘放入到p的本地队列中，让全局队列的g也有充分执行的机会 `runqput`函数
    + 如果本地队列已满，则把本地队列的一半连同g'一起交还给全局队列，帮助p缓解压力
    + 如果本地队列还有空间，则直接把g'放入到p的局部队列中。
4. 返回g

```go
func schedule() {
    // ...
    // 寻找下一个可以执行的GoRoutine
    gp, inheritTime, tryWakeP := findRunnable() // 阻塞直到工作可用
    // ...
    // 执行该Goroutines
    execute(gp, inheritTime)
}

func findRunnable() (gp *g, inheritTime, tryWakeP bool) {
    // 如果Processor已经执行了61次调度，并且当前的全局队列中存在待执行的g
    if _p_.schedtick % 61 == 0 && sched.runqsize > 0 {
        // 对全局队列进行加锁
        lock(&sched.lock)
        // 从全局队列中获取一个g
        gp = globrunqget(_p_, 1)
        // 解锁
        unlock(&shced.lock)
        /*
        gp == nil ?
        可能会获取失败？
        */
        if gp != nil {
            return gp, false, false
        }
    }
}

func globrunqget(_p_, *p, max int32) *g {
    /*
    判断全局队列的size
    如果全局队列为空，则无法获取g
    */
    if sched.runqsize == 0 {
        return nil
    }


    /*
    当p的个数等于一时，n就是全局队列g的个数
    */
    n := sched.runqsize/gomaxprocs + 1

    if n > sched.runqsize {
        n = sched.runqsize
    }

    if max > 0 && n > max {
        n = max
    }

    if n > int32(len(_p_.runq)) / 2 {
        n = int32(len(_p_.runq)) / 2
    }

    sched.runqsize -= n

    gp 
}
```



# 调度器调度

```go
func schedinit() {
    // 初始化调度器
	_g_ := getg()
	...

    // 一个Go语言程序能够创建的最大线程数。
    // 能够运行的线程数由GOMAXPROCS决定
    // 可能有些线程去执行阻塞任务
	sched.maxmcount = 10000

	...
	sched.lastpoll = uint64(nanotime())
	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
    /*
    procresize函数执行流程
    全局变量 allp = []p* 存储处理器结构体
    1. GOMAXPROCS的个数代表正在执行的线程数，也代表着当前P的个数，所以该函数的第一个操作就是判断当前存在的p的个数和期望的p的个数。如果不够，需要进行扩容。
    2. new p; append(allp, p) until len(allp) == GOMAXPROCS
    3. 将allp[0]和线程m0绑定在一块 main thread
    4. 释放多余的处理器结构,截断allp,确保allp的长度和期望的处理器的数量相同
    5. 将多余的P设置成_Pidle状态,并且加入到全局空闲队列中
    */
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
}
```

# 创建Goroutine

Go通过go关键字创建一个新的Goroutine。编译器通过
`gc.state.stmt`和`gc.state.call`两个方法转换成`runtime.newproc`函数调用。
```go
func (s *state) call(n *Node, k callKind) *ssa.Value {
	if k == callDeferStack {
		...
	} else {
		switch {
		case k == callGo:
			call = s.newValue1A(ssa.OpStaticCall, types.TypeMem, newproc, s.mem())
		default:
		}
	}
	...
}
```

```go
func newproc(siz int32, fn *funcval) {
	/*
    unsafe.Pointer 可以对地址进行操作
    add函数将当前函数的地址+sys.PtrSize
    */
    argp := add(unsafe.Pointer(&fn), sys.PtrSize)
    // 获取当前正在运行的Goroutine
	gp := getg()
    /* 
    获取调用该函数的PC数值,得到返回地址,用于新的Goroutine的调用栈
    为什么需要获取PC值？
    */
	pc := getcallerpc()
    /*
    在系统栈？内核栈上执行一段代码
    一种在系统级别上运行的特殊Goroutine
    允许运行时执行一些可能需要特殊处理的任务
    */
	systemstack(func() {
        /*
        fn Go func func的函数地址
        argp 函数地址+sys.PtrSize的地址
        siz
        新的结构体的地址
        PC
        */
		newg := newproc1(fn, argp, siz, gp, pc)
        /*
        获取执行g的p的指针
        */
		_p_ := getg().m.p.ptr()
        /*
        将新建的goroutine放入当前p的运行队列
        true表示如果新的Goroutine是可以运行的,则唤醒任何正在等待的阻塞P
        */
		runqput(_p_, newg, true)
        /*
        如果mainStarted为真,则唤醒一个P
        以便他可以运行新的Goroutine
        通常发生在程序的启动阶段
        */
		if mainStarted {
			wakep()
		}
	})
}

func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g {
    // 获取执行当前Goroutine的g结构体
	_g_ := getg()
    // 将narg赋值给siz 并且进行内存对齐
	siz := narg
	siz = (siz + 7) &^ 7
    //  
	_p_ := _g_.m.p.ptr()
	newg := gfget(_p_)
	if newg == nil {
		newg = malg(_StackMin)
		casgstatus(newg, _Gidle, _Gdead)
		allgadd(newg)
	}
	...
```