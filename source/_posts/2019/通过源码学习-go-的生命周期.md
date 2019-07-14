---
layout: "post"
title: "通过源码学习 Go 的生命周期"
date: "2019-07-14 16:26"
tags:
  - go
categories:
  - go
---
当我们学习一个新事物时，一定要做到「知其然知其所以然」。最近看了一些 go 的源码，对 go 的设计有了更深的认识，也让自己在工作中使用的更得心应手。

<!-- more -->

本文基于以下版本：
``` sh
go version go1.12.7 darwin/amd64
```

## 四个重要结构
首先，我们了解下这四个核心的 struct，有助于帮助我们理解下文。
### schedt
> src/runtime/runtime2.go

``` go
type schedt struct{
  // ...

  // M 相关
  midle        muintptr // idle m's waiting for work
  nmidle       int32    // number of idle m's waiting for work
  nmidlelocked int32    // number of locked m's waiting for work
  mnext        int64    // number of m's that have been created and next M ID
  maxmcount    int32    // maximum number of m's allowed (or die)
  nmsys        int32    // number of system m's not counted for deadlock
  nmfreed      int64    // cumulative number of freed m's

  // ...

  // P 相关
  pidle      puintptr // idle p's
  npidle     uint32
  nmspinning uint32 // See "Worker thread parking/unparking" comment in proc.go.

  // G 全局队列
  runq     gQueue
  runqsize int32

  // Global cache of dead G's.
  gFree struct {
    lock    mutex
    stack   gList // Gs with stacks
    noStack gList // Gs without stacks
    n       int32
  }

  // ...

  // 等待释放的 M 列表
  freem *m

  // ...

  // GC 、sysmon相关
  gcwaiting  uint32 // gc is waiting to run
  stopwait   int32
  stopnote   note
  sysmonwait uint32
  sysmonnote note

  // ...
}
```
### g
> src/runtime/runtime2.go

```go
type g struct {
	// stack 相关
	stack       stack   // offset known to runtime/cgo
	stackguard0 uintptr // offset known to liblink
	stackguard1 uintptr // offset known to liblink

	// ...

	// 当前关联的 M
	m              *m      // current m; offset known to arm liblink

	// ...

	// 参数地址
	param          unsafe.Pointer // passed parameter on wakeup

	// ...

	// GC 相关
	preemptscan    bool       // preempted g does scan for gc
	gcscandone     bool       // g has scanned stack; protected by _Gscan bit in status
	gcscanvalid    bool       // false at start of gc cycle, true if G has not run since last scan; TODO: remove?

	// ...

	waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order

	// ...

	// 定时器
	timer          *timer         // cached timer for time.Sleep
	// select
	selectDone     uint32         // are we participating in a select and did someone win the race?

	// Per-G GC state
	gcAssistBytes int64
}

```

### m
> src/runtime/runtime2.go

```go
type m struct {
	// ...

	// P 相关
	p             puintptr // attached p for executing go code (nil if not executing go code)
	nextp         puintptr
	oldp          puintptr // the p that was attached before executing a syscall

	// ...

	// M 自旋状态
	spinning      bool // m is out of work and is actively looking for work
	blocked       bool // m is blocked on a note
	inwb          bool // m is executing a write barrier
	newSigstack   bool // minit on C thread called sigaltstack
	printlock     int8

	// cgo
	incgo         bool   // m is executing a cgo call
	// 等待 G 的数量
	freeWait      uint32 // if == 0, safe to free g0 and delete m (atomic)
	// ...

	// cgo
	ncgocall      uint64      // number of cgo calls in total
	ncgo          int32       // number of cgo calls currently in progress
	cgoCallersUse uint32      // if non-zero, cgoCallers in use temporarily
	cgoCallers    *cgoCallers // cgo traceback if crashing in cgo call

	park          note

	createstack   [32]uintptr    // stack that created this thread.
	lockedExt     uint32         // tracking for external LockOSThread
	lockedInt     uint32         // tracking for internal lockOSThread
	nextwaitm     muintptr       // next m waiting for lock
	// ...

	// OS 线程
	thread        uintptr // thread handle
	// 待释放的 M
	freelink      *m      // on sched.freem

	// ...
}

```


### p
> src/runtime/runtime2.go

```go
type p struct {
	// 相关联的 M
	m           muintptr   // back-link to associated m (nil if idle)
	mcache      *mcache
	racectx     uintptr

  // G 本地队列
	// Queue of runnable goroutines. Accessed without lock.
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr

	// 下个可被执行的 G
	runnext guintptr

	// Available G's (status == Gdead)
	gFree struct {
	   gList
	   n int32
	}

	// sudog 相关
	sudogcache []*sudog
	sudogbuf   [128]*sudog

	palloc persistentAlloc // per-P to avoid mutex

	// Per-P GC state
	gcAssistTime         int64 // Nanoseconds in assistAlloc
	gcFractionalMarkTime int64 // Nanoseconds in fractional mark worker
	gcBgMarkWorker       guintptr
	gcMarkWorkerMode     gcMarkWorkerMode

	// gcMarkWorkerStartTime is the nanotime() at which this mark
	// worker started.
	gcMarkWorkerStartTime int64

	// gcw is this P's GC work buffer cache. The work buffer is
	// filled by write barriers, drained by mutator assists, and
	// disposed on certain GC state transitions.
	gcw gcWork
}
```

