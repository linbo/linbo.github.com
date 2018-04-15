---
layout: post
title: "XV6 - First Process(1)"
description: ""
category:  IT
tags: 
  - xv6
---

# 进程信息
## 地址空间
进程地址空间分成三部分：

* 用户内存  
用户内存位于 `0 ～ 0x7FFFFFFF`，从底向上依次是  
 1. 进程的指令和数据
 2. 进程的栈
 3. 进程的堆

* BIOS  
BIOS 被映射到 `0x80000000 ~ 0x800FFFFF`处

* Kernel 区域  
内核指令和数据映射到 `0x80100000 ~ 0xFFFFFFFF`。当进程使用系统调用时，系统调用会在进程地址空间的内核区域运行，这样可以使得系统调用代码直接指向用户内存。

**发生系统调用时，所有代码（包括内核代码）都在同一个进程的地址空间中，所以不需要切换页表**

## 进程状态
`proc.h`的结构体维护了进程的状态

```c
// Per-process state
struct proc {
  uint sz;                     // Size of process memory (bytes)
  pde_t* pgdir;                // Page table
  char *kstack;                // Bottom of kernel stack for this process
  enum procstate state;        // Process state
  int pid;                     // Process ID
  struct proc *parent;         // Parent process
  struct trapframe *tf;        // Trap frame for current syscall
  struct context *context;     // swtch() here to run process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

每个进程都有一个线程用来执行进程的指令，系统在进程间切换，就是切换进程内的线程，线程的大部分状态保存在线程的栈上。

每个进程有用户栈和内核栈，当执行用户指令时，使用用户栈，当进程通过系统调用和中断进入内核时，切换用户栈到内核栈。

其它重要的状态有如下这些   

* 页表 `proc->pgdir`
* 进程状态 `proc->state`
* 进程上下文切换的 Trap frame `proc->tf` //syscall有具体说明
* 切换进程的上下文信息 `proc->context`

# 创建第一个用户进程
`userinit` 用来创建第一个用户进程，主要工作就是分配内存，设置 `struct proc` 结构体相关的信息。

`userinit` 调用 `allocproc` 分配 `struct proc`，并设置相关字段。xv6 维护一个 `struct proc` 数组，当创建新进程的时候，找到表中未用的元素，用来存放当前进程的 `struct proc`。如果没有找到，返回 NULL指针。如果表中有可用的元素，接下来就是设置 `struct proc`的相关字段，首先设置 pid 和 进程状态，然后分配内核堆栈内存，并初始化内核堆栈。

### 初始化内核堆栈
进程对应的内核堆栈结构如下

![func]({{ site.url }}/assets/post/xv6/kstack.png)

内核堆栈从底向上分成三部分  

* `struct trapframe`: 系统调用或者中断发生时，需要保存的信息
* `trapret`
* `struct context`: 进程切换需要保存的上下文

`allocproc`只是分配了内核堆栈，并在 `struct proc` 中设置 `trapframe` 和 `context` 初始位置。同时清零 `context`，设置 `context->eip` 为 `forkret`。

### 初始化进程运行需要的信息
第一个进程为 `initcode.S`，链接器将这个代码嵌入到内核中，并定义两个特殊的符号：`_binary_initcode_start`(指令开始位置），`_binary_initcode_size`（进程大小）。 

`userinit` 调用 `setupkvm` 创建页表，映射内核代码到用户的地址空间。然后调用 `inituvm` 分配物理内存，将程序拷贝到物理内存，创建用户程序的页表映射。下面代码可以知道，程序被映射到虚拟地址 0 开始的位置，所以第一条指令的地址是 0。

```c
mappages(pgdir, 0, PGSIZE, V2P(mem), PTE_W|PTE_U);
```

接下来初始化 `struct trapframe`，主要是段寄存器，用户栈相关的寄存器，状态寄存器和 eip。最后将进程状态设置为 `RUNNABLE`，等待内核调度线程调度，获取 CPU 运行。


# 进程调度器
到现在为止，CPU 运行的所有代码都是内核代码，包括前面的进程创建代码。接下来，每个 CPU 会起一个调度器，找到一个 `RUNNABLE` 进程，切换当前内核调度器到可运行的用户线程上，运行用户进程。

`main.c:mpmain(void)` 会调用 `proc.c:scheduler(void)` ，运行调度器。调度器是一个死循环，它查找 `proc` 数组，找到可运行的进程，切换内核调度器到可运行的进程并运行，并设置进程状态为 `RUNNING`。

## 设置 CPU 
调度器首先设置当前 CPU 的一些信息，这部分在系统调用写过，重新贴一下。

xv6 会为每个 CPU 创建一些 CPU 的状态信息，具体为 `proc.h:struct cpu`。在每个 CPU 结构体中有两个字段，一个是 GDT，一个是 TSS。其中 TSS 描述符会安装到 GDT 中。

```c
struct cpu {
  ...
  struct taskstate ts;         // Used by x86 to find stack for interrupt
  struct segdesc gdt[NSEGS];   // x86 global descriptor table
  ...
 }
