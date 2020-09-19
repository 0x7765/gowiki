# Context的原理

## context是什么

> `context`是golang 1.7 标准库加入的api。一般翻译为上下文，准确的说是线程的上下文。

常见的用法主要有三种

1. goroutine之间传递上下文及参数
2. 并发超时控制  
3. goroutine之间信号传递(cancel、超时)

## 组成结构

### 顶层接口

```go
type Context interface {
  // 返回 context 是否会被取消以及自动取消时间（即 deadline）
	Deadline() (deadline time.Time, ok bool)
  // 当 context 被取消或者到了 deadline，返回一个被关闭的 channel
	Done() <-chan struct{}
	// 在 channel Done 关闭后，返回 context 取消原因
	Err() error
	// 获取指定key 对应的 value
	Value(key interface{}) interface{}
}

// cancel的接口 
// 实现这个接口的context 都是可以取消的 
type canceler interface {
	// 
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}
```

### 对应的实现

### emptyCtx

这是一个空的实现 所有的都是默认值 不会取消 也没有deadline。 对外提供的Background 和 TODO 两个context 实际是emptyCtx。一般用于占位

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}

var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)
```

### cancelCtx

```go
type cancelCtx struct {
	Context

	// 保护下面的三个字段 相关的操作需要加锁
	mu       sync.Mutex            // protects following fields
	// 懒汉式创建 
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}

// 操作加锁
// 懒汉式创建 
func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()
	if c.done == nil {
		c.done = make(chan struct{})
	}
	d := c.done
	c.mu.Unlock()
	return d
}

// 核心方法 
// 这个方法会 close掉done的channel
// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	// 赋值err 
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		// 关闭channel 通知其它goroutine 
		close(c.done)
	}
	// 递归取消所有的子context
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	// 置空 子节点 
	c.children = nil
	c.mu.Unlock()

	// 从父节点中移除本身 
	if removeFromParent {
		removeChild(c.Context, c)
	}
}

// 创建一个可取消的context 
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	// 创建一个上面的cancelCtx
	c := newCancelCtx(parent)
	// 向上寻找一个可挂载&&可取消的父context 
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}

