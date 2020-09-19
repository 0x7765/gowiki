# go test 

## 常用工具 

```go
// https://github.com/vektra/mockery/
// https://github.com/stretchr/testify

go get -u -v github.com/stretchr/testify/...
brew install vektra/tap/mockery
```

说明

1.  mockery 用于mock无法直接进行单元测试的外部依赖，比如rpc请求、db查询等。
2.  Table-driven: 编写多条case 测试代码的覆盖率

## 基本测试 

str_convert.go

```go
import (
	"reflect"
	"unsafe"
)

func Str2Bytes(str string) []byte {
	x := (*reflect.StringHeader)(unsafe.Pointer(&str))
	h := reflect.SliceHeader{Data: x.Data, Len: x.Len, Cap: x.Len}
	return *(*[]byte)(unsafe.Pointer(&h))
}

func Bytes2Str(bs []byte) string {
	x := (*reflect.SliceHeader)(unsafe.Pointer(&bs))
	h := reflect.StringHeader{Data: x.Data, Len: x.Len}
	return *(*string)(unsafe.Pointer(&h))
}
```

str_convert_test.go

```go
func TestStr2Bytes(t *testing.T) {
	type args struct {
		str string
	}
	tests := []struct {
		name string
		args args
		want []byte
	}{
		{name: "1", args: args{str: "hello"}, want: []byte("hello")},
		{name: "2", args: args{str: ""}, want: []byte("")},
		{name: "3", args: args{str: "hello world"}, want: []byte("hello world")},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := Str2Bytes(tt.args.str)
			if len(got) == 0 && len(got) == len(tt.want) {
				return
			}
			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("Str2Bytes() = %v, want %v", got, tt.want)
			}
		})
	}
}
```

运行

```shell
 go test -v .
```

##  mock测试 

一般场景是 要做单元测试的函数中 依赖了外部rpc等无法直接测试的逻辑。mockery 测试时基于接口的。

假设这里是 rpc的服务

```go
//go:generate mockery --name IRpcService

type IRpcService interface {
	GetHost() string
	GetPort() uint
	GetResp() string
}

type RpcService struct {

}

func (r *RpcService) GetHost() string {
	panic("implement me")
}

func (r *RpcService) GetPort() uint {
	panic("implement me")
}

func (r *RpcService) GetResp() string {
	panic("implement me")
}
```

2.  要做单元测试的函数

```GO
func Exec(service IRpcService) (string, error) {

	if len(service.GetHost()) == 0 {
		return "", errors.New("host is empty")
	}
	if service.GetPort() == 0 {
		return "", errors.New("port is zero")
	}
	return service.GetResp(), nil
}
```

现在对 Exec 函数做单元测试前，需要先生成mock的逻辑

```go
// interface定义的位置
go generate
// 输出一下内容 会在当前目录下生成mocks 目录 
// 14 Jul 20 00:29 CST INF Starting mockery dry-run=false version=2.0.4
// 14 Jul 20 00:29 CST INF Walking dry-run=false version=2.0.4
// 14 Jul 20 00:29 CST INF Generating mock dry-run=false interface=IRpcService qualified-name=gotest/service version=2.0.4
```

4.  编写单元测试

```go
func TestExec(t *testing.T) {
	type args struct {
		service IRpcService
	}

	rs1 := &mocks.IRpcService{}
  // on 指定对应的 函数  return mock 当前函数的返回值
	rs1.On("GetHost").Return("127.0.0.1")
	rs1.On("GetPort").Return(uint(10086))
	rs1.On("GetResp").Return("Hello World")

	rs2 := &mocks.IRpcService{}
	rs2.On("GetHost").Return("")
	rs2.On("GetPort").Return(uint(10086))
	rs2.On("GetResp").Return("Hello World")

	rs3 := &mocks.IRpcService{}
	rs3.On("GetHost").Return("127.0.0.1")
	rs3.On("GetPort").Return(uint(0))
	rs3.On("GetResp").Return("Hello World")

	tests := []struct {
		name    string
		args    args
		want    string
		wantErr bool
	}{
		{name: "1", args: args{rs1}, want: "Hello World", wantErr: false},
		{name: "2", args: args{rs2}, want: "", wantErr: true},
		{name: "3", args: args{rs3}, want: "", wantErr: true},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := Exec(tt.args.service)
			require.Equal(t, tt.wantErr, err != nil)
			assert.Equal(t, tt.want, got)
		})
	}
}
```

5.  测试 

```go
// 覆盖率测试
go test -v -coverprofile=cover.out
// 生成html的报告
go tool cover -html=cover.out -o coverage.html
// 查看html的内容 可以看见具体覆盖了哪些语句
```

## 基准测试 

```go
func Str2Bytes(str string) []byte {
	x := (*reflect.StringHeader)(unsafe.Pointer(&str))
	h := reflect.SliceHeader{Data: x.Data, Len: x.Len, Cap: x.Len}
	return *(*[]byte)(unsafe.Pointer(&h))
}

func Str2BytesOld(str string) []byte {
	return []byte(str)
}
```

测试文件

```go
var (
	strs = []string{
		"1mYzu33hfMsdfbakzanb",
		"HQTmmIR1dnd1xqoKLqRy",
		"PzVKw9pGb7NWIRJkJYW7",
		"PtnF9sa0pSYPVAvn8Er7",
		"b3qIDV62QOPvIsB2qyGh",
		"moeZwetNOb6iOjjNvmYNVGXmjiuQRNwfdx1XasLx",
		"yISsdhBgmCtjhI7bQ4YuVcfrzBYZTT7RfRs92QIe",
		"wKQNrdN435xETda8IB6JqTXI8ZkDAJW4JTyH6wb3g7HXRk6vGBAlCXrOTfj8",
		"J06sViW3kVTVmpS1U7NYR6DeY9d8R9a0TCPxJwPvkdXY4VNxt4sCa5KGU9dt",
		"6j0iXRVjusGyuyMquCLhKShRuGSsl39Q4THhhLfLG3AfcvkGdVNafMgiYaNunEBeKjvOAjnaXH8wthalXd7ugXvpbdYeZQ91urzn",
	}
)

func BenchmarkStr2BytesOld(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		for _, s := range strs {
			Str2BytesOld(s)
		}

	}
}

func BenchmarkStr2Bytes(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		for _, s := range strs {
			Str2Bytes(s)
		}

	}
}
```

执行测试

```go
go test -bench=. -run=none -benchmem -benchtime=15s
// goos: darwin
// goarch: amd64
// pkg: gotest/basic
// BenchmarkStr2BytesOld-16    	87806494	       197 ns/op	     336 B/op	       5 allocs/op
// BenchmarkStr2Bytes-16       	1000000000	         3.09 ns/op	       0 B/op	       0 allocs/op
// PASS
// ok  	gotest/basic	21.432s
```

同时可以指定生成profile文件

```go
go test -bench=. -run=none -benchmem -benchtime=15s -cpuprofile=cpu.out -memprofile=mem.out -trace=str.out
// 分别是生成cpu、memory profile文件  trace 用于生成更细的trace 分析
```

分析

```shell
go tool pprof -http=:8080 cpu.out
go tool pprof -http=:8080 mem.out 
# trace
go tool trace str.out
```

内存这里只能看到 old的版本有内存分配