```

每个进程运行之前，内核会设置这些数据结构，具体代码在 `vm.c:switchuvm()`，主要是安装 TSS 描述符到 GDT中，然后设置 TSS 的内核栈和 iomb，其它3个特权级的栈信息和其它信息都没有用到。其中 TSS 的内核栈就是内核为进程创建的内核栈。最后装载 TSS 段，并切换页表。

```c
  cpu->gdt[SEG_TSS] = SEG16(STS_T32A, &cpu->ts, sizeof(cpu->ts)-1, 0);
  cpu->gdt[SEG_TSS].s = 0;
  cpu->ts.ss0 = SEG_KDATA << 3;
  cpu->ts.esp0 = (uint)p->kstack + KSTACKSIZE;
  // setting IOPL=0 in eflags *and* iomb beyond the tss segment limit
  // forbids I/O instructions (e.g., inb and outb) from user space
  cpu->ts.iomb = (ushort) 0xFFFF;
  ltr(SEG_TSS << 3);
  lcr3(V2P(p->pgdir));  // switch to process's address space
```


这里载入了进程的页表，为什么后面代码还能继续运行呢？因为对于内核物理内存，在内核的地址空间和进程地址空间，都被映射到相同的地方，即从 `0x08100000 ~ 0xFFFFFFFF`

### 内核初始页表
先看看内核初始页表内容

```bash
(gdb) b 22
Breakpoint 1 at 0x801038ab: file main.c, line 22.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x801038ab <main+34>:	call   0x80103cae <mpinit>

Breakpoint 1, main () at main.c:22
22	  mpinit();        // detect other processors
```

```bash
(qemu) info pg
VPN range     Entry         Flags        Physical page
[80000-803ff]  PDE[200]     ----A--UWP
  [80000-800ff]  PTE[000-0ff] --------WP 00000-000ff
  [80100-80102]  PTE[100-102] ---------P 00100-00102
  [80103-80103]  PTE[103]     ----A----P 00103
  [80104-80106]  PTE[104-106] ---------P 00104-00106
  [80107-80108]  PTE[107-108] ----A----P 00107-00108
  [80109-80109]  PTE[109]     ---------P 00109
  [8010a-8010c]  PTE[10a-10c] --------WP 0010a-0010c
  [8010d-8010d]  PTE[10d]     ----A---WP 0010d
  [8010e-803ff]  PTE[10e-3ff] --------WP 0010e-003ff
[80400-8dfff]  PDE[201-237] -------UWP
  [80400-8dfff]  PTE[000-3ff] --------WP 00400-0dfff
[fe000-fffff]  PDE[3f8-3ff] -------UWP
  [fe000-fffff]  PTE[000-3ff] --------WP fe000-fffff
