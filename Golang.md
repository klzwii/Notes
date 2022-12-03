## PMG
Golang采用PMG作为线程模型。
G: Goroutine 
<details>
  <summary>go runtime G的结构</summary>

  ```Go
  type g struct {
	// Stack parameters.
	// stack describes the actual stack memory: [stack.lo, stack.hi).
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	// stackguard1 is the stack pointer compared in the C stack growth prologue.
	// It is stack.lo+StackGuard on g0 and gsignal stacks.
	// It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
	stack       stack   // offset known to runtime/cgo
	stackguard0 uintptr // offset known to liblink
	stackguard1 uintptr // offset known to liblink

	_panic    *_panic // innermost panic - offset known to liblink
	_defer    *_defer // innermost defer
	m         *m      // current m; offset known to arm liblink
	sched     gobuf
	syscallsp uintptr // if status==Gsyscall, syscallsp = sched.sp to use during gc
	syscallpc uintptr // if status==Gsyscall, syscallpc = sched.pc to use during gc
	stktopsp  uintptr // expected sp at top of stack, to check in traceback
	// param is a generic pointer parameter field used to pass
	// values in particular contexts where other storage for the
	// parameter would be difficult to find. It is currently used
	// in three ways:
	// 1. When a channel operation wakes up a blocked goroutine, it sets param to
	//    point to the sudog of the completed blocking operation.
	// 2. By gcAssistAlloc1 to signal back to its caller that the goroutine completed
	//    the GC cycle. It is unsafe to do so in any other way, because the goroutine's
	//    stack may have moved in the meantime.
	// 3. By debugCallWrap to pass parameters to a new goroutine because allocating a
	//    closure in the runtime is forbidden.
	param        unsafe.Pointer
	atomicstatus uint32
	stackLock    uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
	goid         int64
	schedlink    guintptr
	waitsince    int64      // approx time when the g become blocked
	waitreason   waitReason // if status==Gwaiting

	preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt
	preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule
	preemptShrink bool // shrink stack at synchronous safe point

	// asyncSafePoint is set if g is stopped at an asynchronous
	// safe point. This means there are frames on the stack
	// without precise pointer information.
	asyncSafePoint bool

	paniconfault bool // panic (instead of crash) on unexpected fault address
	gcscandone   bool // g has scanned stack; protected by _Gscan bit in status
	throwsplit   bool // must not split stack
	// activeStackChans indicates that there are unlocked channels
	// pointing into this goroutine's stack. If true, stack
	// copying needs to acquire channel locks to protect these
	// areas of the stack.
	activeStackChans bool
	// parkingOnChan indicates that the goroutine is about to
	// park on a chansend or chanrecv. Used to signal an unsafe point
	// for stack shrinking. It's a boolean value, but is updated atomically.
	parkingOnChan uint8

	raceignore     int8     // ignore race detection events
	sysblocktraced bool     // StartTrace has emitted EvGoInSyscall about this goroutine
	tracking       bool     // whether we're tracking this G for sched latency statistics
	trackingSeq    uint8    // used to decide whether to track this G
	runnableStamp  int64    // timestamp of when the G last became runnable, only used when tracking
	runnableTime   int64    // the amount of time spent runnable, cleared when running, only used when tracking
	sysexitticks   int64    // cputicks when syscall has returned (for tracing)
	traceseq       uint64   // trace event sequencer
	tracelastp     puintptr // last P emitted an event for this goroutine
	lockedm        muintptr
	sig            uint32
	writebuf       []byte
	sigcode0       uintptr
	sigcode1       uintptr
	sigpc          uintptr
	gopc           uintptr         // pc of go statement that created this goroutine
	ancestors      *[]ancestorInfo // ancestor information goroutine(s) that created this goroutine (only used if debug.tracebackancestors)
	startpc        uintptr         // pc of goroutine function
	racectx        uintptr
	waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
	cgoCtxt        []uintptr      // cgo traceback context
	labels         unsafe.Pointer // profiler labels
	timer          *timer         // cached timer for time.Sleep
	selectDone     uint32         // are we participating in a select and did someone win the race?

	// goroutineProfiled indicates the status of this goroutine's stack for the
	// current in-progress goroutine profile
	goroutineProfiled goroutineProfileStateHolder

	// Per-G GC state

	// gcAssistBytes is this G's GC assist credit in terms of
	// bytes allocated. If this is positive, then the G has credit
	// to allocate gcAssistBytes bytes without assisting. If this
	// is negative, then the G must correct this by performing
	// scan work. We track this in bytes to make it fast to update
	// and check for debt in the malloc hot path. The assist ratio
	// determines how this corresponds to scan work debt.
	gcAssistBytes int64
}
  ```
