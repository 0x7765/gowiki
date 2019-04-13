# slice 

参考

[https://gocn.vip/article/1695](https://gocn.vip/article/1695)

[https://www.cnblogs.com/yjf512/p/5147365.html](<https://www.cnblogs.com/yjf512/p/5147365.html>)

## 定义

动态数组，类似java中的list

## 结构

`runtime\slice.go`

```go
type slice struct {
	array unsafe.Pointer // 指向数组的指针
	len   int			// 长度
	cap   int			// 容量
}
```

## 创建方式

| 创建方式   | 示例                 | 说明                                           |
| ---------- | -------------------- | ---------------------------------------------- |
| var 声明   | var arr []string     | 创建一个nil slice。len==cap= 0,arr == nil      |
| 初始化声明 | s1 := []int{1, 2, 3} |                                                |
| make创建   | s1 := make([]int, 0) | 创建一个empty slice。等同于 `var s1 = []int{}` |
| 截取       | s2 := s1[1:3]        |                                                |
| new        | s3 := *new([]int)    | 创建了一个 nil slice                           |

## nil vs empty slice

1. nil slice 的 len 和 cap 都是0 ，和nil 比较是true
2. empty slice的len 和cap也是0，但是底层数组指向的地址不为0
3. **官方推荐使用nil slice。对一个nil的slice append 并不会空指针**

> 示例
>
> ```go
> func testAppend() (res []int) {
> 	fmt.Println(res == nil)
> 	res = append(res, 1)
> 	return
> }
> ```

## make 创建slice的过程

```go
package main

import "fmt"

func main() {
	arr := make([]int, 0)
	fmt.Println(arr)
}

// go tool compile -S main.go
```

大致流程是 

1.  CALL    runtime.makeslice(SB)
2.  CALL    runtime.convTslice(SB) 

### makeslice

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {

    // ... 核心是mallocgc
	return mallocgc(mem, et, true)
}
```

### mallocgc

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {

	// Set mp.mallocing to keep from being preempted by GC.
    // 获取当前goroutine对应的M
	mp := acquirem()
    // mallocing标志位 非0表示正在进行内存分配
	if mp.mallocing != 0 {
		throw("malloc deadlock")
	}
	if mp.gsignal == getg() {
		throw("malloc during signal")
	}
    // 锁住当前的M 进行内存分配
	mp.mallocing = 1

	shouldhelpgc := false
	dataSize := size
    // 获取当前goroutine的m的mcache
	c := gomcache()
	var x unsafe.Pointer
	noscan := typ == nil || typ.kind&kindNoPointers != 0
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			// Tiny allocator.
			// reduces heap size by ~20%.
			off := c.tinyoffset
			// Align tiny pointer for required (conservative) alignment.
			if size&7 == 0 {
				off = round(off, 8)
			} else if size&3 == 0 {
				off = round(off, 4)
			} else if size&1 == 0 {
				off = round(off, 2)
			}
			if off+size <= maxTinySize && c.tiny != 0 {
				// The object fits into existing tiny block.
				x = unsafe.Pointer(c.tiny + off)
				c.tinyoffset = off + size
				c.local_tinyallocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}
			// Allocate a new maxTinySize block.
			span := c.alloc[tinySpanClass]
			v := nextFreeFast(span)
			if v == 0 {
				v, _, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
			if size < c.tinyoffset || c.tiny == 0 {
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
			size = maxTinySize
		} else {
			var sizeclass uint8
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
			} else {
				sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
			}
			size = uintptr(class_to_size[sizeclass])
			spc := makeSpanClass(sizeclass, noscan)
			span := c.alloc[spc]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc)
			}
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(unsafe.Pointer(v), size)
			}
		}
	} else {
		var s *mspan
		shouldhelpgc = true
        // 大对象
		systemstack(func() {
			s = largeAlloc(size, needzero, noscan)
		})
		s.freeindex = 1
		s.allocCount = 1
		x = unsafe.Pointer(s.base())
		size = s.elemsize
	}

	if msanenabled {
		msanmalloc(x, size)
	}
	// 释放标志位 
	mp.mallocing = 0
	releasem(mp)
	return x
}
```

1. 如果要申请的对象是tiny大小，看mcache中的tiny block是否足够，如果足够，直接分配。如果不足够，使用mcache中的tiny class对应的span分配
2. 如果要申请的对象是小对象大小，则使用mcache中的对应span链表分配
3. 如果对应span链表已经没有空span了，先补充上mcache的对应链表，再分配（mCache_Refill）
4. 如果要申请的对象是大对象，直接去heap中获取（largeAlloc）

## 扩容

growslice

1. 小于1024时，newcap = oldcap * 2
2. 大于1024，newcap > oldcap * 2 or old * 1.25 需要内存对齐