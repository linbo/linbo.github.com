---
layout: post
title: "XV6 Homework - Lazy page allocation"
description: ""
category:  IT
tags: 
  - xv6
---

进程的地址空间除了进程的代码，数据，栈外，还有堆内存。其中堆内存是按需分配的，当进程需要的时候，通过调用 `malloc` 标准库函数，向操作系统动态申请物理堆内存。

# Part one
Homework 要求修改 `sbrk syscall`，去掉实际分配内存的代码，进入 shell 后运行命令会发送错误。

## sbrk 禁止内存分配
作业很简单，注释掉内存分配代码 `growproc`（但是别忘记修改进程的大小)，运行 shell 命令，出现类似错误

```bash
$ echo hi
pid 3 sh: trap 14 err 6 on cpu 0 eip 0x13bf addr 0x4004--kill proc
```
当 CPU 访问内存时，MMU 将线性地址转换为物理地址，期间会访问页目录表 -> 页表 -> 页。如果页表或者页不存在（table entry 的 P 位为0），将会导致 page fault，会产生中断号为14的中断，`trap.c:trap` 没有处理这个中断，只是简单的打印相应的信息。

为什么执行 shell 命令的时候，会发生 page fault，查看 `sh.c`，发现有通过 `malloc` 动态申请内存的地方，比如

```c
struct cmd*
execcmd(void)
{
  struct execcmd *cmd;

  cmd = malloc(sizeof(*cmd))
```

为什么注释掉 `sys_sbrk` 分配内存的代码，调用 malloc 就会发生 page fault？下面来看看 malloc/free 的实现机制
## malloc/free
当进程需要动态申请内存时，是调用 syscall:sbrk 向操作系统申请内存，不用的时候调用 syscall:free 向操作系统释放内存。如果每次都调用 syscall，性能会受到影响。具体原因可以参考 syscall 的分析，每次发生系统调用，都需要保存 user space 的上下文，切换栈和上下文等耗时操作。所以 C 语言标准库会提供相应函数，用来管理进程的堆内存，提高内存使用的效率。

这两个库函数来自 TCPL [^1] 8.7 节，具体实现就是通过链表来管理动态分配的内存。链表头 `base` 存放在进程的数据段中，链表的节点是在堆中。

![malloc]({{ site.url }}/assets/post/xv6/malloc.jpeg)

需要注意的是

* 空闲块包括 `header` 和 后面的空闲空间
* 空闲块大小是 `header` 大小的整数倍
* `malloc` 返回指针指向的是 `header` 后面的空闲空间的地址

所以 malloc 分配的内存，`header` 部分进程用不到的，**利用率有一定的损失**。

具体代码解析参考 TCPL。这里只关注和 `sbrk` 相关的代码，在 `morecore` 中，进程会调用 `sbrk` 向 kernel 申请内存，如果成功则返回新内存的首地址。

`sysproc.c:sys_sbrk` 会调用 `proc.c:growproc` 分配内存，逻辑比较简单

* 调用 kalloc 申请物理页内存
* 建立虚拟内存和物理内存的映射关系
* 更新 CPU 的段寄存器，TSS，重新加载页表

**在进程的虚拟地址空间里面，进程的代码，数据，栈，堆内存是连续的**。每个进程的进程信息里面都有一个字段，表示进程的大小。每次新申请内存的首地址，就是申请前进程的大小。申请成功后，需要将进程大小更新。

## page fault 解析
回到前面的问题，为什么注释 `sysproc.c:sys_sbrk` 的 `growproc`，执行 shell 命令的时候，发生了 page fault。

malloc 如果没有空闲内存，会向 kernel 申请，虽然 malloc 返回了申请内存的首地址，但是其实没有物理内存分配出来。当访问一个线性地址时，CPU会访问页目录和页表，如果页目录项或者页表项的 P 位为0，表示当前页表或者页不存在，就会发生 page fault。

具体看一下 `umalloc:morecore` 代码

```
static Header*
morecore(uint nu)
{
  char *p;
  Header *hp;

  if(nu < 4096)
    nu = 4096;
  p = sbrk(nu * sizeof(Header));
  if(p == (char*)-1)
    return 0;
  hp = (Header*)p;
  hp->s.size = nu;
  free((void*)(hp + 1));
  return freep;
}
```

前面说到虽然首地址返回了，但是并没有物理内存分配出来，访问这段内存就会出现 page fault。代码有初始化 `header` 结构体字段的逻辑，正是这行代码导致了 page fault。

```
  hp->s.size = nu;
```

查看 `sh.asm`，发现这行代码地址是 `0x13bf`，shell 命令异常退出的信息显示，正是 `eip` 这段代码导致了 page fault。