</details>

P: Processor

<details>
  <summary>go runtime P的结构</summary>

  ```Go
  type p struct {
	id          int32
	status      uint32 // one of pidle/prunning/...
	link        puintptr
	schedtick   uint32     // incremented on every scheduler call
	syscalltick uint32     // incremented on every system call
	sysmontick  sysmontick // last tick observed by sysmon
	m           muintptr   // back-link to associated m (nil if idle)
	mcache      *mcache
	pcache      pageCache
	raceprocctx uintptr

	deferpool    []*_defer // pool of available defer structs (see panic.go)
	deferpoolbuf [32]*_defer

	// Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
	goidcache    uint64
	goidcacheend uint64

	// Queue of runnable goroutines. Accessed without lock.
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	// runnext, if non-nil, is a runnable G that was ready'd by
	// the current G and should be run next instead of what's in
	// runq if there's time remaining in the running G's time
	// slice. It will inherit the time left in the current time
	// slice. If a set of goroutines is locked in a
	// communicate-and-wait pattern, this schedules that set as a
	// unit and eliminates the (potentially large) scheduling
	// latency that otherwise arises from adding the ready'd
	// goroutines to the end of the run queue.
	//
	// Note that while other P's may atomically CAS this to zero,
	// only the owner P can CAS it to a valid G.
	runnext guintptr

	// Available G's (status == Gdead)
	gFree struct {
		gList
		n int32
	}

	sudogcache []*sudog
	sudogbuf   [128]*sudog

	// Cache of mspan objects from the heap.
	mspancache struct {
		// We need an explicit length here because this field is used
		// in allocation codepaths where write barriers are not allowed,
		// and eliminating the write barrier/keeping it eliminated from
		// slice updates is tricky, moreso than just managing the length
		// ourselves.
		len int
		buf [128]*mspan
	}

	tracebuf traceBufPtr

	// traceSweep indicates the sweep events should be traced.
	// This is used to defer the sweep start event until a span
	// has actually been swept.
	traceSweep bool
	// traceSwept and traceReclaimed track the number of bytes
	// swept and reclaimed by sweeping in the current sweep loop.
	traceSwept, traceReclaimed uintptr

	palloc persistentAlloc // per-P to avoid mutex

	_ uint32 // Alignment for atomic fields below

	// The when field of the first entry on the timer heap.
	// This is updated using atomic functions.
	// This is 0 if the timer heap is empty.
	timer0When uint64

	// The earliest known nextwhen field of a timer with
	// timerModifiedEarlier status. Because the timer may have been
	// modified again, there need not be any timer with this value.
	// This is updated using atomic functions.
	// This is 0 if there are no timerModifiedEarlier timers.
	timerModifiedEarliest uint64

	// Per-P GC state
	gcAssistTime         int64 // Nanoseconds in assistAlloc
	gcFractionalMarkTime int64 // Nanoseconds in fractional mark worker (atomic)

	// limiterEvent tracks events for the GC CPU limiter.
	limiterEvent limiterEvent

	// gcMarkWorkerMode is the mode for the next mark worker to run in.
	// That is, this is used to communicate with the worker goroutine
	// selected for immediate execution by
	// gcController.findRunnableGCWorker. When scheduling other goroutines,
	// this field must be set to gcMarkWorkerNotWorker.
	gcMarkWorkerMode gcMarkWorkerMode
	// gcMarkWorkerStartTime is the nanotime() at which the most recent
	// mark worker started.
	gcMarkWorkerStartTime int64

	// gcw is this P's GC work buffer cache. The work buffer is
	// filled by write barriers, drained by mutator assists, and
	// disposed on certain GC state transitions.
	gcw gcWork

	// wbBuf is this P's GC write barrier buffer.
	//
	// TODO: Consider caching this in the running G.
	wbBuf wbBuf

	runSafePointFn uint32 // if 1, run sched.safePointFn at next safe point

	// statsSeq is a counter indicating whether this P is currently
	// writing any stats. Its value is even when not, odd when it is.
	statsSeq uint32

	// Lock for timers. We normally access the timers while running
	// on this P, but the scheduler can also do it from a different P.
	timersLock mutex

	// Actions to take at some time. This is used to implement the
	// standard library's time package.
	// Must hold timersLock to access.
	timers []*timer

	// Number of timers in P's heap.
	// Modified using atomic instructions.
	numTimers uint32

	// Number of timerDeleted timers in P's heap.
	// Modified using atomic instructions.
	deletedTimers uint32

	// Race context used while executing timer functions.
	timerRaceCtx uintptr

	// maxStackScanDelta accumulates the amount of stack space held by
	// live goroutines (i.e. those eligible for stack scanning).
	// Flushed to gcController.maxStackScan once maxStackScanSlack
	// or -maxStackScanSlack is reached.
	maxStackScanDelta int64

	// gc-time statistics about current goroutines
	// Note that this differs from maxStackScan in that this
	// accumulates the actual stack observed to be used at GC time (hi - sp),
	// not an instantaneous measure of the total stack size that might need
	// to be scanned (hi - lo).
	scannedStackSize uint64 // stack size of goroutines scanned by this P
	scannedStacks    uint64 // number of goroutines scanned by this P

	// preempt is set to indicate that this P should be enter the
	// scheduler ASAP (regardless of what G is running on it).
	preempt bool

	// Padding is no longer needed. False sharing is now not a worry because p is large enough
	// that its size class is an integer multiple of the cache line size (for any of our architectures).
}
  ```
