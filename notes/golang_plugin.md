#  plugin

## 基本用法

### 定义

```go
package main

import (
	"errors"
)

func Add(a, b int) int {
	return a + b
}

func Subtract(a, b int) int {
	return a - b
}

func Multiply(a, b int) int {
	return a * b
}

func Divide(a, b int) (float64, error) {
	if b == 0 {
		return 0, errors.New("can not divide zero")
	}
	return float64(a) / float64(b), nil
}
```

### 编译

**不支持windows**，必须包含`main`包

```GO
go build --buildmode=plugin tools/calc.go
```

会在当前目录下生成 `calc.so`

### 使用

```GO
package main

import (
	"log"
	"plugin"
)

// 使用的api很简答 只有两个
// 1. open
// 2. lookup
func main() {
	// open
	p, err := plugin.Open("calc.so")

	if err != nil {
		panic(err)
	}

	// 寻找函数
	addf, err := p.Lookup("Add")
	if err != nil {
		panic(err)
	}
	// 转换
	add := addf.(func(int, int) int)

	log.Printf("add\t%d\n", add(100, 200))

	devidef, err := p.Lookup("Divide")
	if err != nil {
		panic(err)
	}

	devide := devidef.(func(int, int) (float64, error))

	f1, err := devide(200, 100)
	log.Printf("devide\t%f\terr:%v\n", f1, err)

	f2, err := devide(200, 0)
	log.Printf("devide\t%f\terr:%v\n", f2, err)
}
```



