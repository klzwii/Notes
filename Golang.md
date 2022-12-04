## PMG
Golang采用PMG作为线程模型。
G: Goroutine  
P: Processor  
M: Machine

### scheduler overview

#### 工作线程自旋的定义

某一工作线程M已经结束了对应的工作，同时全局的任务队列以及netpoller中也没有对应的任务。
m.spinning和sched.nmspinning分别标识了当前工作线程是否自旋，以及全局的自旋工作线程的数量。

#### 新线程的建立

当有一个空闲的P并且没有处于自旋状态的工作线程时
只要在我们将某一P唤醒的时候存在自旋的工作线程，那我们就不需要释放一个新的工作线程，
最后一个自旋的工作线程有责任在找到工作结束自旋状态前申请一个新的自旋工作线程。

任务的提交遵循以下模式
1. 将任务提交到本地任务队列/timer heap/GC
2. StoreLoad barrier
3. 检查sched.nmspinning

自旋状态的解除遵循以下模式
1. 减少runtime.nmspinning
2. StoreLoad barrier
3. 查看每个P的本地工作队列以及全局工作队列来找到可执行任务

可执行的任务有下面的几种情况
- 准备就绪的Goroutine
- 新老定时器
- GC任务
  
#### 抢占

goroutine可以在任意一个安全点被抢占，安全点分为如下几类:
- 阻塞安全点：descheduled，同步阻塞。系统调用
- 同步安全点：主动查看是否存在抢占请求
- 异步安全点：用户代码中的任意一个可以暂停并且接受栈和寄存器扫描来获取stack roots的地方，runtime可以通过signal来使goroutine停在一个异步安全点  
阻塞安全点以及同步安全点下，goroutine的cpu消耗被降到最小并且GC拥有该Goroutine的整个stack的完整信息。这样可以在最小内存消耗下对goroutine进行调度停止并精确扫描它的栈  
同步安全点通过重写函数开始前的stack边界检查来实现。为了能够在下一个同步安全点抢占goroutine，runtime将需要抢占的goroutine的栈的边界进行更改来使下一次的边界检查失败，并进入动态栈增长的实现，这样goroutine就可以发现已经发起的抢占并重定向到抢占处理。  
异步安全点的抢占通过OS相关的机制实现线程的暂停(e.g.信号)，并在暂停后观察该goroutine是否位于一部安全点。因为通常线程暂停都是异步的，因此还会再次检查想要抢占的goroutine是否依然试图抢占。如果上述的条件都被满足了，runtime会更改信号的上下文让收到信号的线程就像刚调用asncPreempt一样，之后再继续执行。asyncPreempt清空全部的寄存器并执行调度器。  

## Golang 内嵌汇编(amd64)

- #include "go_asm.h" 在go build时，会产生包含当前package的一些符号定义的文件（常量，结构体大小，结构体成员偏移量）。
  如const bufSize = 1024, 变为const_bufSize
  ```Go
  type reader struct {
	buf [bufSize]byte
	r   int
  }
  ```
  将会生成：
  - reader__size reader大小
  - reader_buf buf的偏移量
  - reader_r r的偏移量
- #include "go_tls.h" 是runtime中的一个头文件，引入该头文件可以获取当前的goroutine
  常见用法如下:
  ```GO
  get_tls(CX) 
  MOVL	g(CX), AX     // Move g into AX.
  MOVL	g_m(AX), BX   // Move g.m into BX.
  ```
- asm中定义了一些自定义的寄存器:
  - SB 静态的基址 使用如`TEXT symbol(SB)`定义了一个函数，其中symbol的值就是该函数针对SB的偏移量。可以使用`CALL symbol(SB)`来进行函数调用
  - PC 程序计数器 指向当前指令地址
  - FP 用来索引函数的参数,0(FP)标识第一个参数，8(FP)标识第二个参数。注意在使用时应该为其添加名称如arg_1+0(FP)用来标识对应变量的作用。
  - SP 栈顶指针 用来索引当前栈的变量和FP一样需要命名 注意对于架构中本身就包含SP寄存器的情况，不带名字的指令标识原本的硬件寄存器比如amd 8(SP)中的SP标识真正的寄存器中的值
- 函数定义`TEXT a_simple_function<>(SB),NOSPLIT,$0-8`，标识a_simple_function只能在当前文件可见，NOSPLIT为flag<cited>[[3]]</cited>，$0-8为当前函数本地大小为0，接受8byte的参数

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

