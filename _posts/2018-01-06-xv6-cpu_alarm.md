---
layout: post
title: "XV6 Homework - CPU alarm"
description: ""
category:  IT
tags: 
  - xv6
---

```
这个 Homework 不知道做的对不对，但是跑起来感觉没什么问题 
```

[CPU alrm](https://pdos.csail.mit.edu/6.828/2017/homework/xv6-alarm.html) 这个 Homework 还是和 interrupt/fault 有关，一个区别是 handler 是用户代码，而不是内核代码。interrupt/fault 发生时，要执行用户的一段代码。(interrupt/fault 处理流程都差不多，后面统一用中断替代）

中断发生时，如果 CPU 执行的是用户代码，是需要进行内核态/用户态切换的。而 handler 涉及用户代码，刚开始的思路是这样的:

* 中断发生时，CPU 切换到内核中，保留上下文（如果发生用户态/内核态切换），判断中断类型，执行相应的 handler
* 如果 handler 是用户代码，那岂不是又要切换回用户态？
* 执行完 handler 中的用户代码，又切换回到内核，完成 handler剩余工作
* 最后切换回用户代码，继续执行用户代码的剩余部分

但是再细想，感觉不太可行。上面步骤的1和4，前面分析 syscall 的时候，已经了解怎么实现。内核态和用户态相互切换，非常复杂，要保存各种上下文。而且只接触过用户态切换到内核态，然后通过 iret 从内核态恢复到用户态，没见过内核态直接切换到用户态的，想想各种上下文（寄存器，堆栈，页表等），感觉不大现实。

# 进程执行
上面方向不行，换个思路，先理一下进程是怎么执行的。

进程运行的模型很简单，获得时间片后，恢复进程的上下文，循环两个操作，**取指令，运行**。如果时间片运行完后，暂停运行，保存上下文。

如果没有分支判断，函数调用等情况，那么取的指令就是下一条指令，这时进程一直在顺序执行指令。如果是分支，则会跳转到其它地方的指令，就不是下一条指令了。

函数调用前面已经讲过，会把函数参数压栈，把下一条指令 eip 压栈，然后跳转到函数代码处执行。

# homework
再分析一下 homework，当发生时钟中断的时候，在时钟中断的 handler 中判断当前进程是否消耗完 ticks 时间，如果消耗完时间，执行 alarm handler。

alarm handler 就一个简单的函数，也就是说，要调用这个 alarm handler 函数。那么在时钟中断的 handler 里面如果调用这个用户态的函数呢？

在时钟中断里面，我们是可以获取用户进程的上下文的，如果手动把这个 alarm handler 函数安装到进程的上下文中，让进程恢复运行的时候，就调用这个 handler，不就OK了吗？

我们已经知道汇编怎么实现函数调用，那么手动实现一个函数调用，也是很简单的事情。因为 alarm handler 没有参数，所以只需要把进程的 eip 压入进程栈，然后把进程的 eip 指向 alarm handler。那么下次进程运行的时候，就会直接执行 alarm handler。当 alarm handler 运行完，也会恢复 eip，继续执行后续的指令。这样我们就把一个函数手动注入到进程的上下文中，当函数恢复运行的时候，自动调用注入的函数。

# 实现
## 增加 alarmtest.c
需要增加一个用户程序，测试这个功能。实现逻辑 homework 已经给出，怎么增加一个用户程序，前面的 homework 已经实现过了。

## 增加 alarm syscall
首先需要增加一个 syscall 

`int alarm(int ticks, void (*handler)());`

而且 homework 也给出了 alarm 需要实现的功能，其实就是初始化进程相关的字段，一个是需要把 alarm handler 的地址保存到进程的结构体中，一个是要初始化进程消耗的时间片。

当然，进程的结构体需要增加相应字段，不然 `alarm syscall` 没办法初始化这些字段。

## 注入 alarm handler
主要在时钟中断的 handler 里面实现。判断时钟中断的时候，是不是用户进程在执行（即是不是从用户进程切换到内核）。如果是，就要判断进程是否存活，而且进程有没有 `alarm handler`（没有就不用处理）。

然后还要判断，如果没有在进程上下文注入 `alarm handler`(防止 Homework 说的 Prevent re-entrant calls to the handler)，而且时间片没有用完，就在进程上下文注入 `alarm handler`，大致逻辑如下。

```c
//用的是2016的代码，和2017有点区别

if(proc){
          if((tf->cs & 3) == 3 && proc->alarmhandler && proc->killed != 1){
              if((ticks - proc->ticks) >= proc->alarmticks && tf->eip != (uint)proc->alarmhandler){
                  tf->esp -= 4;
                  *(uint *)(tf->esp) = tf->eip;
                  tf->eip = (uint)proc->alarmhandler;
                  proc->tf = tf;
                  //cprintf("ticks %d %d %d\n", ticks , proc->ticks, (uint)proc->alarmhandler);
                  proc->ticks = ticks;
              }
          }
      }
```

# 总结
当中断发生时，如何在中断的 handler 里调用用户进程的一个函数，原理就是把这个用户函数注入到进程的上下文中。当进程恢复上下文，继续执行时，就可以先执行注入的函数，从而达到需要的效果。

很多漏洞应该也是用类似手段。比如函数返回时，会从栈中弹出以前压栈的 eip，继续执行。如果在栈中做了手脚，把栈内容改一下，让这个出栈的 eip 变成黑客的程序，不就在程序中植入病毒了吗？