</details>

M: Machine

<details>
  <summary>go runtime M的结构</summary>

  ```Go
  type m struct {
	g0      *g     // goroutine with scheduling stack
	morebuf gobuf  // gobuf arg to morestack
	divmod  uint32 // div/mod denominator for arm - known to liblink
	_       uint32 // align next field to 8 bytes

	// Fields not known to debuggers.
	procid        uint64            // for debuggers, but offset not hard-coded
	gsignal       *g                // signal-handling g
	goSigStack    gsignalStack      // Go-allocated signal handling stack
	sigmask       sigset            // storage for saved signal mask
	tls           [tlsSlots]uintptr // thread-local storage (for x86 extern register)
	mstartfn      func()
	curg          *g       // current running goroutine
	caughtsig     guintptr // goroutine running during fatal signal
	p             puintptr // attached p for executing go code (nil if not executing go code)
	nextp         puintptr
	oldp          puintptr // the p that was attached before executing a syscall
	id            int64
	mallocing     int32
	throwing      throwType
	preemptoff    string // if != "", keep curg running on this m
	locks         int32
	dying         int32
	profilehz     int32
	spinning      bool // m is out of work and is actively looking for work
	blocked       bool // m is blocked on a note
	newSigstack   bool // minit on C thread called sigaltstack
	printlock     int8
	incgo         bool   // m is executing a cgo call
	freeWait      uint32 // if == 0, safe to free g0 and delete m (atomic)
	fastrand      uint64
	needextram    bool
	traceback     uint8
	ncgocall      uint64      // number of cgo calls in total
	ncgo          int32       // number of cgo calls currently in progress
	cgoCallersUse uint32      // if non-zero, cgoCallers in use temporarily
	cgoCallers    *cgoCallers // cgo traceback if crashing in cgo call
	park          note
	alllink       *m // on allm
	schedlink     muintptr
	lockedg       guintptr
	createstack   [32]uintptr // stack that created this thread.
	lockedExt     uint32      // tracking for external LockOSThread
	lockedInt     uint32      // tracking for internal lockOSThread
	nextwaitm     muintptr    // next m waiting for lock
	waitunlockf   func(*g, unsafe.Pointer) bool
	waitlock      unsafe.Pointer
	waittraceev   byte
	waittraceskip int
	startingtrace bool
	syscalltick   uint32
	freelink      *m // on sched.freem

	// these are here because they are too large to be on the stack
	// of low-level NOSPLIT functions.
	libcall   libcall
	libcallpc uintptr // for cpu profiler
	libcallsp uintptr
	libcallg  guintptr
	syscall   libcall // stores syscall parameters on windows

	vdsoSP uintptr // SP for traceback while in VDSO call (0 if not in call)
	vdsoPC uintptr // PC for traceback while in VDSO call

	// preemptGen counts the number of completed preemption
	// signals. This is used to detect when a preemption is
	// requested, but fails. Accessed atomically.
	preemptGen uint32

	// Whether this is a pending preemption signal on this M.
	// Accessed atomically.
	signalPending uint32

	dlogPerM

	mOS

	// Up to 10 locks held by this m, maintained by the lock ranking code.
	locksHeldLen int
	locksHeld    [10]heldLockInfo
}
  ```