func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	// 父节点是一个空节点 或者非cancel的节点
	if done == nil {
		return // parent is never canceled
	}

	select {
	case <-done:
		// parent is already canceled
		// 父节点已经被取消了 
		// 子节点也要被取消 
		child.cancel(false, parent.Err())
		return
	default:
	}

	// 找到一个可取消的context 
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			// 将当前context放在map中 
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		// 如果没有找到可取消的父 context。新启动一个协程监控父节点或子节点取消信号
		atomic.AddInt32(&goroutines, +1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

> 总结下

> 1. done() 方法是创建(首次)并返回一个空的只读的channel

> 2. cancel() 方法主要是 关闭channel，将取消信号发送给所有的子节点和外层的其它监听的goroutine，`(select case ←done)`

> 3. 创建一个可取消的context时，优先从父节点中寻找可取消的context挂载（这样父节点取消时，会自动取消父节点下的所有节点），如果没有找到可取消的父节点，则创建一个新的协程，监控父节点或者子节点的取消。

### timerCtx

> `timerCtx` 基于 `cancelCtx`，只是多了一个 `time.Timer` 和一个 `deadline`。Timer 会在 deadline 到来时，自动取消 context

```go
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}

// 重点看 如何创建一个timerCtx 
// context.WithDeadline()
// context.WithTimeout()

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	// 如果父节点的deadline在子节点的deadline之前，直接创建一个cancel的context
	// 以为父节点先取消，子节点也会跟着取消 会自动调用子节点的cancel方法 
  // 所以不用单独处理 子节点的timer问题  
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	// 挂载在父节点上 
	propagateCancel(parent, c)

	// 计算距离deadline的时间 
	dur := time.Until(d)
	// 小于0 直接取消 
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		// d 时间后，timer 会自动调用 cancel 函数。自动取消
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

### valueCtx

这个比较简单，需要注意的点事k的查找事递归查询。先从自身寻找，找不到的时候从父节点查询。

## 基本用法

### 数据传递

```go
func step1(ctx context.Context) context.Context {
	logId, ok := ctx.Value("log_id").(string)
	if !ok {
		logId = uuid.NewV4().String()
		ctx = context.WithValue(ctx, "log_id", logId)
	}
	ctx = context.WithValue(ctx, "host", "127.0.0.1")
	log.Printf("%s step1\n", logId)
	return ctx
}

func step2(ctx context.Context) context.Context {
	logId, ok := ctx.Value("log_id").(string)
	if !ok {
		logId = uuid.NewV4().String()
		ctx = context.WithValue(ctx, "log_id", logId)
	}
	log.Printf("%s step2\n", logId)
	return ctx
}

func main() {

	ctx := context.TODO()

	ctx = step1(ctx)

	step2(ctx)
}

// 2020/06/30 00:33:05 09e67d39-1c1e-416a-8085-f7586c0273d5 step1
// 2020/06/30 00:33:05 09e67d39-1c1e-416a-8085-f7586c0273d5 step2
```

### 超时控制

> 这个demo只是为了举例  http自带有支持超时控制的api

```go
func Get(url string) (string, error) {
	if len(url) == 0 {
		return "", errors.New("url is empty")
	}
	log.Printf("request to %s \n", url)
	rsp, err := http.Get(url)
	defer rsp.Body.Close()
	if err != nil {
		return "", err
	}

	if bytes, err := ioutil.ReadAll(rsp.Body); err != nil {
		return "", err
	} else {
		return string(bytes), nil
	}
}

func GetWithTimeout(ctx context.Context, url string) (string, error) {
	resp := make(chan string)
	go func() {
		if rsp, err := Get(url); err != nil {
			log.Printf("request to %s error %+v\n", url, err)
			resp <- ""
		} else {
			resp <- rsp
		}
	}()

	// 给定的超时时间后 ctx 会自动调用 cancel方法
	// http get 先返回 就直接返回  
	// http get 给定时间未返回  cancel方法会执行 ctx.Done() 执行 
	select {
	case <-ctx.Done():
		log.Printf("request to %s timeout\n", url)
		return "", errors.New("timeout")
	case res := <-resp:
		return res, nil
	}
}

func main() {

	ctx, cancelFunc := context.WithTimeout(context.TODO(), 1000*time.Millisecond)
	defer cancelFunc()
	now := time.Now()
	timeout, err := GetWithTimeout(ctx, "https://www.google.com")
	log.Printf("resp %+v %+v %v\n", timeout, err, time.Since(now))
}
```

### 取消子任务

> 和上一个类似 区别是不关心返回值 一般用于终止循环任务

```go
// 默认每2秒执行一次
// 当外部主动执行 cancel或者 超时时间到了之后 会主动退出
func Clock(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			log.Printf("job cancel ...")
			return
		case <-time.Tick(time.Second * 2):
			log.Printf("job run , now is %s\n", time.Now())
		}
	}
}

func main() {

	ctx, cancelFunc := context.WithTimeout(context.TODO(), 10*time.Second)
	defer cancelFunc()
	go Clock(ctx)
	time.Sleep(time.Second * 5)
	// 主动执行 
	//cancelFunc()
	time.Sleep(time.Minute)
}

// 2020/06/30 01:28:00 job run , now is 2020-06-30 01:28:00.773146 +0800 CST m=+2.001169523
// 2020/06/30 01:28:02 job run , now is 2020-06-30 01:28:02.777742 +0800 CST m=+4.005716953
// 2020/06/30 01:28:04 job run , now is 2020-06-30 01:28:04.777925 +0800 CST m=+6.005851741
// 2020/06/30 01:28:06 job run , now is 2020-06-30 01:28:06.778217 +0800 CST m=+8.006094715
// 2020/06/30 01:28:08 job cancel ...
```