(qemu) info registers
...
CR0=80010011 CR2=00000000 CR3=003ff000 CR4=00000010 
...
```

### 没有切换页表前
在调用 `vm.c:switchuvm()` 前，可以看到页表很多熟悉已经被改变了，有些被访问过了(A被置位)，有些页面已经被修改(D被置位)

```bash
(qemu) info registers
...
CR0=80010011 CR2=00000000 CR3=003ff000 CR4=00000010
...
(qemu) info pg
VPN range     Entry         Flags        Physical page
[80000-803ff]  PDE[200]     ----A--UWP
  [80000-80000]  PTE[000]     ----A---WP 00000
  [80001-80006]  PTE[001-006] --------WP 00001-00006
  [80007-80007]  PTE[007]     ---DA---WP 00007
  [80008-8009e]  PTE[008-09e] --------WP 00008-0009e
  [8009f-8009f]  PTE[09f]     ----A---WP 0009f
  [800a0-800b7]  PTE[0a0-0b7] --------WP 000a0-000b7
  [800b8-800b8]  PTE[0b8]     ---DA---WP 000b8
  [800b9-800ef]  PTE[0b9-0ef] --------WP 000b9-000ef
  [800f0-800f1]  PTE[0f0-0f1] ----A---WP 000f0-000f1
  [800f2-800ff]  PTE[0f2-0ff] --------WP 000f2-000ff
  [80100-80108]  PTE[100-108] ----A----P 00100-00108
  [80109-80109]  PTE[109]     ---------P 00109
  [8010a-8010a]  PTE[10a]     ----A---WP 0010a
  [8010b-8010b]  PTE[10b]     --------WP 0010b
  [8010c-80112]  PTE[10c-112] ---DA---WP 0010c-00112
  [80113-80113]  PTE[113]     ----A---WP 00113
  [80114-80114]  PTE[114]     ---DA---WP 00114
  [80115-80116]  PTE[115-116] ----A---WP 00115-00116
  [80117-80117]  PTE[117]     ---DA---WP 00117
  [80118-803ff]  PTE[118-3ff] --------WP 00118-003ff
[80400-8dfff]  PDE[201-237] ----A--UWP
  [80400-8dfff]  PTE[000-3ff] ---DA---WP 00400-0dfff
[fe000-febff]  PDE[3f8-3fa] -------UWP
  [fe000-febff]  PTE[000-3ff] --------WP fe000-febff
[fec00-fefff]  PDE[3fb]     ----A--UWP
  [fec00-fec00]  PTE[000]     ---DA---WP fec00
  [fec01-fedff]  PTE[001-1ff] --------WP fec01-fedff
  [fee00-fee00]  PTE[200]     ---DA---WP fee00
  [fee01-fefff]  PTE[201-3ff] --------WP fee01-fefff
[ff000-fffff]  PDE[3fc-3ff] -------UWP
  [ff000-fffff]  PTE[000-3ff] --------WP ff000-fffff
```

###切换页表后
`CR3` 变化了，但是页表映射内核的部分没有变化，而且页表属性和内核初始化页表差不多。

```bash
(qemu) info registers
...
CR0=80010011 CR2=00000000 CR3=0dffe000 CR4=00000010
...
(qemu) info pg
VPN range     Entry         Flags        Physical page
[00000-003ff]  PDE[000]     -------UWP
  [00000-00000]  PTE[000]     -------UWP 0dfbd
[80000-803ff]  PDE[200]     ----A--UWP
  [80000-800ff]  PTE[000-0ff] --------WP 00000-000ff
  [80100-80103]  PTE[100-103] ---------P 00100-00103
  [80104-80105]  PTE[104-105] ----A----P 00104-00105
  [80106-80106]  PTE[106]     ---------P 00106
  [80107-80108]  PTE[107-108] ----A----P 00107-00108
  [80109-80109]  PTE[109]     ---------P 00109
  [8010a-8010c]  PTE[10a-10c] --------WP 0010a-0010c
  [8010d-8010d]  PTE[10d]     ---DA---WP 0010d
  [8010e-80113]  PTE[10e-113] --------WP 0010e-00113
  [80114-80114]  PTE[114]     ---DA---WP 00114
  [80115-803ff]  PTE[115-3ff] --------WP 00115-003ff