</details>

## Golang内存模型 <cited>[[1]]</cited>

### Advice from Go Team

修改被多个goroutine同时访问的数据的程序必须将这种访问序列化。  
因此，你需要用Channel或其他比如sync和sync/atomic包中的同步原语来保护这种数据。
如果你坚持阅读本文档的其余部分来理解你的程序的行为，那么你就太聪明了。  
不要这么聪明。

### 内存模型

内存模型通过内存操作的四种性质来定义。

- 它的种类 -- 一个普通的数据读取、一个普通的数据写入、一个同步操作（如一个原子数据访问）、一个互斥操作、或一个通道操作。
- 它在程序中的位置。
- 被访问的内存位置或变量，以及该操作所读取或写入的值。
- 一些操作是read-like，包括读、原子读、Mutex加锁和Channel接收。其他操作是write-like，包括写、原子写、Mutex解锁、Channel发送和Channel关闭。有些操作，例如CAS(compare and swap)，既是读操作，又是写操作

一个goroutine的执行被视为由单一goroutine执行的一组内存操作。

内存操作遵循如下原则
  1. 每个goroutine从内存中读出和写入的值必须对应于该goroutine执行的正确顺序。执行必须符合Go语言规范为Go的控制流结构所规定的顺序要求，以及表达式的评估顺序。<cited>[[2]]</cited>  Go程序的执行被视为一组goroutine的执行，和一个映射W。该映射指定了每个read-like操作所读取到的write-like。
  2. 对于一个给定的程序执行，映射W，当限于同步操作时，必须可以用一些隐式的与同步操作发生的次序以及所操作的值的次序来解释。Synchronized before关系是部分同步操作之间的先后顺序。如果一个同步的read-like操作r可以读取到一个同步的write-like操作w写入的值，那么我们就把w称为synchronized before r。
  3. 对于一个内存位置x上的普通（非同步）读r，W(r)=w代表w必被r可见，其中可见意味着以下两个条件都成立。
     1. w发生在r之前。
     2. w和r之间不存在w'对x进行写入  

内存位置x上的读写数据争用由x上的read-like操作r和x上的write-like操作w组成，其中至少有一个是非同步的，这两个操作没有可用的happens before原则（也就是说，r和w的发生次序是未定义的）。

内存位置x上的写写数据争用由x上的两个write-like内存操作w和w'组成，其中至少有一个是不同步的，这两个操作也没有可用的happens before原则。

Go语言内存模型遵守DRF-SC，
即如果程序是DRF的(Data-Race Free 无数据争用)，那么SC将被满足。
SC(Sequential Consistency): 并行的内存访问顺序，与程序指定顺序完全相同

### Synchronizes Before

