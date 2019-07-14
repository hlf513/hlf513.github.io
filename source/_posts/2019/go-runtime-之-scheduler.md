---
layout: post
title: Go 并发原理 - 调度篇
date: '2019-07-14 10:02'
tags:
  - go
  - runtime
  - scheduler
categories:
  - Go runtime
---

众所周知，`Goroutine` 是 Go 最小的执行单位，作为开发者，我们在使用「廉价」的 `Goroutine` 的时候，有没有想过背后的实现原理？有没有因为「滥用」`Goroutine` ，而踩过坑？我们今天来简要分析下 Go 的并发原理。

<!-- more -->

# 背景知识
## 线程模型
在讲解 `Go Scheduler` 之前，我们先来了解下目前主流的线程模型；按照用户线程和内核线程之间的关系，可以分为以下三种：

### 内核级线程模型
> 用户线程：内核线程 = 1：1

![thread2](/images/2019/06/thread2.png)
每个用户线程绑定一个内核线程，调度由「内核」完成；

优点：实现简单（直接调用「系统调用」），可以并行（利用多核）
缺点：占用资源多，上下文切换慢

### 用户级线程模型
> 用户线程：内核线程 = N：1

![thread1](/images/2019/06/thread1.png)
多个用户线程(从属于单个进程)绑定一个内核线程，调度由「用户」完成；也就是说，操作系统只知道用户进程而对其中的线程是无感知的，内核的所有调度都是基于用户进程；许多语言的「协程库」都属于这种方式。

优点：占用资源少，上下文切换快，性能优于「内核级线程模型」
缺点：不支持多核，原生不支持并发（单个用户线程有阻塞调用，则所有线程都阻塞；因为此模式下操作系统的调度最小单位为进程）

可以通过封装阻塞调用为非阻塞调用，来避免所有线程都阻塞。

### 两级(混合型)线程模型
> 用户线程：内核线程 = M：N

一个用户进程（包含多个线程）可以「动态绑定」多个内核线程；调度由「内核、用户」完成（用户调度器实现用户线程到内核线程的调度，内核调度器实现内核线程到CPU上的调度）；比如：某个内核线程由于其绑定的用户线程阻塞了，这个内核线程关联的其他用户线程可以重新绑定其他内核线程。

优点：支持并发，占用资源少
缺点：调度复杂

Go 采用的是此线程模型。

## CSP 并发模型
> Do not communicate by sharing memory; instead, share memory by communicating

CSP 最早是由 Tony Hoare 在 1977 年提出，严格来说，CSP 是一门形式语言，用于描述并发系统中的互动模式，也因此成为一众面向并发的编程语言的理论源头，并衍生出了 Occam/Limbo/Golang…
> [《Communicating Sequential Processes》](http://www.usingcsp.com/cspbook.pdf)

不同于传统的多线程通过「共享内存」来通信，CSP 提倡「以通信的方式来共享内存」。

Go 其实只用到了 CSP 的很小一部分，即理论中的 Process/Channel（对应到 Go 中的 goroutine/channel）：这两个并发原语之间相互独立，Process 通过对 Channel 进行读写，形成一套有序阻塞和可预测的并发模型。

# Runtime

> 本质上，操作系统运行线程，线程运行你的代码。

Go 编译器会在 Go 运行时的一些地方插入系统调用，（比如通过 channel 发送值,调用 runtime 包等），所以 Go 可以通知调度器执行特定的操作。

![go scheduler1](/images/2019/06/go-scheduler1.png)

## Goroutine
在 Golang 中，任何代码都是运行在 goroutine 里，即便没有显式的 `go func()` ，默认的 main 函数也是一个 goroutine。

1. Goroutine 不是协程；Go 使用的是两级线程模型
2. Goroutine 开销很小；它有自己的栈，不同于内核线程固定大小的内存块(2MB)，goroutine 的栈初始值是 2KB，可以动态增长到 1G（64位1G，32位256M），GC 还会周期性回收不再使用的内存，收缩栈空间）

## Scheduler
> [《Go Scheduler 设计文档》](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw)

### 概述

Go scheduler 内部有三个重要的结构：M、P、G
* M：OS 线程
* P：Go 代码执行所需要的资源
* G：Goroutine

![MPG](/images/2019/06/mpg.png)

在 Go 1.0 的时候，它的调度器是 MG 模型，有以下几个问题：

1. 全局锁 (每个 Goroutine 的操作都要上锁)
2. M 直接和 G 交互，导致调度延迟、性能损耗（OS 调度）
3. 每个 M 都有自己的 cache，导致内存占用过高
4. 系统调用形成的阻塞，导致性能呢损耗

为了解决以上几个问题，在 Go 1.1 的时候，引入了 P，并实现了 [work stealing](http://supertech.csail.mit.edu/papers/steal.pdf) 调度算法，演进为现在的 MPG 模型。

### MPG 模型

![Go scheduler2](/images/2019/06/go-scheduler2.png)


# 参考资料
1. [Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/)
1. [Goroutine并发调度模型深度解析之手撸一个协程池](https://segmentfault.com/a/1190000015464889)
2. [Go并发原理-贝壳](https://mp.weixin.qq.com/s?__biz=MzIyMTg0OTExOQ==&mid=2247484436&idx=2&sn=2864adc11a787c1c3ee79e8dbd96c27a&chksm=e8373764df40be72df5a6240998b19ab0ff45ac9a53a9e5c6b7ed0d7eabedd5a8098db9d87c2&mpshare=1&scene=1&srcid=07272kBJCM3KUJy3tfps1FdM%23rd)
3. [Go 调度器: M, P 和 G](https://colobu.com/2017/05/04/go-scheduler/)
4. [Go Runtime Scheduler](https://speakerdeck.com/retervision/go-runtime-scheduler)
