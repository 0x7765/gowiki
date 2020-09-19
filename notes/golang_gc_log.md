# gc log

> `golang`的gc日志主要包含两部分。 一部分是 gc周期中的耗时和内存变化信息。另一部分是与操作系统交互的内存信息。

# part-one

##  字段说明 

```txt
gc 7 @0.029s 3%: 0.047+0.38+0.042 ms clock, 0.75+0.064/0.73/0+0.67 ms cpu, 4->4->0 MB, 5 MB goal, 16 P
gc 8 @0.031s 3%: 0.040+0.51+0.032 ms clock, 0.64+0.21/0.82/0.001+0.52 ms cpu, 4->4->0 MB, 5 MB goal, 16 P
gc 9430 @100.008s 3%: 0.079+0.85+0.013 ms clock, 1.2+0/1.7/3.0+0.21 ms cpu, 1->1->1 MB, 2 MB goal, 16 P (forced)
gc 9431 @100.010s 3%: 0.026+0.44+0.006 ms clock, 0.43+0/1.0/1.5+0.10 ms cpu, 1->1->1 MB, 2 MB goal, 16 P (forced)
```



| 序号 | 字段      | 说明                                                     |
| ---- | --------- | -------------------------------------------------------- |
| 1    | gc 7      | 第7次gc(第七个gc周期)                                    |
| 2    | @0.029s   | 程序启动到现在的时间 0.029s                              |
| 3    | 3%        | 该gc周期中的cpu使用率                                    |
| 4    | 0.047     | 标记开始时 stw 所花费的时间 **wall clock**               |
| 5    | 0.38      | 标记过程中，并发标记所花费的时间 **wall clock**          |
| 6    | 0.042     | 标记终止时， STW 所花费的时间 **wall clock**             |
| 7    | 0.75      | 标记开始时， STW 所花费的时间 **cpu time**               |
| 8    | 0.064     | 标记过程中，标记辅助所花费的时间 **cpu time**            |
| 9    | 0.73      | 标记过程中，并发标记所花费的时间 **cpu time**            |
| 10   | 0         | 标记过程中，GC 空闲的时间 **cpu time**                   |
| 11   | 0.67      | 标记终止时， STW 所花费的时间**cpu time**                |
| 12   | 4         | 标记开始时，堆的大小的实际值                             |
| 13   | 4         | 标记结束时，堆的大小的实际值                             |
| 14   | 0         | 标记结束时，标记为存活的对象大小                         |
| 15   | 5 MB goal | 标记结束时，堆的大小的预测值                             |
| 16   | 16 P      | P 的数量                                                 |
| 17   | forced    | 强制触发的gc `runtime.GC() ` 或者 `debug.FreeOSMemory()` |

## 时间说明 

> - **wall clock 是程序实际的执行时间。包含从开始执行到完成，包括本程序及其他程序占用(比如io等待、等待用户输入等)所消耗的时间**
> - **cpu time 是指程序占用 CPU 的时间**
> - **程序不涉及网络io等情况下(cpu密集型应用)，一般满足以下关系 。而io密集型程序，一般来说 wall clock 是大于 cpu time 的**
>   - wall_time < cpu_time 充分利用多核优势 
>   - wall clock ≈ cpu time 多核优势不明显或者未开启并行执行
>   - wall clock > cpu time  无法利用多核优势 或者 存在类似block在io等待的情况

# part-two

## 字段说明 

```txt
scvg: 8 KB released
scvg: inuse: 3, idle: 60, sys: 63, released: 57, consumed: 6 (MB)
```

**scvg** 是 **scavenging ** 的缩写  原意是 清除 

| 序号 | 字段          | 说明                                                         |
| ---- | ------------- | ------------------------------------------------------------ |
| 1    | 8 KB released | 向操作系统归还了 8 KB 内存                                   |
| 2    | inuse: 4      | 已经分配给用户代码、正在使用的总内存大小 (MB)。MB used or partially used spans |
| 3    | idle: 60      | 空闲以及等待归还给操作系统的总内存大小（MB）。MB spans pending scavenging |
| 4    | sys: 64       | 通知操作系统中保留的内存大小（MB）MB mapped from the system  |
| 5    | released: 57  | 已经归还给操作系统的（或者说还未正式申请）的内存大小（MB）。MB released to the system |
| 6    | consumed: 6   | 已经从操作系统中申请的内存大小（MB）。MB allocated from the system |