- 创建goroutine的语句Synchronizes Before goroutine的执行
- 对channel的发送Synchronizes Before对channel的接受完成
- 对无缓冲channel的接受Synchronizes Before此次发送完成
- 对于容量为C的channel，第k次接受Synchronizes Before k+C次发送完成
- 对于sync.Mutex和sync.RWMutex l， 任取n < m, 第n次l.UnLock，Synchronizes Before 第m次l.Lock()
- once.Do(f)的完成Synchronizes Before该函数的返回
- 对于sync/atomics，对值的操作Synchronizes Before该值对当前goroutine可见
- 对SetFinalizer的调用Synchronizes Before对Finalizer的执行

### 内存管理 <cited>runtime/malloc.go</cited>
主分配器是以页为单位工作的。
分配小对象时（小于等于32kB）被向上取整到大约70个大小类别中的一个。
每个大小类别都有自己的空闲对象集，一个集合中的对象大小完全相等。
任何空闲的内存页都可以被分割成相同大小的一组对象，然后用一个标识对应对象是否被使用的位图来管理。

分配器的数据结构如下:

- fixalloc：一个分配固定大小的堆外对象的空闲列表(free-list)分配器，被用来管理分配器使用的存储空间。
- mheap：malloc使用的堆，以页（8192字节）的粒度管理。
- mspan: 由mheap管理的一系列已被使用的内存页。
- mcentral: 存有某一大小类别的全部mspan。
- mcache: 一个有空闲空间的mspans的每个P独占的缓存。  
- mstats：分配统计。

小对象的分配遵循如下的缓存层次:
1. 将大小向上取整到大小类中的一个。并在这个P对应的mcache中寻找相应的mspan。
扫描mspan的空闲位图来找到一个空闲槽。如果有一个空闲的槽，就分配它。
这一切都可以在不获取锁的情况下完成。

1. 如果mspan没有空闲槽位，从mcentral的空闲槽位中获取一个新的mspan。
从mcentral的所需大小的mspans列表中获得一个新的mspan
有空闲空间的类。获得一个完整的mspan可以分摊锁定mcentral的成本。
mcentral。

1. 如果mcentral的mspan列表是空的，就从mheap中获取一系列内存也。
从mheap中获取一个页面，用于构建mspan。

1. 如果mheap是空的或者没有足够大的一系列内存页。go将会从操作系统中分配一系列新的内存页（至少1MB）。
分配一个很大的内存均摊了系统调用的成本。

清理一个mspan并释放其上对象的过程遵循相似层次结构。

1. 如果mspan是为了满足新的空间分配从而被清理，那么它将被返回到mcache以满足分配。

2. 否则，如果mspan中仍有没有被释放的对象。它将被放置在与mspan大小类相同的mcentral的空闲列表中。

3. 最后，如果mspan中的所有对象都被释放，那么mspan的的页面被返回到mheap，此时mspan被标记为dead。

分配和释放一个大对象时，将会绕过mcache和mcentral，直接使用mheap。

如果mspan.needzero为false，那么mspan中的空闲对象都已经被提前置空。
如果needzero为true，则对象在分配时被置空。
延迟置空有如下的好处:

1. 栈帧分配完全不需要置空。

2. 它更好的利用了时间局部性，因为程序很可能要对这块内存进行写入。

3. 我们不需要置空之后不会再重复使用的页面。

虚拟内存布局

堆由一组arena组成，arena的大小在64位上是64MB，在32位上是4MB。
每个arena的起始地址与arena的大小对齐。

每个arena都有一个相关的heapArena对象，存储该arena的元数据:
- 该arena中所有word的位图
- 该arena中所有内存页对应的spanmap。
heapArena对象在堆外分配。

由于arena内存地址是对齐的，地址空间可以被看作是一个一系列的arena块。
arena map（mheap_.arena）将从arena块的编号映射到*heapArena。
不由go堆管理的地址空间将被映射为nil。
arena map是由一个 L1 arena map和许多 L2 arena map组成的两维数组。
然而，由于arena很大，在许多架构上，arena map仅由一个单独的大的L2 map组成。

