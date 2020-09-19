# asm

## 伪汇编

>   Go 语言编译器生成的汇编是一种平台无关、可移植的汇编代码。并不与某种具体的硬件架构相对应。
>
>   Go的汇编器会使用这种伪汇编，再对目标硬件生成具体的机器指令。

# Hello World

## 准备

```GO
// add.go
package main

//go:noinline
func add(a, b int32) (int32, bool) { return a + b, true }

func main() { add(11, 7) }
```

注意 这里取消内联 `//go:noinline`

生成汇编代码

```shell
GOOS=linux GOARCH=amd64 go tool compile -S add.go > add.s
```

生成的汇编代码如下 (只截取的add函数的部分，剩下的部分后面分析)

```assembly
"".add STEXT nosplit size=20 args=0x10 locals=0x0
	0x0000 00000 (add.go:4)	TEXT	"".add(SB), NOSPLIT|ABIInternal, $0-16
	0x0000 00000 (add.go:4)	PCDATA	$0, $-2
	0x0000 00000 (add.go:4)	PCDATA	$1, $-2
	0x0000 00000 (add.go:4)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (add.go:4)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (add.go:4)	FUNCDATA	$2, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (add.go:4)	PCDATA	$0, $0
	0x0000 00000 (add.go:4)	PCDATA	$1, $0
	0x0000 00000 (add.go:4)	MOVL	"".b+12(SP), AX
	0x0004 00004 (add.go:4)	MOVL	"".a+8(SP), CX
	0x0008 00008 (add.go:4)	ADDL	CX, AX
	0x000a 00010 (add.go:4)	MOVL	AX, "".~r2+16(SP)
	0x000e 00014 (add.go:4)	MOVB	$1, "".~r3+20(SP)
	0x0013 00019 (add.go:4)	RET
	0x0000 8b 44 24 0c 8b 4c 24 08 01 c8 89 44 24 10 c6 44  .D$..L$....D$..D
	0x0010 24 14 01 c3                                      $...
```

## 逐行说明

1.  `0x0000 00000 (add.go:4)	TEXT	"".add(SB), NOSPLIT|ABIInternal, $0-16`

    1.  `0x0000 00000 ` 当前指令相对于当前函数的偏移量 

    2.  `TEXT	"".add` `TEXT` 指令声明了 `"".add` 是 `.text` 段(程序代码在运行期会放在内存的 .text 段中)的一部分. 并标明跟在这个之后的是函数的函数体。在链接器, `""` 会被替换成package name。也就是`main.add`

    3.  `(SB)` 是一个虚拟寄存器，保存了静态基地址(static-base) 指针，即我们程序地址空间的开始地址。`"".add(SB)` 表明我们的符号(mian.add)位于某个固定的相对地址空间起始处的偏移位置 (最终是由链接器计算得到的).也就是说 它有一个直接的绝对地址。

        >   所有用户定义的符号都被写为相对于伪寄存器FP(参数以及局部值)和SB(全局值)的偏移量.
        >
        >   SB伪寄存器可以被认为是内存的起始位置，所以对于符号foo(SB)就是名称foo在内存的地址.

    4.  `NOSPLIT` 向编译器表明*不应该*插入 *stack-split* 的用来检查栈需要扩张的前导指令.在我们 `add` 函数的这种情况下，编译器自己帮我们插入了这个标记: 它足够聪明地意识到，由于 `add` 没有任何局部变量且没有它自己的栈帧，所以一定不会超出当前的栈；因此每次调用函数时在这里执行栈检查就是完全浪费 CPU 循环了.

    5.  `$0-16` `$0` 即将分配的栈帧的大小。`-`是分隔符,16 是函数的参数的大小。

        >   在这里，由于函数没有申请局部变量，所以栈帧大小是0
        >
        >   16, 64位平台上，int32 是4个字节。两个int32  大小是16字节。

2.  `PCDATA` 和`FUNCDATA` 

    >   FUNCDATA以及PCDATA指令包含有被垃圾回收所使用的信息；这些指令是被编译器加入的. 后序单独分析 