### malloc源码分析

#### GoAssist

每个goroutine都包含一个gcAssistBytes标识当前goroutine对于GC的"负债", 在每次mallocgc调用时，该值会被首先减去本次需要分配的内存大小，当负债小于0时，该goroutine需要堆GC进行协助，之后再进行内存分配 。

## 相关源码

### system stack

普通的goroutines在一个小堆栈上运行。每个普通的Go函数在进入时都会检查它是否有足够的堆栈空间来运行。如果它没有,它将分配一个新的更大的堆栈，并将现有的堆栈复制到新的空间。但当然，这个复制过程所需要的代码也应该在某个栈上运行，而这显然不能是已经用完空间的栈。

系统堆栈是操作系统为新线程创建的堆栈。
它比goroutine堆栈大。栈的复制就是在这个堆栈上运行的。
每个线程仅有一个系统栈，而不是每个goroutine都有一个系统栈。
一般来说，goroutine比线程多得多。

一些垃圾收集步骤也是在系统栈上运行的，以做到泾渭分明的分离GC正在执行的代码和检查堆栈中的存活对象。

在系统堆栈上运行的函数不能像普通函数一样增加栈空间，同时也不能被go的调度器抢占。
所以只能被用来执行一些短小独立的函数。

#### amd64汇编分析 src/runtime/asm_amd64.s

```GO
// func systemstack(fn func())
TEXT runtime·systemstack(SB), NOSPLIT, $0-8
	MOVQ	fn+0(FP), DI	// 0(FP)是函数的第一个参数，即需要在系统栈执行的函数
	get_tls(CX) 
	MOVQ	g(CX), AX	// AX = g
	MOVQ	g_m(AX), BX	// BX = m 拿到当前的g和m

	CMPQ	AX, m_gsignal(BX) // 检查是否是信号处理goroutine 是则不跳转
	JEQ	noswitch

	MOVQ	m_g0(BX), DX	// 检查是否是信号处理g0 是则不跳转
	CMPQ	AX, DX
	JEQ	noswitch

	CMPQ	AX, m_curg(BX) // m当前执行的goroutine 是否是当前的g 不是则异常
	JNE	bad

	// switch stacks
	// save our state in g->sched. Pretend to
	// be systemstack_switch if the G stack is scanned.
	CALL	gosave_systemstack_switch<>(SB) // 将当前栈数据存入g.sched

	// 切换至g0
	MOVQ	DX, g(CX)  
	MOVQ	DX, R14 // 将R14寄存器设置为g0 该寄存器为calle_saved
	MOVQ	(g_sched+gobuf_sp)(DX), BX
	MOVQ	BX, SP //将m存到栈顶

	// 调用func
	MOVQ	DI, DX
	MOVQ	0(DI), DI
	CALL	DI

	// switch back to g
	get_tls(CX)
	MOVQ	g(CX), AX
	MOVQ	g_m(AX), BX
	MOVQ	m_curg(BX), AX
	MOVQ	AX, g(CX)
	MOVQ	(g_sched+gobuf_sp)(AX), SP
	MOVQ	$0, (g_sched+gobuf_sp)(AX)
	RET

noswitch:
	// already on m stack; tail call the function
	// Using a tail call here cleans up tracebacks since we won't stop
	// at an intermediate systemstack.
	MOVQ	DI, DX
	MOVQ	0(DI), DI
	JMP	DI

bad:
	// Bad: g is not gsignal, not g0, not curg. What is it?
	MOVQ	$runtime·badsystemstack(SB), AX
	CALL	AX
	INT	$3
```

### 禁止抢占
`mp := acquirem()` 将m锁死在当前g上

### //go:nosplit
在runtime包可以看到很多该标签 该标签禁止紧随其后的函数进行栈溢出检查，该检查在golang调度器尝试抢占某一goroutine是必须的，因此带有该标签的函数无法被抢占。

### //go:noescape
被该注释标识的函数没有函数体，只有声明代表该函数不通过Go实现。 表明该函数不会让任何传递进入的指针逃逸到堆上或者返回值中，以便进行逃逸分析。

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

## Go GC

### gc 消耗模型

1. GC只涉及两种资源。CPU时间，和物理内存。

