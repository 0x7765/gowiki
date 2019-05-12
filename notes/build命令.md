# go的工具链

# go build

格式`go build args...`

| 参数    | 说明                            | 示例                                          |
| ------- | ------------------------------- | --------------------------------------------- |
| -o      | 指定可执行文件名 默认和目录同名 | go build -o hello / go build -o hello main.go |
| -a      | 强制重新编译所有的包            | go build -a                                   |
| -v      | 打印待编译的包的名字            | go build -v                                   |
| -x      | 显示正在执行的编译命令          | go build -x                                   |
| -work   | 显示临时工作目录，完成后不删除  | go build -work                                |
| -race   | 启动数据竞争检查                |                                               |
| -gcflag | 指定编译器参数                  | go build -gcflag="-m -l"                      |
| -ldflag | 指定链接器参数                  | go build -ldflag="-s -w"                      |

## 编译器参数说明`gcflag`

| 编译器参数 | 说明         |
| ---------- | ------------ |
| -B         | 禁用越界检查 |
| -N         | 禁用优化     |
| -l         | 禁用内联     |
| -u         | 禁用unsafe   |
| -S         | 输出汇编     |
| -m         | 输出优化信息 |

## 链接器参数说明`ldflag`

| 参数 | 说明                                    |
| ---- | --------------------------------------- |
| -s   | 禁用符号表                              |
| -w   | 禁用DRAWF 调试信息                      |
| -X   | 设置字符串全局变量值 `-X hello="world"` |
| -H   | 设置可执行文件格式                      |

# 条件编译

## 将平台和架构信息添加到文件尾部

- [ ]  hello_linux.go
- [ ]  hello_darwin.go

## 使用build 指令

hello_linux.go

```go
// +build linux

package main

import "time"

func Now() string {
	return "linux:" + time.Now().String()
}
```

build 指令和package之间必须有空行,可以指定多个

```go
// +build linux drawin
```

多个指令的关系

1. 空格 表示or 即上面的表示 linux或者mac环境可以编译
2. `,` 表示and `// +build amd64,!cgo` 表示是64位环境并且非cgo
3. `！`表示not

**build 也可指定版本 编译器等**

```go
// +build ignore
// +build gccgo
// +build go1.12 
```

## 使用tags指令

debug.go

```go
// +build !release

package gindemo

func Demo() {
	print("debug...")
}
```

release.go

```go
// +build release

package gindemo

func Demo() {
	print("release...")
}
```

main.go

```go
func main() {
	Demo()
}
```

编译 tags 可以指定多个 空格分开

`go build -tags "release"`