arena map覆盖整个可能的地址空间，允许Go堆使用地址空间的任何部分。
分配器试图保持arena连续，以便大的span（以及通过其分配的大的对象）可以跨arena。
mcache:
mcache通过spcanClass进行mspan的索引。
spanClass的类型是uint8，最低位表示该mspan是否包含指针(不包含指针可以减少GC对其的扫描)，其它位表示他所索引的mspan的大小类。

tinyalloc:
Go对不包含指针的小对象(<=16B)有一些分配优化。mcache会直接通过保存的一个大小为16B的block为其分配对应的空间，这样会让多个不同的小对象存在于同一个block中从而增加空间利用率。

## 相关源码

### //go:nosplit
在runtime包可以看到很多该标签 该标签禁止紧随其后的函数进行栈溢出检查，该检查在golang调度器尝试抢占某一goroutine是必须的，因此带有该标签的函数无法被抢占。

### casgstatus
```Go
// 通过cas进行进行goroutine的状态切换
func casgstatus(gp *g, oldval, newval uint32) {
	if (oldval&_Gscan != 0) || (newval&_Gscan != 0) || oldval == newval {
		systemstack(func() {
			print("runtime: casgstatus: oldval=", hex(oldval), " newval=", hex(newval), "\n")
			throw("casgstatus: bad incoming values")
		})
	} // 检查是否会更改gc状态/更改前后值不一样
    
    const yieldDelay = 5 * 1000
	var nextYield int64

	// 通过CAS进行状态改变 如果状态不一致代表gc正在扫描该goroutine 等待gc完成
	for i := 0; !atomic.Cas(&gp.atomicstatus, oldval, newval); i++ {
		if oldval == _Gwaiting && gp.atomicstatus == _Grunnable {
			throw("casgstatus: waiting for Gwaiting but is Grunnable")
		}
		if i == 0 {
			nextYield = nanotime() + yieldDelay
		}
		if nanotime() < nextYield {
			for x := 0; x < 10 && gp.atomicstatus != oldval; x++ {
				procyield(1)
			}
		} else {
			osyield()
			nextYield = nanotime() + yieldDelay/2
		}
	}
}
```

### sematable
sematable是Golang中信号量的实现

```Go
type semaRoot struct {
	lock  mutex
	treap *sudog // root of balanced tree of unique waiters.
	nwait uint32 // Number of waiters. Read w/o the lock.
}
type semTable [semTabSize]struct {
	root semaRoot
	pad  [cpu.CacheLinePadSize - unsafe.Sizeof(semaRoot{})]byte
}
func (t *semTable) rootFor(addr *uint32) *semaRoot {
	return &t[(uintptr(unsafe.Pointer(addr))>>3)%semTabSize].root
}
```

### sudog

sudog表示一个正在等待(channel/semaphore...)的goroutine
```Go
type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g *g

	next *sudog
	prev *sudog
	elem unsafe.Pointer // data element (may point to stack)

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32

	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool

	// success indicates whether communication over channel c
	// succeeded. It is true if the goroutine was awoken because a
	// value was delivered over channel c, and false if awoken
	// because c was closed.
	success bool

	parent   *sudog // semaRoot binary tree
	waitlink *sudog // g.waiting list or semaRoot
	waittail *sudog // semaRoot
	c        *hchan // channel
```
## Go GC

### gc 消耗模型

1. GC只涉及两种资源。CPU时间，和物理内存。

2. GC的内存成本由存活堆内存、标记阶段前分配的新堆内存和元数据占用组成。其中元数据即使与前两者消耗成正比，但相比之下也很小。

3. GC的CPU消耗被分为为每个周期的固定消耗，以及与活堆大小成比例的额外消耗。

## 常见golang优化手段

### fastslow path
通过将快速常见轻量逻辑的分支与不常见的分支分离在不同的函数，保证快速分支能够被inline，减少函数调用时间。

### false-sharing emit
对于需要被并发访问的结构体数组，对其中每个结构体增加padding确保其对cpu的cacheline的大小对齐，防止出现false-sharing。

[1]: https://go.dev/ref/mem
[2]: https://go.dev/ref/spec