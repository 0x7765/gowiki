# struct compare

## 哪些类型不能比较

1.  slice
2.  map
3.  function
4.  以及包含了上述三种类型之一的struct

## 为什么这些类型不能比较

尝试比较

```bash
invalid operation: a1 == a2 (slice can only be compared to nil)
invalid operation: a1 == a2 (map can only be compared to nil)
invalid operation: a1 == a2 (func can only be compared to nil)
```

从程序的实现角度来讲，类似于底层没有实现 `==` 操作符

另一方面，这三种类型的相等 有歧义，所以未实现

## interface、channel 可比较

1.  channel。只有在它们引用着相同的底层内部通道或者它们都为nil时比较结果才为true
2.  interface。只有这两个接口值的动态类型相同、动态类型为可比较类型、并且动态值相等 或者都为nil 才为true

## 如果要比较slice、map时如何比较？

```go
reflect.DeepEqual
func DeepEqual(x, y interface{}) bool {
	if x == nil || y == nil {
		return x == y
	}
	v1 := ValueOf(x)
	v2 := ValueOf(y)
	if v1.Type() != v2.Type() {
		return false
	}
	return deepValueEqual(v1, v2, make(map[visit]bool), 0)
}
```

结论：如果struct中包含了 上述不能比较的类型，是不能直接比较的。

