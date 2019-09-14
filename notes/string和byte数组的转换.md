# 底层结构

- [x] 任何类型的指针和 `unsafe.Pointer` 可以相互转换
- [x] `uintptr` 类型和 `unsafe.Pointer` 可以相互转换

转换的基本语法是 `使用unsafe.Pointer来将T1转化为T2，一个大致的语法为*(*T2)(unsafe.Pointer(&t1))`

## string

```go
// reflect/value.go:1947
type StringHeader struct {
	Data uintptr
	Len  int
}
```

## slice
```go
// reflect/value.go:1964
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
```

## string和byte的互转
```go
import (
	"reflect"
	"unsafe"
)

func BytesToString(b []byte) string {
	bh := (*reflect.SliceHeader)(unsafe.Pointer(&b))
	sh := reflect.StringHeader{bh.Data, bh.Len}
	return *(*string)(unsafe.Pointer(&sh))
}

func BytesToStringOrigin(b []byte) string {
	return string(b)
}

func StringToBytes(s string) []byte {
	sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
	bh := reflect.SliceHeader{sh.Data, sh.Len, 0}
	return *(*[]byte)(unsafe.Pointer(&bh))
}

func StringToBytesOrigin(s string) []byte {
	return []byte(s)
}
```

# 性能测试

## 测试代码

```go
import (
	"reflect"
	"testing"
)

var (
	str = "hello world hello world hello world hello world hello world hello world hello world hello world hello world hello world hello world hello world"
	bs  = []byte{104, 101, 108, 108, 111, 32, 119, 111, 114, 108, 100, 104, 101, 108, 108,
		111, 32, 119, 111, 114, 108, 100, 104, 101, 108, 108, 111, 32, 119, 111, 114, 108, 100,
		104, 101, 108, 108, 111, 32, 119, 111, 114, 108, 100, 104, 101, 108, 108, 111, 32, 119,
		111, 114, 108, 100, 104, 101, 108, 108, 111, 32, 119, 111, 114, 108, 100}
)

func TestBytesToString(t *testing.T) {
	type args struct {
		b []byte
	}
	tests := []struct {
		name string
		args args
		want string
	}{
		{"1", args{b: []byte{104, 101, 108, 108, 111, 32, 119, 111, 114, 108, 100}}, "hello world"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := BytesToString(tt.args.b); got != tt.want {
				t.Errorf("BytesToString() = %v, want %v", got, tt.want)
			}
		})
	}
}

func TestStringToBytes(t *testing.T) {
	type args struct {
		s string
	}
	tests := []struct {
		name string
		args args
		want []byte
	}{
		{"1", args{s: "hello world"}, []byte{104, 101, 108, 108, 111, 32, 119, 111, 114, 108, 100}},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := StringToBytes(tt.args.s); !reflect.DeepEqual(got, tt.want) {
				t.Errorf("StringToBytes() = %v, want %v", got, tt.want)
			}
		})
	}
}

func BenchmarkBytesToString(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		BytesToString(bs)
	}
}

func BenchmarkBytesToStringOrigin(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		BytesToStringOrigin(bs)
	}
}

func BenchmarkStringToBytes(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		StringToBytes(str)
	}
}

func BenchmarkStringToBytesOrigin(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		StringToBytesOrigin(str)
	}
}
```

## 结果

```text
# 简单字符串时
$ go test -v -bench=. -benchtime=10s -benchmem  -run=none
goos: windows
goarch: amd64
pkg: github.com/0x7765/learngo/bench-demo
BenchmarkBytesToString-4                1000000000               0.338 ns/op           0 B/op          0 allocs/op
BenchmarkBytesToStringOrigin-4          1000000000               6.05 ns/op            0 B/op          0 allocs/op
BenchmarkStringToBytes-4                1000000000               0.340 ns/op           0 B/op          0 allocs/op
BenchmarkStringToBytesOrigin-4          1000000000               7.04 ns/op            0 B/op          0 allocs/op
PASS

# 较长的字符串时
$ go test -v -bench=. -benchtime=10s -benchmem  -run=none
goos: windows
goarch: amd64
pkg: github.com/0x7765/learngo/bench-demo
BenchmarkBytesToString-4                1000000000               0.341 ns/op           0 B/op          0 allocs/op
BenchmarkBytesToStringOrigin-4          297084030               40.1 ns/op            80 B/op          1 allocs/op
BenchmarkStringToBytes-4                1000000000               0.343 ns/op           0 B/op          0 allocs/op
BenchmarkStringToBytesOrigin-4          195566586               61.0 ns/op           144 B/op          1 allocs/op

```