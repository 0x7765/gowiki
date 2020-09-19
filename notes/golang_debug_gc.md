# golang debug gc的方式 

> 有四种方式 可以跟踪 golang的gc 
>
> - [ ] 注入`GODEBUG=gctrace=1` 环境变量到进程中
> - [ ] `trace ` 可视化的跟踪线程的gc行为
> - [ ] `debug`的api
> - [ ] `runtime`的api



## 方式一 

`GODEBUG=gctrace=1 `环境变量 , 还可以加上`gcpercent=1` 查看比例 

```go
import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	engine := gin.Default()

	engine.GET("/ping", func(context *gin.Context) {
		context.String(http.StatusOK, "pong")
	})

	engine.Run(":8080")
}

// 启动 
// GODEBUG=gctrace=1 go run main.go
// 或者GODEBUG=gctrace=1,gcpercent=1 go run main.go
//gc 1 @0.008s 2%: 0.023+0.67+0.12 ms clock, 0.37+0.56/0.51/0.001+2.0 ms cpu, 4->4->0 MB, 5 MB goal, 16 P
//gc 2 @0.010s 2%: 0.036+0.34+0.016 ms clock, 0.57+0.14/0.48/0.18+0.26 ms cpu, 4->4->0 MB, 5 MB goal, 16 P
//gc 3 @0.015s 2%: 0.040+0.47+0.024 ms clock, 0.65+0.15/0.76/0+0.38 ms cpu, 4->4->0 MB, 5 MB goal, 16 P
//gc 4 @0.018s 3%: 0.051+0.38+0.031 ms clock, 0.83+0.53/0.82/0+0.49 ms cpu, 4->4->0 MB, 5 MB goal, 16 P
//gc 5 @0.022s 3%: 0.053+0.52+0.026 ms clock, 0.85+0.46/0.78/0.022+0.42 ms cpu, 4->4->0 MB, 5 MB goal, 16 P
//gc 6 @0.025s 3%: 0.054+0.51+0.059 ms clock, 0.87+0.63/0.90/0.044+0.95 ms cpu, 4->4->0 MB, 5 MB goal, 16 P
//gc 7 @0.029s 3%: 0.047+0.38+0.042 ms clock, 0.75+0.064/0.73/0+0.67 ms cpu, 4->4->0 MB, 5 MB goal, 16 P
//gc 8 @0.031s 3%: 0.040+0.51+0.032 ms clock, 0.64+0.21/0.82/0.001+0.52 ms cpu, 4->4->0 MB, 5 MB goal, 16 P
```

##  方式二

`trace`的方式  http服务直接开启`pprof `

```go
import (
	"net/http"

	"github.com/DeanThompson/ginpprof"
	"github.com/gin-gonic/gin"
)

func main() {
	engine := gin.Default()

	// 集成pprof 也可以自己手动支持 
	ginpprof.Wrap(engine)

	engine.GET("/ping", func(context *gin.Context) {
		context.String(http.StatusOK, "pong")
	})

	engine.Run(":8080")
}
// 打开 http://localhost:8080/debug/pprof/ 可以看到trace、profile、heap相关的pprof已经打开。
```

###### ![image-20200620235838508](https://daily-note.github.io/images/image-20200620235838508.png)

1. wrk 模拟请求测试

```SHELL
wrk -t12 -c100 -d40s http://localhost:8080/ping
```

2. dump trace信息 

```SHELL
curl http://localhost:8080/debug/pprof/trace?seconds=30 > trace.out
```

3. 然后分析trace

```shell
go tool trace trace.out
```

点击进入goroutine 或者view trace 就可以跟踪当前线程的gc信息  后续单独分析trace相关

![image-20200621000510904](https://daily-note.github.io/images/image-20200621000510904.png)

**后序单独分析trace部分**

## 方法三 

`debug.ReadGCStats`

```GO
func gcInfo() {
	stat := debug.GCStats{}
	for range time.Tick(time.Second) {
		debug.ReadGCStats(&stat)
		log.Printf("gc %d last@%v, PauseTotal %v\n", stat.NumGC, stat.LastGC, stat.PauseTotal)
	}
}

// 在main 函数中新起一个线程一步执行 
```

## 方法四 

`runtime.ReadMemStats` 和方法三类似 

```GO
func gcInfo() {
	stat := runtime.MemStats{}
	for range time.Tick(time.Second) {
		runtime.ReadMemStats(&stat)
		log.Printf("gc %d last@%v, PauseTotal %v\n", stat.NumGC, stat.LastGC, stat.HeapInuse)
	}
}
// 这里可以统计的信息有很多  具体可以参看 src/runtime/mstats.go:150 这个struct的定义
```