[80400-8dfff]  PDE[201-237] -------UWP
  [80400-8dfff]  PTE[000-3ff] --------WP 00400-0dfff
[fe000-fffff]  PDE[3f8-3ff] -------UWP
  [fe000-fffff]  PTE[000-3ff] --------WP fe000-fffff
```  

## 调度
`swtch` 调用的是 `swtch.S` 的汇编代码，因为 `swtch` 是函数调用，所以内核堆栈会把参数和 `eip` 压栈，然后跳转到 `swtch.S` 代码处。

### 调用 swtch

```bash
=> 0x80104ac3 <scheduler+73>:	mov    -0xc(%ebp),%eax
303	      swtch(&cpu->scheduler, p->context);
(gdb) si
=> 0x80104ac6 <scheduler+76>:	mov    0x1c(%eax),%eax
0x80104ac6	303	      swtch(&cpu->scheduler, p->context);
(gdb) si
=> 0x80104ac9 <scheduler+79>:	mov    %gs:0x0,%edx
0x80104ac9	303	      swtch(&cpu->scheduler, p->context);
(gdb) si
=> 0x80104ad0 <scheduler+86>:	add    $0x4,%edx
0x80104ad0	303	      swtch(&cpu->scheduler, p->context);
(gdb) si
=> 0x80104ad3 <scheduler+89>:	mov    %eax,0x4(%esp)
0x80104ad3	303	      swtch(&cpu->scheduler, p->context);
(gdb) si
=> 0x80104ad7 <scheduler+93>:	mov    %edx,(%esp)
0x80104ad7	303	      swtch(&cpu->scheduler, p->context);
(gdb) si
=> 0x80104ada <scheduler+96>:	call   0x80105528
0x80104ada	303	      swtch(&cpu->scheduler, p->context);
(gdb) info reg
eax            0x8dffff9c	-1912602724
ecx            0x40	64
edx            0x80114844	-2146351036
ebx            0x10074	65652
esp            0x8010d600	0x8010d600
ebp            0x8010d628	0x8010d628
esi            0x0	0
edi            0x1178c8	1145032
eip            0x80104ada	0x80104ada <scheduler+96>
eflags         0x86	[ PF SF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x0	0
gs             0x18	24
(gdb) s
=> 0x80105528:	mov    0x4(%esp),%eax
?? () at swtch.S:10
10	  movl 4(%esp), %eax
(gdb) info reg
eax            0x8dffff9c	-1912602724
ecx            0x40	64
edx            0x80114844	-2146351036
ebx            0x10074	65652
esp            0x8010d5fc	0x8010d5fc
ebp            0x8010d628	0x8010d628
esi            0x0	0
edi            0x1178c8	1145032
eip            0x80105528	0x80105528
eflags         0x86	[ PF SF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x0	0
gs             0x18	24
(gdb) x/4x 0x8010d5fc
0x8010d5fc:	0x80104adf	0x80114844	0x8dffff9c	0x801052d9
```
可以看到，压入 `swtch` 参数后，其中第一个参数是 `0x80114844`， 第二个参数是 `0x8dffff9c`。执行 `call` 指令后，`esp` 从原来的 `0x8010d600` 变成 `0x8010d5fc`，原因就是压入了 `eip:0x80104adf`，从 `kernel.asm` 可以知道，`80104adf` 就是 `switchkvm()` 函数。

### 保存旧的 context

```bash
(gdb) s
=> 0x8010552c:	mov    0x8(%esp),%edx
11	  movl 8(%esp), %edx
(gdb) s
=> 0x80105530:	push   %ebp
14	  pushl %ebp
(gdb) s
=> 0x80105531:	push   %ebx
15	  pushl %ebx
(gdb) s
=> 0x80105532:	push   %esi
16	  pushl %esi
(gdb) s
=> 0x80105533:	push   %edi
17	  pushl %edi
(gdb) s
=> 0x80105534:	mov    %esp,(%eax)
20	  movl %esp, (%eax)
(gdb) info reg
eax            0x80114844	-2146351036
ecx            0x40	64
edx            0x8dffff9c	-1912602724
ebx            0x10074	65652
esp            0x8010d5ec	0x8010d5ec
ebp            0x8010d628	0x8010d628
esi            0x0	0
edi            0x1178c8	1145032
eip            0x80105534	0x80105534
eflags         0x86	[ PF SF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x0	0
gs             0x18	24
(gdb) x/5x 0x8010d5ec
0x8010d5ec:	0x001178c8	0x00000000	0x00010074	0x8010d628
0x8010d5fc:	0x80104adf
```

`swtch.S` 首先获取两个参数，并将当前的 `context` 压栈，查看寄存器，可以看到，第一个参数值存放在 `eax`，第二个参数存放在 `edx`。这里将当前的 `context: %ebp,%ebx,%esi,%edi` 压入当前内核栈中。

但是这里只保存了四个寄存器，`context` 有5个寄存器。在前面讲过，执行 `call` 后，会把 `eip` 压栈，因为 `swtch.S` 是汇编代码，和 C 代码不一样，不会把 `ebp` 压栈。进入 `swtch.S` 后只 push 4个寄存器，当保存旧 `context` 后

```nasm
movl %esp, (%eax) // cpu->scheduler = 当前esp
```

**`call` 后的 `eip` 就是 `struct context` 的 `eip`**，也就是说 `cpu->scheduler->eip= 0x80104adf`。

```c
struct context {
  uint edi;
  uint esi;
  uint ebx;
  uint ebp;
  uint eip;
};
```
### 载入新的 context
swtch 的原型是

```c
void swtch(struct context **old, struct context *new);
```
其中 `**old` 的值是需要保存的，`*new` 是要新载入的 `context`，汇编代码如下

```bash
# Switch stacks
  movl %esp, (%eax)  // 赋值 old
  movl %edx, %es     // 切换到 new 中
```
在创建进程中，会把进程的 `context` 内容清零，并且把 `context->eip=forkret`。下面调试信息显示新 `context` 内容：

```bash
(gdb) s
=> 0x80105536:	mov    %edx,%esp
21	  movl %edx, %esp
(gdb) s
=> 0x80105538:	pop    %edi
24	  popl %edi
(gdb) info reg
eax            0x80114844	-2146351036
ecx            0x40	64
edx            0x8dffff9c	-1912602724
ebx            0x10074	65652
esp            0x8dffff9c	0x8dffff9c
ebp            0x8010d628	0x8010d628
esi            0x0	0
edi            0x1178c8	1145032
eip            0x80105538	0x80105538
eflags         0x86	[ PF SF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x0	0
gs             0x18	24
(gdb) x/5x 0x8dffff9c
0x8dffff9c:	0x00000000	0x00000000	0x00000000	0x00000000
0x8dffffac:	0x80104bf7
(gdb) s
=> 0x80105539:	pop    %esi
25	  popl %esi
(gdb) s
=> 0x8010553a:	pop    %ebx
26	  popl %ebx
(gdb) s
=> 0x8010553b:	pop    %ebp
27	  popl %ebp
(gdb) s
=> 0x8010553c:	ret
?? () at swtch.S:28
28	  ret
(gdb) info reg
eax            0x80114844	-2146351036
ecx            0x40	64
edx            0x8dffff9c	-1912602724
ebx            0x0	0
esp            0x8dffffac	0x8dffffac
ebp            0x0	0x0
esi            0x0	0
edi            0x0	0
eip            0x8010553c	0x8010553c
eflags         0x86	[ PF SF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x0	0
gs             0x18	24
(gdb) s
=> 0x80104bf7 <forkret>:	push   %ebp
forkret () at proc.c:354
354	{
```

使用栈可以很灵活，任何一块内存都可以当作栈，只需要合理设置 `esp`即可。这里重新设置栈顶寄存器为 `p->context`，那么栈就从新位置开始了。而 `p->context` 的内容已经初始化过了，最重要的 `eip` 设置为 `forkret`，弹出4个寄存器后(全部为0），代码跳转到 `forkret` 中。

### 运行用户代码
代码进入 `forkret` 不是通过函数调用，所以不会有什么参数，或者 `eip` 压栈。在前面的图中可以看到，`context->eip` 之前的 `trapret`，所以 `forkret` 结束之后，调用 `ret` 指令，代码将跳转到 `trapret` 中。

```bash
(gdb) info reg
eax            0x80114844	-2146351036
ecx            0x40	64
edx            0x8dffff9c	-1912602724
ebx            0x0	0
esp            0x8dffffb0	0x8dffffb0
ebp            0x0	0x0
esi            0x0	0
edi            0x0	0
eip            0x80104bf7	0x80104bf7 <forkret>
eflags         0x86	[ PF SF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x0	0
gs             0x18	24
(gdb) x/8x 0x8dffffb0
0x8dffffb0:	0x80106820	0x00000000	0x00000000	0x00000000
0x8dffffc0:	0x00000000	0x00000000	0x00000000	0x00000000
```
可以看到，`trapret` 的地址是 `0x80106820`。而 `trapasm.S:trapret` 的流程，就是系统调用后，从内核返回到用户进程的代码。

```nasm
# Return falls through to trapret...
.globl trapret
trapret:
  popal
  popl %gs
  popl %fs
  popl %es
  popl %ds
  addl $0x8, %esp  # trapno and errcode
  iret
```

因为前面已经设置过 `trap frame`，所以调度器终于完成调度工作，从当前的内核调度器代码，切换到用户的代码当中，并从第一条指令 `eip:0` 开始运行。

```c
//proc.c:userinit(void)

  memset(p->tf, 0, sizeof(*p->tf));
  p->tf->cs = (SEG_UCODE << 3) | DPL_USER;
  p->tf->ds = (SEG_UDATA << 3) | DPL_USER;
  p->tf->es = p->tf->ds;
  p->tf->ss = p->tf->ds;
  p->tf->eflags = FL_IF;
  p->tf->esp = PGSIZE;
  p->tf->eip = 0;  // beginning of initcode.S
```

# 总结
## 进程初始化
每个进程都有自己的用户栈和内核栈，进程状态保存在 `struct proc` 中。创建新进程，主要过程是:

* 分配物理内存（存放程序，进程相关的页表）
* 设置用户进程对应的内核栈，主要是这两个结构体

  * 初始化 `struct context *context;`，最重要是设置 `context->eip`，进程刚开始运行的指令地址
  * 初始化 `struct trapframe *tf;`，保存当前进程的状态。内核调度器切换到用户进程，需要进行特权转变，过程和系统调用返回到用户进程一样。

## 调度器调度
内核调度器不断循环，找到可以运行的进程，分配 `CPU` 资源，运行用户程序。主要过程如下:

* 设置 `CPU` 信息，主要是设置 `TSS` 任务段，切换页表
* 通过 `swtch.S`，保存当前内核栈的信息到 `cpu->scheduler`，切换到进程的 `context` 中。通过结合 `ret` 和堆栈信息，CPU 跳转到进程的 `context->eip`，也就是 `forkret` 代码
* 调用 `forkret` 代码没有通过 `call` 指令，当 `forkret` 结束执行 `ret`，会将当前栈顶元素弹出，当作 `eip`，也就是创建进程设置的 `trapret`
* 代码跳转到 `trapret`，逻辑就和系统调用返回到用户态一样了。

## Tips
如果精心构造栈内容，结合 `ret` 或者 `iret(有特权级转换)`，可以让 `CPU` 跳转到预先准备好的指令上。这样就可以实现系统调用返回到用户进程，或者内核调度线程与用户线程的切换。