```
  p = sbrk(nu * sizeof(Header));
    1395:	8b 45 08             	mov    0x8(%ebp),%eax
    1398:	c1 e0 03             	shl    $0x3,%eax
    139b:	89 04 24             	mov    %eax,(%esp)
    139e:	e8 51 fc ff ff       	call   ff4 <sbrk>
    13a3:	89 45 f4             	mov    %eax,-0xc(%ebp)
  if(p == (char*)-1)
    13a6:	83 7d f4 ff          	cmpl   $0xffffffff,-0xc(%ebp)
    13aa:	75 07                	jne    13b3 <morecore+0x34>
    return 0;
    13ac:	b8 00 00 00 00       	mov    $0x0,%eax
    13b1:	eb 22                	jmp    13d5 <morecore+0x56>
  hp = (Header*)p;
    13b3:	8b 45 f4             	mov    -0xc(%ebp),%eax
    13b6:	89 45 f0             	mov    %eax,-0x10(%ebp)
  hp->s.size = nu;
    13b9:	8b 45 f0             	mov    -0x10(%ebp),%eax
    13bc:	8b 55 08             	mov    0x8(%ebp),%edx
    13bf:	89 50 04             	mov    %edx,0x4(%eax)
```
 
 虽然汇编显示代码多次用到了 `sbrk` 返回的地址，但都是比较或者存储这个地址本身，并没有访问这个地址所指的内存，只有 `mov %edx, 0x4(%eax)` 访问了 这个地址+4 处的内容，而这个地址的物理页是不存在的。

但是细心的同学会发现，发生中断时， eip 是指向产生中断代码的下一行指令，但是这里的 eip 却是产生中断的那条指令。

前面没有讲太清楚，`page fault` 其实是属于 Exception，而且是 `Faults` 类型的 Exception，这种类型的 Exception，eip 是指向当前行，原因是这种异常是可以恢复的，恢复后应该继续尝试刚才出错的指令。比如 page fault 后，kernel 进行 lazy allocation，完成后应该继续执行刚才那条 `page fault` 的指令，按上面的例子，就是完成 `hp->s.size = nu` 赋值操作。

Exception 相应的解释，可以参考 《Intel(R) 64 and IA-32 Architectures Software Developer’s Manuals-vol-3A》[^2] 

不管是 interrupt，还是 exception，xv6 都采用同样的机制处理，所以 `page fault` 和 syscall 的代码逻辑基本上一致。
# Part two
第二部分要求实现 `Lazy allocation`，有了前面的分析，这一部分难度不是很大。

首先发生 `page fault` 的时候，系统会产生中断，中断号是 14

```
#define T_PGFLT         14      // page faul
```

在系统调用的文章中，已经分析了中断处理流程，这里只需要关注 `interrupt handler`，代码在 `trap.c:trap` 中。

在 `switch` 语句的 `default` 分支中，需要处理 `T_PGFLT` 中断。在 `page allocte` 之前，需要知道哪个线性地址发生了 `page fault`，这个地址是存放在 `cr2` 寄存器上的。

后面的逻辑和 `proc.c:growproc` 相差不大，分配物理内存，建立虚拟内存和物理内存的映射关系，更新 CPU 相应的状态和进程的页表信息。

处理完这些流程后，`page fault handler` 就应该返回，当 `trap.c:trap()` 返回后，`trapasm.S` 就会恢复发生 `page fault` 的上下文，如果是 malloc 发生了 `page fault`，那么就切换栈，切换回用户空间，继续执行发生 `page fault` 的指令。

# 总结
因为 `syscall` 消耗性能，所以 C 标准库提供 `malloc/free` 来管理用户使用的堆内存。但是当用户申请堆内存后，不一定会立即使用，为了提高性能，`homework` 采用 `lazy allocation` 机制。用户申请堆内存的时候，`kernel` 实际没有分配物理内存，而是在需要的时候才会分配。这样就尽量减少不必要的 `syscall`，提高性能。

具体逻辑就是，用户申请内存时，不分配物理内存。当实际用到物理内存时，发生 `page fault` 的时候，产生 `Faults` 类型的 `Exception` (中断号14 T_PGFLT) 。在 中断/异常 `handler(trap.c:trap())` 中捕获 `page fault`，实现 `lazy allocation`。

# Reference
[^1]: [C程序设计语言](https://book.douban.com/subject/1139336/)
[^2]: [Intel(R) 64 and IA-32 Architectures Software Developer’s Manuals-vol-3A](https://software.intel.com/en-us/articles/intel-sdm) 第6.5节: EXCEPTION CLASSIFICATIONS