3.  `MOVL	"".b+12(SP), AX` 和 `MOVL	"".a+8(SP), CX`

    >   a. Go 的调用规约要求每一个参数都通过栈来传递，这部分空间由 caller 在其栈帧(stack frame)上提供。
    >
    >   b. 调用其它函数之前，caller 就需要按照参数和返回变量的大小来对应地增长(返回后收缩)栈。
    >
    >   c. Go 编译器不会生成任何 PUSH/POP 族的指令: 栈的增长和收缩是通过在栈指针寄存器 `SP` 上分别执行减法和加法指令来实现的。
    >
    >   d. SP伪寄存器是虚拟的栈指针，用于引用帧局部变量以及为函数调用准备的参数。 它指向局部栈帧的顶部，所以应用应该使用负的偏移且范围在[-framesize, 0): x-8(SP), y-4(SP), 等等。
    >
    >   e. 参数是反序传入的
    >
    >   f. 第一个参数的地址不是0(SP). 这是因为调用方(caller)将返回值保存在0(SP)的位置。 参数实际的地址是8(SP),因为返回值有两个，一个是四字节的int32，另一个是bool ，四字节是为了对齐 ？

    1.  `.b`、`.a` 参数名 .
    2.  `"".b+12(SP)` 参数二 指向栈的低12字节位置 (栈朝向低位地址方向增长)
    3.  `"".a+8(SP)` 参数一 指向栈的低8字节位置 
    4.  `MOVL	"".b+12(SP), AX` 将栈低12字节位置的数据赋值到ax寄存器上
    5.  `MOVL	"".a+8(SP), CX` 将栈低8字节位置的数据赋值到cx寄存器上

4.  `ADDL	CX, AX` 和`MOVL	AX, "".~r2+16(SP)` 以及 `MOVB	$1, "".~r3+20(SP)`

    1.  将cx、ax寄存器上的值相加
    2.  相加的结果 移动到 `16(SP)`的位置，``"".~r2|r3` 没有实际意义
    3.  返回值 `true`  移动到 `20(SP)`的位置  

5.  `RET` jump到返回值 `0(SP)`

## 简化后的过程

```assembly
"".add STEXT nosplit size=20 args=0x10 locals=0x0
	;; Declare global function symbol "".add (actually main.add once linked)
	;; Do not insert stack-split preamble
	;; 0 bytes of stack-frame, 16 bytes of arguments passed in
	;; func add(a, b int32) (int32, bool)
	0x0000 00000 (add.go:4)	TEXT	"".add(SB), NOSPLIT|ABIInternal, $0-16
	0x0000 00000 (add.go:4)	MOVL	"".b+12(SP), AX  ;; move second Long-word (4B) argument from caller's stack-frame into AX
	0x0004 00004 (add.go:4)	MOVL	"".a+8(SP), CX ;; move first Long-word (4B) argument from caller's stack-frame into CX
	0x0008 00008 (add.go:4)	ADDL	CX, AX ;; compute AX=CX+AX
	0x000a 00010 (add.go:4)	MOVL	AX, "".~r2+16(SP) ;; move addition result (AX) into caller's stack-frame
	0x000e 00014 (add.go:4)	MOVB	$1, "".~r3+20(SP) ;; move `true` boolean (constant) into caller's stack-frame
	0x0013 00019 (add.go:4)	RET                     ;; jump to return address stored at 0(SP)
```

## 栈帧的结构

```text
   |    +-------------------------+ <-- 32(SP)
   |    |                         |
 G |    |                         |
 R |    |                         |
 O |    | main.main's saved       |
 W |    |     frame-pointer (BP)  |
 S |    |-------------------------| <-- 24(SP)
   |    |      [alignment]        |
 D |    | "".~r3 (bool) = 1/true  | <-- 21(SP)
 O |    |-------------------------| <-- 20(SP)
 W |    |                         |
 N |    | "".~r2 (int32) = 42     |
 W |    |-------------------------| <-- 16(SP)
 A |    |                         |
 R |    | "".b (int32) = 32       |
 D |    |-------------------------| <-- 12(SP)
 S |    |                         |
   |    | "".a (int32) = 10       |
   |    |-------------------------| <-- 8(SP)
   |    |                         |
   |    |                         |
   |    |                         |
 \ | /  | return address to       |
  \|/   |     main.main + 0x30    |
   -    +-------------------------+ <-- 0(SP) (TOP OF STACK)
```