## 生命周期
Go 启动引导文件是汇编写的，根据不同的系统，引导文件不同；文件存放于 runtime/asm_*.s:

``` arm
CALL	runtime·args(SB)
CALL	runtime·osinit(SB)
CALL	runtime·schedinit(SB)

// create a new goroutine to start program
MOVQ	$runtime·mainPC(SB), AX		// entry
PUSHQ	AX
PUSHQ	$0			// arg size
CALL	runtime·newproc(SB)
POPQ	AX
POPQ	AX

// start this M
CALL	runtime·mstart(SB)

CALL	runtime·abort(SB)	// mstart should never return
RET
```

### runtime.osinit
> 根据 OS 不同，在不同的文件中：src/runtime/os_*.go

获取 cpu 的核数

```go
func osinit() {
	ncpu = getproccount()
}
```

### runtime·schedinit
> src/runtime/proc.go

schedt 结构初始化:
1. 设置 M 的最大数量为 10000
1. stack、args、envs、gc 等初始化
1. P 数量初始化；此时 STW，sched 上锁，生成的 P 会被放入 `schedt.pidle` 中

```go
func schedinit(){
	// ...
	// M 最多为10000个
	sched.maxmcount = 10000

	// 根据 CPU 核数、GOMAXPROCS调整 procs
	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
	// 根据 procs 调整 P 的数量
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
	// ...
}
```

### runtime·newproc
> src/runtime/proc.go

创建一个 G

``` go
func newproc(size int32,fn *funcval){
	// ...
	systemstack(func() {
	  newproc1(fn, (*uint8)(argp), siz, gp, pc)
	})
	// ...
}
func newproc1(fn *funcval, argp *uint8, narg int32, callergp *g, callerpc uintptr) {
	// ...

	// 从 gFree 中获取 G；若本地为空，则从全局拿32个放入本地
	newg := gfget(_p_)

	// ...

	// 把 G 放入到 p.runnext，若本地队列满了(>256),则从本地队列拿一半+当前 G 放入全局队列)
	runqput(_p_, newg, true)
	// 若有空闲 P， M0 已启动，并且没有自旋 M（启动过程中不会执行）
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && mainStarted {
		// 尝试再添加一个 P 来执行 G
		wakep()
	}
}
// 引导过程中不会执行以下函数
func wakep(){
	// ...
	// 获取一个空闲 P，M；并绑定
	startm(nil, true)
}
func startm(_p_ *p, spinning bool) {
	lock(&sched.lock)
  	if _p_ == nil {
    	// 从 schedt.pidle 中获取空闲的 P
		_p_ = pidleget()
    	// ...
  	}
	// ...
	// 从 schedt.midle 中获取空闲的 M
	mp := mget()
	unlock(&sched.lock)
	if mp == nil {
	// ...
	// 创建新的 M（ OS thread）
		newm(fn, _p_)
		return
	}
}
```
### runtime·mstart
> src/runtime/proc.go

找到一个 P，并绑定一个 M，然后执行 G
```go
func mstart(){
	// 获取当前 G
	_g_ := getg()

	// ...
	mstart1()

	// ...
	mexit(osStack)
}

func mstart1(){
	// 获取当前 G
	_g_ := getg()

	// ...

	// 启动 M0
	if _g_.m == &m0 {
		mstartm0()
	}

	// ...

	if _g_.m != &m0 {
		// 关联 P 和 M
		acquirep(_g_.m.nextp.ptr())
		_g_.m.nextp = 0
	}
	// 找到一个可执行的 G，并执行
	schedule()
}

func schedule(){
	_g_ := getg()

	// // ...

	if gp == nil {
		// 1/61 的几率从全局取 G，调用 rungput 放入本地队列
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
		// 从本地队列取 G
		gp, inheritTime = runqget(_g_.m.p.ptr())
		if gp != nil && _g_.m.spinning {
			throw("schedule: spinning with local work")
		}
	}
	if gp == nil {
		// 循环：本地队列 -> 全局队列 -> poll network -> 偷其他 P 一半 G -> 检查 P -> GC
		gp, inheritTime = findrunnable() // blocks until work is available
	}

	// 若是自旋 M, 则停止当前 M，然后调用 wakep 开启新的自旋 M
	if _g_.m.spinning {
		resetspinning()
	}

	// 执行 G，从 G0 -> G
	execute(gp, inheritTime)
}

func main(){
	g := getg()

	// ...

	if sys.PtrSize == 8 {
		maxstacksize = 1000000000
	} else {
		maxstacksize = 250000000
	}

  if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
    // 开启 sysmon 监控线程，若 P 处于 syscall，则获取一个 M 拿走 P 下的 G
		systemstack(func() {
			newm(sysmon, nil)
		})
	}
	// ...

	// 执行用户的 init()
	fn := main_init // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime

	// ...

	// 执行 main.main()
	fn = main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime

	// ...

	// 退出
	exit(0)
}
```
