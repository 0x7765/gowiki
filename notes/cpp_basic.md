# `std::endl ` vs `\n`

### 共同点

都可以输出换行。

### 不同点

1.  `\n` 表示内容为一个回车符的字符串.
2.  `std::endl` 是流操作
3.  `\n` 不会刷新缓冲区
4.  `std::endl` 会刷新缓冲区
5.  频繁刷新缓冲区可能有性能问题，耗时敏感的情况下 尽量使用`\\n`

```cpp
std::cout << "Hello, World!" << std::endl;

// 等同于 
std::cout << "Hello, World!\\n" << std::flush;
```



## `NULL` vs `nullptr`

1.  `NULL` 是宏替换 `#define NULL 0`
2.  `nullptr 是空指针 (指针类型)

