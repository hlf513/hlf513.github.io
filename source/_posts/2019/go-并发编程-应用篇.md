---
layout: "post"
title: "Go 并发编程-应用篇"
date: "2019-07-12 01:16"
tags:
  - go
categories:
  - go
---
当提到并发编程的时候，人们往往会想到多线程，而 Go 最被人熟知的是借鉴 CSP 的 gorountine & channel 并发模式，那 Go 中是否支持类似传统多线程的并发编程方式呢？答案是支持；因为 Go 的 Sync 包给我们提供了互斥锁、原子操作、条件变量等同步原语。

<!-- more -->

本文主要介绍下同步原语的用法，并发下的数据存储，以及 Goroutine 的使用方式和注意事项。

# 同步原语

## Mutex

> 互斥锁，1.9 引入「饥饿模式」解决了尾延时（队列头的 Goroutine 会被新建 Goroutine 抢占锁）

``` go
var s sync.RWMutex

s.Lock()

// handler something

s.Unlock()
```



## RWMutex

>  读写互斥锁，用于并发读

```go
var s sync.RWMutex

s.RLock()

// handler something

s.RUnlock()
```

## Once

> 只执行一次；即使执行的是不同的 func，也只会执行一次。

Once 只有一个 Do 方法，参数是函数

```go
var s sync.Once

onceFunc := func(){
  print("once")
}

onceFunc2 := func(){
  print("once")
}

// 只会输出一个 once
s.Do(onceFunc)
s.Do(onceFunc2)
```

## Atomic

> 原子操作

sync/atomic 提供了针对整型、通用类型的原子操作。

针对整型的方法有：加(Add)、CAS(交换并比较 compare and swap)、存储(store)、读取(load)以及交换(swap)：

```go
// 还支持 uint32、int64 等
var i int32
i = 1
atomic.AddInt32(&i, 2)
// 不会改变 i 的值
atomic.CompareAndSwapInt32(&i, 2, 10)
// 会改变 i 的值
atomic.CompareAndSwapInt32(&i, 3, 11)
// 交换值，并返回旧值的指针
atomic.SwapInt32(&i, -1)
fmt.Println(atomic.LoadInt32(&i))
// 存储值
atomic.StoreInt32(&i, 1)
```

针对通用类型的方法有：Store 存储、Load 读取

``` go
var atomicVal atomic.Value

str := "hello"

// 存储
atomicVal.Store(str)
// 读取
fmt.Println(atomicVal.Load())
```

## Cond

> 让 Goroutine 在某个条件下被阻塞/唤醒

Cond 提供三个方法：Wait 阻塞、Signal 唤醒、Broadcast 唤醒所有 Goroutine；

```go
 func main() {
	// 初始化需要一个锁
	c := sync.NewCond(&sync.Mutex{})

	for i := 0; i < 10; i++ {
		go listen(c)
	}

	go broadcast(c)

	ch := make(chan os.Signal, 1)
	signal.Notify(ch, os.Interrupt)
	<-ch
}

func listen(c *sync.Cond) {
	c.L.Lock()
	c.Wait() // 阻塞；以下代码暂不执行
	fmt.Println("listen")
	c.L.Unlock()
}

func broadcast(c *sync.Cond) {
	c.L.Lock()
	c.Broadcast() // 唤醒所有的 Goroutine
	fmt.Println("release")
	c.L.Unlock()
}
```

调用 Singal 会唤醒阻塞时间最长的 Goroutine。

## WaitGroup

> 最常用的同步机制，多用于等待一批 Goroutine 的返回

WaitGroup 提供三个方法：Add 计数、Done 完成（计数-1）、Wait 阻塞到计数为0为止

```go
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
  wg.Add(1)
  go func() {
    defer func() {
      wg.Done()
    }()
    fmt.Println("handler something")
  }()
}

wg.Wait()
```

# 数据存储

## Map

> 1.9 引入的并发安全 sync.map（1.9 之前的 map 在不加锁的前提下进行写操作会报错）

Map 提供五个方法：Store 存储键值对、LoadOrStore 读取或存储键值对、Load 读取键值对、Delete 删除键值对、Range 遍历键值对