2. GC的内存成本由存活堆内存、标记阶段前分配的新堆内存和元数据占用组成。其中元数据即使与前两者消耗成正比，但相比之下也很小。

3. GC的CPU消耗被分为为每个周期的固定消耗，以及与活堆大小成比例的额外消耗。

### GOGC
GOGC参数决定了CPU消耗和内存回收之间的权衡。
该权衡主要通过每次GC结束后的预期的一共扫描的堆大小来决定。
在Go1.8之后，有如下的公式  
Target heap memory = Live heap + (Live heap + GC roots) * GOGC / 100  
一般来说，GOGC翻倍会导致GC扫描的堆大小翻倍，并减少大约一半的CPU消耗
该值可以通过GOGC环境变量/runtime.SetGCPercent设置

### Memory Limit
在Go1.9之后, 该参数被引入表示最大的可使用的内存空间的大小

### write barrier

Go采用经典的三色指针标记法，因为mark的的过程是并发的，因此需要通过一些手段防止并发造成的染色异常。Go通过采用如下的写屏障

对于所有的Move,Store,Zero操作，只有当如下情况发生时，不需要添加writebarrier
- 所写入的对象不包含指针
- 在当前栈上写
- 从只读内存拷贝数据到新的对象
- 当将指针写入某个已经被置0的不需要被堆管理的全局空间时

### GoGC过程 1.19
#### GC初始化
在用户代码开始执行前调度器会启动两个goroutine，`go bgsweep(c); go bgscavenge(c)`。
#### bgsweep
将未被标记的内存释放回堆中
- 在span中查看没有被标记的slot，并将其释放。
- 如果整个span里面都没有存活的object，整个span都将会被释放。
#### bgscavenge
将不使用的内存页返还给操作系统
#### gcController
1. `func (c *gcControllerState) startCycle(markStartTime int64, procs int, trigger gcTrigger)` 开始新一轮GC 需要STW 恢复记录的GC状态 计算出后台并发的mark操作应占全部资源的百分比 对全部的P清空其GCState 根据GC触发条件是时间还是内存占用调整空闲工作线程的最大值
2. `func (c *gcControllerState) revise()` 对state中的heapScan/heapLive/heapGoal进行改变后都需要调用该函数 来对状态进行更新 从而计算出新的预期扫描的堆的大小 以及剩余需要扫描的堆大小和预期的堆增长到的大小，这样在malloc的GoAssist阶段对应的goroutine会协助扫描相应的堆大小。
3. `func (c *gcControllerState) endCycle(now int64, procs int, userForced bool)` 
#### goAssist
并发扫描遵循一种欠债还钱的方式，对于每次malloc，goroutine都会欠垃圾回收器自己需要的空间大小的债务，当添加完这笔债务后该goroutine依然处于欠债状态，该goroutine就会先停止对内存的继续获取，转向帮助GC进行并发的堆的mark。具体需要扫描的工作量由gcController中revise算出的参数决定。
`func gcAssistAlloc(gp *g)`在每个goroutine调用malloc时都会首先检查自己的gcAssistBytes是否小于0

## 常见golang优化手段

### fastslow path
通过将快速常见轻量逻辑的分支与不常见的分支分离在不同的函数，保证快速分支能够被inline，减少函数调用时间。

### false-sharing emit
对于需要被并发访问的结构体数组，对其中每个结构体增加padding确保其对cpu的cacheline的大小对齐，防止出现false-sharing。

## 奇奇怪怪的技巧

### 通过反射获取非导出域

```GO
p := *util.HAHA()
c := reflect.TypeOf(p).Field(0)
pc := reflect.NewAt(c.Type, (unsafe.Pointer)((uintptr)(unsafe.Pointer(&p))+c.Offset)).Interface().(*int)
```
Golang通过reflect.Value中的flag来标识该Value的一些元数据，其中就包含是否为导出域。通过NewAt可以消除flag中的导出标识。

## Golang部分思想

- 分级缓存 优先从P相关的缓存中取，因为P是单线程的(malloc/sync.Pool)
- 工作窃取 在处理完当前任务时，从未完成的任务队列中窃取一般(schedule)
- 信用积分 消耗资源减少积分 释放资源增加积分 批量释放资源可以另积分为正 减少均摊操作时间



[1]: https://go.dev/ref/mem
[2]: https://go.dev/ref/spec
[3]: https://go.dev/doc/asm