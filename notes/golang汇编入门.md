# golang 汇编入门

[参考地址](http://xargin.com/plan9-assembly/)

## 基本说明

**基本格式**

```plan9_x86
TEXT pkgname·funname(SB),NOSPLIT,param_return_value_size
```

```text
                              参数及返回值大小
                                  | 
 TEXT pkgname·add(SB),NOSPLIT,$32-32
       |        |               |
      包名     函数名         栈帧大小(局部变量+可能需要的额外调用函数的参数空间的总大小，但不包括调用其它函数时的 ret address 的大小)
```

## 最小的例子

```plan9_x86
// func Add(a, b int) int
TEXT ·Add(SB),$0-24 
    MOVQ a+0(FP), BX // 参数a
    MOVQ b+8(FP), BP // 参数b
    ADDQ BP, BX  // BX += BP
    MOVQ BX, ret+16(FP) // 返回
    RET

```

# 入门

## 寄存器

1. 主要使用的右14个寄存器 rax, rbx, rcx, rdx, rdi, rsi, r8~r15 这 14 个寄存器。
2. rbp 和 rsp 一般用于管理栈顶和栈底，尽量不用于计算
3. plan9 中使用寄存器不需要带 r 或 e 的前缀

## 伪寄存器

1. 有四个 分别是 FP、PC、SB、SP
2. FP: 使用形如 `symbol+offset(FP)` 的方式，引用函数的输入参数。例如 arg0+0(FP)，arg1+8(FP)，使用 FP 不加 symbol 时，无法通过编译，在汇编层面来讲，symbol 并没有什么用，加 symbol 主要是为了提升代码可读性
3. PC: 实际上就是在体系结构的知识中常见的 pc 寄存器。用于计数，存放指令的位置
4. SB: 全局静态基指针，一般用来声明函数或全局变量
5. SP: plan9 的这个 SP 寄存器指向当前栈帧的局部变量的开始位置，使用形如 `symbol+offset(SP)` 的方式，引用函数的局部变量。offset 的合法取值是 [-framesize, 0)，注意是个左闭右开的区间。假如局部变量都是 8 字节，那么第一个局部变量就可以用 localvar0-8(SP) 来表示

## 说明

- **`plan9` 中 `TEXT` 是一个指令，用来定义一个函数**
- 沒有局部变量 栈桢是0,两个int型参数的大小是8个字节，返回值一个int 一共是3*8=24 个字节
- 最后一行的空行是必须的，否则可能报 unexpected EOF

## 指令说明
1. `MOVQ` 是数据搬运指令 MOVB 一个字节 MOVW 两个字节 MOVD  四个字节 MOVQ  八个字节
2. `ADDQ` 加法指令

## 代码说明
```plan9_x86
TEXT ·Add(SB),$0-24  //没有局部变量,栈桢是0 两个参数+返回值=3*8=24个字节
    MOVQ a+0(FP), BX // 参数a a+0(FP) 引用输入参数a 赋值到BX寄存器上
    MOVQ b+8(FP), BP // 参数b b+8(FP) 引用输入参数b 赋值到BP寄存器上
    ADDQ BP, BX  // BX += BP 操作这俩寄存器 
    MOVQ BX, ret+16(FP) // 返回 
    RET

```
