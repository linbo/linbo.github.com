---
layout: post
title: "epoll的那些事"
description: ""
category: IT
tags: 
    - Linux
    - Network
---

一直没搞明白 epoll 的机制，以前看不明白 epoll 资料就放弃了。最近重新看这些资料，感觉看明白了大部分。记一下，省的以后又糊涂了。以下内容都是各种资料的小结，以后翻阅省事一点。

# I/O 模型与 epoll
## I/O 流
阻塞模式下，一个线程很难处理多个 I/O 流。比如一个线程要读两个 I/O 事件流，可能 read 第一个I/O 时，因为数据没有就绪，所以整个线程都阻塞了，而第二个 I/O 数据虽然已经就绪，却得不到处理。具体原因个人理解是，线程不知道哪个 I/O 事件已经就绪，只能一个个试。第二个原因是阻塞模式下，如果事件没有就绪，系统调用会阻塞，导致整个线程都阻塞。

非阻塞模式，线程可以通过忙轮询处理多个 I/O 流，同样因为无法知道哪个 I/O 流是否已经就绪，导致很多系统调用都是无效的，效率非常低下。

## I/O 多路复用
如果有一个代理，帮助管理多个 I/O 流，当没有可用的 I/O，线程继续阻塞，I/O 就绪时，唤醒线程处理 I/O，效率会大大提高。在 Linux 平台上，select，poll，epoll 就是这个代理。它们之间具体的优缺点就不讲了，这里只讲 epoll 的机制。

I/O 相关的机制，可以参考知乎上面的讨论 [I/O与epoll](https://www.zhihu.com/question/20122137/answer/14049112)

# epoll 基础
以下内容基本上来自[The method to epoll’s madness](https://medium.com/@copyconstruct/the-method-to-epolls-madness-d9d2d6378642) 和 manpage。

![epoll](https://cdn-images-1.medium.com/max/1600/1*KDk1AVzQJegkcWKJQURYfw.jpeg)

内核内部用数据结构来维护 epoll 的相关信息，epoll 的三个 API 分别操作这些数据结构。
## epoll_create
epoll_create 在内核创建 epoll instance（图中下方褐色方块），返回指向这个 epoll instance 的 file descriptor。
## epoll_ctl
epoll_ctl 可以让 file descriptor 注册到 epoll instance 中，这些 file descriptor 称为 epoll set（图中 INTEREST LIST）。当 epoll set 里面的 file descriptor 有 I/O 就绪情况下，这些 file descriptor 会放到 READY LIST 里面（图中蓝色部分）。

```c
#include <sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
* epfd - epoll_create 返回的 epoll file descriptor
* fd - epoll instance 要监听的 file descriptor
* op - 对 file descriptor 的操作
    * EPOLL_CTL_ADD - 注册 fd 到 epoll instance，fd 成为 epoll set 一员
    * EPOLL_CTL_DEL - 把 fd 从 epoll set 删除，删除后进程无法得到 fd 任何事件的通知。如果 fd 注册到多个epoll instance 中，fd 关闭将导致 fd 从所有 epoll set 中删除
    * EPOLL_CTL_MOD - 修改监听 fd 的事件
* event - 事件信息，具体如下所示

```c
typedef union epoll_data {
    void        *ptr;      
    int          fd;        
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
};
```
其中事件类型通过 uint32_t events 的 bit 表示，epoll_data 一般存放发生事件的 fd。

## epoll_wait
线程调用 epoll_wait 会一直阻塞，直到 epoll set 里面有 fd 的 I/O 就绪。当 epoll_wait 返回后，线程遍历 evlist，处理 READY LIST 里面的就绪的 I/O 事件。

```c
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event *evlist, int maxevents, int timeout);
```

## LT & ET
对于 write 来说，当内核缓冲区非满（包括空和有部分数据数据），LT 模式下 EPOLLOUT 会一直触发，当缓冲区从满到非满，ET 模式下 EPOLLOUT 才会触发。对于 read 来说，当缓冲区非空（包括满和有部分数据），LT 模式下 EPOLLIN 会一直触发，当缓冲区从空到非空，ET 模式下 EPOLLIN 才会触发。默认触发方式是 LT，如果是 ET，在 epoll_ctl 函数里面设置参数 event.events | EPOLLET 。

因为 LT & ET 触发方式不同，处理事件的逻辑也不同。先看 manpage 里面的一个例子

```
1. The file descriptor that represents the read side of a pipe (rfd) is registered on the epoll instance.

2. A pipe writer writes 2 kB of data on the write side of the pipe.

3. A call to epoll_wait(2) is done that will return rfd as a ready file descriptor.

4. The pipe reader reads 1 kB of data from rfd.

5. A call to epoll_wait(2) is done.
```

在 ET 模式下，这种情况可能导致进程一直阻塞。
* 假设 pipe 刚开始是空的，A端发送 2KB，然后等待B端的响应。
* 步骤2完成后，缓冲区从空变成非空，ET 会触发 EPOLLIN 事件
* 步骤3 epoll_wait 正常返回
* B开始读操作，但是只从管道读 1KB 数据
* 步骤5调用 epoll_wait 将一直阻塞。因为 ET 下，缓冲区从空变成非空，才会触发 EPOLLIN 事件，缓冲区从满变成非满，才会触发 EPOLLOUT 事件。而当前情况不满足任何触发条件，所以 epoll_wait 会一直阻塞。

如何解决呢，一个办法就是步骤4一直读，直到数据全部读完，但是在 blocking IO 下会出现另外一个问题，如果某次读完内核缓冲区后，再次调用 read 时，线程将会阻塞。所以需要设置 fd 是非阻塞的，当调用 read 或者 write 时，当返回 EAGIN/EWOULDBLOCK 后才去调用 epoll_wait。

在LT模式下，步骤4结束后，缓冲区还有数据，所以步骤5的 epoll_wait 不会阻塞，因为 EPOLLIN 事件不会丢失，会一直触发。但是也有一个问题，如果一次读的数据太少，将导致多次调用 epoll_wait，所以效率会有所下降。

为了减少 epoll_wait 调用次数，也可以采用ET的模式，使用非阻塞 IO，然后读写直到返回 EAGIN/EWOULDBLOCK。

LT/ET 在非阻塞处理有一点点不同，具体参考网络大神的总结 [epoll LT/ET 深度剖析](https://zhuanlan.zhihu.com/p/21374980)


# reference
* 《Unix环境高级编程》
* [I/O与epoll](https://www.zhihu.com/question/20122137/answer/14049112)
* [epoll LT/ET 深度剖析](https://zhuanlan.zhihu.com/p/21374980)
* [The method to epoll’s madness](https://medium.com/@copyconstruct/the-method-to-epolls-madness-d9d2d6378642)