```go
var m sync.Map

m.Store("k", "v")
m.Load("k")
// 不会覆盖 k 的值
m.LoadOrStore("k", "v1")
// 会覆盖 k 的值
m.Store("k", map[string]string{
  "k1": "v1",
})
m.Range(func(key, value interface{}) bool {
  fmt.Println(key, value)
  return false
})
```

## Pool

> 并发安全，用于存储可复用的临时对象，以减少垃圾回收的压力

Pool 提供了两个方法：Get 获取对象，Put 放入对象

```go
// 一个[]byte的对象池，每个对象为一个[]byte
var bytePool = sync.Pool{
  // 获取对象不存在时则创建
  New: func() interface{} {
    b := make([]byte, 1024)
    return &b
  },
}
// 获取对象
obj := bytePool.Get().(*[]byte)
_ = obj
// 对象放入池子
bytePool.Put(obj)

```



# 上下文 Context

当我们在 Goroutine 中再次产生一个 Goroutine 的时候，若前者异常退出，理论上后者也应该退出，否则就是对资源的浪费。

**Context 的主要作用就是在不同的 Goroutine 之间同步请求特定的数据、取消信号以及处理请求的截止日期。**

可以把 Context 类比为一棵树结构：

* 根节点不可取消；那生成根节点的方法有：context.Background()、context.TODO()；两者的区别是没有本质区别，当不知道使用什么的时候用 TODO()；常用的是 Background()。

* 生成子节点的方法有：WithCancel 手动取消、WithDeadline 定时取消、WithTImeout 定时取消、WithValue 存键值对

``` go
ctx, cancel := context.WithCancel(context.Background())
// 手动取消
cancel()

// 定时取消 - 截止
d := time.Now().Add(50 * time.Millisecond)
ctx, cancel := context.WithDeadline(context.Background(), d)
defer cancel()

// 定时取消 - 超时
ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
defer cancel()

// 存键值对(先找子节点再找父节点)
ctx := context.WithValue(context.Background(), "kkk", "vvv")
fmt.Println(ctx.Value("kkk"))
```

键值对不要存过多的参数，一般常用来存储 global trace id。

# Channel

Channel 分为有 buffer channel 、无 buffer channel；若是无 buffer 的 channel，在一端没有准备好数据之前，另一端会阻塞；若是有 buffer 的 channel，则 buffer 未满之前是不阻塞的。

使用起来比较简单；提供四个方法：

1. 创建 channel
2. 数据写入 channel
3. 读取 channel 数据
4. 关闭 channel（无法写入数据，但可以读取）

```go
func main() {
	// 无缓存
	ch1 := make(chan int)
	// 有缓存
	//ch1 := make(chan int,10)

	// 接收数据
	go receive(ch1)

	// 发送数据
	send(ch1)

	// 关闭 channel
	close(ch1)
}

func send(ch chan int) {
	for i := 0; i < 2; i++ {
		ch <- i
	}
}

func receive(ch chan int) {
	d := <-ch
	fmt.Println("one", d)

	for d := range ch {
		fmt.Println("two", d)
	}
}
```

channel 还支持只读、只写操作：

```go
func main() {
	var ch = make(chan int)
	go receive(ch)
	send(ch)
	close(ch)
	time.Sleep(1 * time.Second)
}

// 只读
func receive(ch <-chan int) {
	for v := range ch {
		fmt.Println(v)
	}
}

// 只写
func send(ch chan<- int) {
	for i := 0; i <= 10; i++ {
		ch <- i
	}
  // 只写里可以执行关闭 channel 的操作
  // close(ch)
}
```

## 使用场景

1. 信号传递
   把数据当做信号放入 channel；一般使用无 buffer 的 channel；常和 WaitGroup 配合控制并发数。

   ```go
   func main() {
   	ch := make(chan int)

   	go func() {
   		fmt.Println("handler something")
   		// 通知 main goroutine 任务处理完毕
   		ch <- 1
   	}()

   	// 阻塞；等待新建 goroutine 中处理任务
   	<-ch
   }
   ```

2. 消息队列
   把数据放入 channel 等待消费；一般使用有 buffer 的 channel。

3. 多个 channel 串联为 Pipeline
   每个 channel 的输出当做另一个 channel 的输入。
