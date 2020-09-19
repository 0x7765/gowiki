# gc

## 分类 

>   跟踪式gc
>
>   > 从根对象出发，根据对象间的引用关系，逐步扫描堆并确定需要保留的对象，回收不需要保留的对象。Go、Java 都是使用的这种方式 
>
>   引用计数式gc
>
>   > 每个对象自身包含一个被引用次数的计数器，当计数器归零时自动得到回收。 可能有循环引用问题，Python、OC 使用的是这种方式

### 什么是根对象 

根对象是垃圾回收扫描的起点。主要包含以下对象

1.  全局变量、对象
2.  线程栈  栈上的对象及指向堆上的指针 

### 跟踪式gc的主要实现

1.  标记清理
2.  标记整理
3.  分带式
4.  增量式
5.  增量整理

## go gc

golang 使用的是跟踪式、无分带、无整理、并发的三色标记清除算法。

gc过程的组件一般包含两部分，一部分是collector，负责内存回收。另一部分是mutator，指代用户态代码。对gc而言，用户态代码仅仅是修改对象间的引用关系。

### 写屏障

write barrier 。**这个本身是并发编程中的概念，主要作用是用于多线程下的同步，在gc中也只是出现在并发式gc的垃圾回收算法里。**最直接的理解是 对一个对象引用进行写操作时，在这个写操作之前或者之后，附加其他执行逻辑。

go gc中的写屏障是指 mutator的写屏障，为了保证在并发情况下，用户态代码对对象的引用操作，能够通知到 collector(垃圾回收器)，而不会破坏三色不变性。

### gc的步骤

`go/1.14.4/libexec/src/runtime/mgc.go:24` 代码位置 

### 步骤说明

在源码中，gc是三个阶段

```go
const (
	_GCoff             = iota // GC not running; sweeping in background, write barrier disabled
	_GCmark                   // GC marking roots and workbufs: allocate black, write barrier ENABLED
	_GCmarktermination        // GC mark termination: allocate black, P's help GC, write barrier ENABLED
)
```

| 阶段               | 说明                           |
| ------------------ | ------------------------------ |
| _GCoff             | 后台线程执行内存清理，无写屏障 |
| _GCmark            | 写屏障开启，并发标记阶段       |
| _GCmarktermination | 写屏障开启，标记终止阶段       |

### 详细步骤 

| 步骤 | 阶段和函数                                                  | 说明                                                         |
| ---- | ----------------------------------------------------------- | ------------------------------------------------------------ |
| 1    | SweepTermination                                            | 对未清扫的span进行清扫, 上一轮的GC的清扫工作完成再进行新一轮的GC。**启动 stw** ，标记gc阶段是mark。涉及的函数systemstack、finishsweep_m |
| 2    | gcBgMarkPrepare                                             | 设置后台标记任务计数                                         |
| 3    | gcMarkRootPrepare                                           | 计算扫描根对象的任务数量                                     |
| 4    | gcMarkTinyAllocs                                            | 标记所有tiny alloc等待合并的对象                             |
| 5    | atomic.Store(&gcBlackenEnabled, 1)                          | 启动辅助gc                                                   |
| 6    | systemstack(startTheWorldWithSema)                          | **停止stw**                                                  |
| 7    | GC Mark Termination                                         | 标记终止阶段  gcMarkDone                                     |
| 8    | **systemstack(stopTheWorldWithSema)**                       | **启动stw **                                                 |
| 9    | gcWakeAllAssists                                            | 唤醒所有因辅助gc而休眠的goroutine                            |
| 10   | gcController.endCycle()                                     | 计算下一次触发gc需要的heap大小                               |
| 11   | setGCPhase(_GCmarktermination)                              | 启用写屏障                                                   |
| 12   | systemstack(func() {gcMark(startTime)})                     | 再次开启标记                                                 |
| 13   | systemstack(func(){setGCPhase(_GCoff)；gcSweep(work.mode)}) | 关闭写屏障，唤醒后台清理任务，将在STW结束后开始运行          |
| 14   | gcSetTriggerRatio(nextTriggerRatio)                         | 更新下次触发gc时的heap大小                                   |
| 15   | **systemstack(func() { startTheWorldWithSema(true) })**     | **停止stw**                                                  |
| 16   | gcSweep                                                     | 后台清理任务                                                 |

## gc 的触发逻辑 

有两种方式可以触发gc。

1.  主动触发  runtime.GC 来触发 GC
2.  被动触发 
    1.  使用系统监控，当超过两分钟没有产生任何 GC 时，强制触发 GC
    2.  使用pacing算法，控制内存的增长比例 

```go
// 设置环境变量 
export GOGC=200
// 或者代码中控制 
debug.SetGCPercent(200)
```

## 什么是辅助gc `Mark Assist`

当 GC 触发后，会首先进入并发标记的阶段。并发标记会设置一个标志，并在 mallocgc 调用时进行检查。当存在新的内存分配时，会暂停分配内存过快的那些 goroutine，并将其转去执行一些辅助标记（Mark Assist）的工作，从而达到放缓继续分配、辅助 GC 的标记工作的目的. 

