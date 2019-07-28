# GMP 模型

## 基本概念

golang的核心线程模型有三部分，分别是GMP。

- [ ] M: machine的缩写。一个M代表一个内核线程，或称为`工作线程`。M和KSE之间是一一对应的关系
- [ ] P: processor的缩写。一个P代表执行一个GO代码片段所需要的必需资源，或称上下文环境
- [ ] G: goroutine 的缩写。一个G代表一个代码片段

## M

代表内核线程。一般情况下，创建一个M都是因为没有足够的M来关联P并运行其中的可运行的G队列。另外，在运行时，执行系统监控或者垃圾回收时，也会创建M。

### 结构

主要字段如下 `src/runtime/runtime2.go`

```go
type m struct {
	g0      *g     // goroutine with scheduling stack g0是系统运行之初创建的，用于执行运行时任务
    mstartfn      func() // 代表用于在新的M上启动某个特殊任务的函数 (GC、自旋、系统监控)
	curg          *g       // current running goroutine 当前M正在运行的G的指针
	p             puintptr // attached p for executing go code (nil if not executing go code) 指向当前与M关联的P
    nextp         puintptr // 用于暂存与当前M有潜在关联关系的P(预关联)
	spinning      bool // m is out of work and is actively looking for work 是否在自旋
	blocked       bool // m is blocked on a note 是否被阻塞
	lockedg       guintptr // 运行时可以将一个M和一个G锁定在一起
    alllink       *m // on allm 创建后被放在全局的allm上,应该是个链表?
	vdsoSP uintptr // SP for traceback while in VDSO call (0 if not in call) SP寄存器用于现场恢复和现场保护
	vdsoPC uintptr // PC for traceback while in VDSO call
}
```

- 当前版本的最大值是10000
- 系统维护了一个全局的m列表:allm

## P

p是G能够在M中运行的关键。go运行时系统会适时的让P与不同的M建立或断开联系，以保证P中的那些G能够及时的获得运行时机。

### 结构

- 改变P的数量 比如 `runtime.GOMAXPROCS(runtime.NumCPU() * 2)`
- 运行时系统也存在一个全局的空闲P列表`runtime.sched.allp`，当一个p不再与任何M关联且它的可运行G列表是空，才会被放入全局的空闲列表

### 状态

| 状态     | 说明                                                    |
| -------- | ------------------------------------------------------- |
| pidle    | p未与任何M关联                                          |
| prunning | p正在和某个M关联                                        |
| psyscall | 当前p中的运行的那个G正在进行系统调用                    |
| pgcstop  | 运行时系统需要停止调度                                  |
| pdead    | 当前p已经不会再被使用(通过设置全局p的数量降低了p的个数) |