![pipeline](/images/2019/07/pipeline.png)
```go
func main() {
	ch1 := make(chan int)
	ch2 := make(chan int)

	// 计数
	go counter(ch1)
	// 平方
	go square(ch1, ch2)
	// 输出
	go output(ch2)

	time.Sleep(1 * time.Second)
}

func counter(ch1 chan int) {
	for i := 0; i < 5; i++ {
		ch1 <- i
	}
	close(ch1)
}

func square(ch1 chan int, ch2 chan int) {
	for i := range ch1 {
		i *= i
		ch2 <- i
	}
	close(ch2)
}

func output(ch2 chan int) {
	for i := range ch2 {
		fmt.Println(i)
	}
}
```

## Select

多路复用；语法类似 Switch，有 default 则不阻塞，无 default 则：

- 条件都未成立，则阻塞
- 条件分支某个成立，则执行
- 条件分支都成立，则随机选择一个

```go
for {
  select {
    case d1 := <-ch1:
    fmt.Println(d1)
    case <-time.After(3 * time.Second): // 设置超时时间
    fmt.Println("timeout")
    break
    default:
    fmt.Println("default")
  }
}
```

## Goroutine 泄露

Goroutine 的开销很小，但若是使用不当，造成 GC 无法回收的话，久而久之就会引起内存耗尽。

## 常见场景

1. nil channel

    永远阻塞

   ```go
    func main() {
    	var ch chan int

    	go nilChannel(ch)
    }

    func nilChannel(ch chan int) {
    	<-ch
    }
   ```

2. 没有接收者的 channel

   例：并发请求两个搜索引擎，响应结果写入 channel；我们使用最先收到的响应，丢失之后的响应，这样会造成后者 goroutine 一直阻塞
   解决方案：保证 channel 里的数据都会被读取，或者使用 context 取消其他请求。
   ``` go
   func main() {
    	var ch = make(chan int)

    	// 查询结果
    	go func() {
    		go baidu(ch)
    		go bing(ch)
    	}()

    	// 输出查询结果
    	go res(ch)

    	time.Sleep(1 * time.Second)
    }

    func baidu(ch chan int) {
    	res, _ := http.Get("https://baidu.com")
    	ch <- res.StatusCode + 1
    }

    func bing(ch chan int) {
    	res, _ := http.Get("https://cn.bing.com")
    	ch <- res.StatusCode + 2
    }

    func res(ch chan int) {
    	code := <-ch
    	fmt.Println(code, runtime.NumGoroutine())
    }
   ```

3. 没有接收者的 channel
4. 程序死循环


由此可以看出，goroutine 泄露都是因为使用不当造成的；所以我们在使用 goroutine 的时候一定要小心。

## 泄露检测

1. 观察 runtime.NumGoroutine

2. pprof

   1. 通过 web 查看
      > net/http/pprof

      ```go
      import (
          "log"
          "net/http"
          _ "net/http/pprof"
      )

      ...

      log.Println(http.ListenAndServe("localhost:6060", nil))
      ```
      访问 http://localhost:6060/debug/pprof/goroutine?debug=1 查看 gorutine 状态

   2. 通过 stdout 查看
      > runtime/pprof

      ```go
      import (
          "os"
          "runtime/pprof"
      )

      ...

      pprof.Lookup("goroutine").WriteTo(os.Stdout, 1)
      ```

3. gops

   > https://github.com/google/gops

   ```go
   import "github.com/google/gops/agent"

   ...

   if err := agent.Start(); err != nil {
       log.Fatal(err)
   }
   time.Sleep(time.Hour)
   ```

4. leaktest

   > https://github.com/fortytw2/leaktest

   基本原理：在测试的开始和结束的时候，利用 runtime.Stack 获取活跃 goroutine 的堆栈跟踪。如果在测试完成后还有一些新的 goroutine，那么将其归类为泄露。



# 参考资料

1. [同步原语与锁](https://draveness.me/golang/concurrency/golang-sync-primitives.html)
2. [Golang非CSP并发模型外的其他并行方法总结](https://juejin.im/post/5c1dce2d51882508506061f2#heading-10)
3. [Goroutine 泄露](https://studygolang.com/articles/12495)
