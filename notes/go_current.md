# 并发控制

>   多线程、多任务之间协作和数据共享的一种控制机制

## go中并发控制的方式

-   channel
-   context
-   waitgroup
-   全局变量(atomic)
-   sync下的mutex、condition、

### channel

channel是最常见的并发控制方式，主要用于多线程之间的信号传递、消息同步、超时控制。

### 超时控制

```go
func GetHttpResp(url string, duration time.Duration) (resp string, err error) {
	done := make(chan struct{})

	go func() {
		defer close(done)
		resp, err = Get(url)
	}()

	select {
	case <-done:
		log.Printf("job success ...")
	case <-time.After(duration):
		log.Printf("job timeout ...")
	}
	return
}
```

### 多任务并发执行

```go
type Resp struct {
	resp string
	err  error
}

func GetRemote(url string, res chan string) {
	finalResp := ""
	defer func() {
		res <- finalResp
	}()

	if resp, err := Get(url); err == nil {
		finalResp = resp
	} else {
		log.Printf("response error %v\\n", err)
	}
}

func GetRemoteV2(url string, res chan *Resp) {
	resp := &Resp{}
	defer func() {
		res <- resp
	}()

	get, err := Get(url)
	resp.resp = get
	resp.err = err
}

func main() {
	chan1 := make(chan string)
	chan2 := make(chan *Resp)

	go GetRemote("<https://www.baidu.com>", chan1)
	go GetRemoteV2("<https://daily-note.github.io>", chan2)

	for i := 0; i < 2; i++ {
		select {
		case str := <-chan1:
			log.Printf("[get_remote1] resp %s\\n", str)
		case resp := <-chan2:
			log.Printf("[get_remote2] resp %+v\\n", resp)
		}
	}
}
```

### context

一般是`context`配合`channel`或者`timer`

```go
func Get(url string) (string, error) {
	if len(url) == 0 {
		return "", errors.New("url is empty")
	}
	log.Printf("request to %s \\n", url)
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
			log.Printf("request to %s error %+v\\n", url, err)
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
		log.Printf("request to %s timeout\\n", url)
		return "", errors.New("timeout")
	case res := <-resp:
		return res, nil
	}
}

func main() {

	ctx, cancelFunc := context.WithTimeout(context.TODO(), 1000*time.Millisecond)
	defer cancelFunc()
	now := time.Now()
	timeout, err := GetWithTimeout(ctx, "<https://www.google.com>")
	log.Printf("resp %+v %+v %v\\n", timeout, err, time.Since(now))
}
```

### waitgroup

```go
func main(){
			wg := sync.WaitGroup{}
			wg.Add(2)
			go func() {
				defer wg.Done()
				Get("<https://www.baidu.com>")
			}()
		
			go func() {
				defer wg.Done()
				Get("<https://daily-note.github.io>")
			}()
		
			wg.Wait()
}
```

### 全局变量
