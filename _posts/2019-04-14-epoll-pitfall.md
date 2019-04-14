---
layout: post
title: "epoll的那些坑"
description: ""
category: IT
tags: 
    - Linux
    - Network
---


可能当初 epoll 设计没有考虑太多并发的情况，单进程单线程下 epoll 工作良好，但是多进程或者多线程下就有一些坑。

# 文件对象
在讲这个之前，需要了解和文件有关的三个数据结构，具体可以参考《Unix环境高级编程》3.10节，或者[The method to epoll’s madness](https://medium.com/@copyconstruct/the-method-to-epolls-madness-d9d2d6378642)的内容。文件涉及三个数据结构：

* 进程维护 file descriptor 表，每个 fd 包含
    * fd 标志
    * 指向内核 file description 表项的指针
* 内核维护所有打开文件的 file description 表，每个 file description 包含
    * 当前文件的 offset
    * 文件状态标志（读、写、阻塞、非阻塞等）
    * 指向该文件 v 节点表项的指针
* 每个打开文件/设备都有一个 v 节点结构，包含 inode 信息等

# epoll 相关的数据结构
![epoll_fd](https://cdn-images-1.medium.com/max/1600/1*ObWegZ_IDTqGVH2KLYxPSA.png)

调用 epoll_create 成功，返回 epoll instance fd，进程通过 efd 对 epoll 进行操作。同时在内核也会创建 epoll instance 对应的 file description，在文件系统也会创建 inode 信息。

例如图中，fd9 和相关联的其它数据结构，就是 epoll instance 相关的数据结构（淡黄色部分）。

同时注册 fd 到 epoll instance，epoll set 其实监听的是 file description，而不是 file descriptor(fd)。所以 epoll set 里面的 fd0 其实是 fd0 指向的 file description（橙色部分）。

# 复制 epoll fd 问题

![epoll_copy](https://cdn-images-1.medium.com/max/1600/1*oYdvrj-gPPkycZdTFqb3fA.png)

假设 Process A 创建了 epoll，并将 fd0 注册到 epoll 中。Process A fork 子进程 B，此时 B 也拥有和 A 同样的 fd table，B  的 epoll file description 和 A 是同一个。

现在 A 创建了一个新的 fd8 并注册到 epoll 中，如果 fd8 有事件，不仅仅 A 能接收到这个事件，B 也能（虽然 B 都不知道有这个 fd8）。

# 复制 fd 问题
继续上面的例子，假设 A close(fd0)，A 以为自己已经关闭了 fd0，不会收到 fd0 任何事件了。但是由于如下原因

* epoll 监听的是 file description
* 只有指向 file description 的 file descriptor 都关闭，file description 才会删除
* 虽然 A 关闭 fd0，但是file description 还有 B 的 fd0 指着，所以不会删除

所以 A 还是会继续收到 fd0 的事件。由此可以看出，epoll 注册对象的生命周期和对象对应的 fd 生命周期不完全一致。

再比如，epoll 监听了 fd，程序执行 fd2 = dup(fd)，然后调用 close(fd)，会出现如下问题

* 程序还是能接收到 fd 的事件
* fd 不能从 epoll 里面删除，即使做如下 epoll_ctl 操作也不行

```c
epoll_ctl(efpd, EPOLL_CTL_DEL, rfd)
epoll_ctl(efpd, EPOLL_CTL_DEL, rfd2)
```

**所以在 close 之前，一定要记得先把 fd 从 epoll set 里面删除。**

上面两个问题，具体可以参考 [The method to epoll’s madness](https://medium.com/@copyconstruct/the-method-to-epolls-madness-d9d2d6378642)

# load balance问题

[Epoll is fundamentally broken 1/2](https://idea.popcount.org/2017-03-20-epoll-is-fundamentally-broken-12/)讨论了负载均衡的问题

## scale out accept()
如果多个worker thread共享epoll fd，将会存在各种问题

### LT模式
LT将会发生惊群效应，示例如下

```
1. Kernel：接收新连接
2. Kernel：通知A线程处理，假如处理不及时，因为连接还在内核缓冲区中，所以也会通知B线程处理
3. Thread A：完成 epoll_wait()
4. Thread B：完成 epoll_wait()
5. Thread A：accept() 返回成功
6. Thread B：accept() 返回 EAGAIN
```

线程B在这种场景下根本没有必要被唤醒

### ET模式

```
1. Kernel: 接收第一个连接，两个线程在等待，此时只有一个被唤醒，例如thread A.
2. Thread A: 完成 epoll_wait().
3. Thread A: accept()返回成功.
4. Kernel: 因为内核缓冲区为空，ET下新事件产生.
5. Kernel: 接收第二个连接.
6. Kernel: 当前只有线程B在等待，线程B被唤醒.
7. Thread A: 循环调用 accept()直到返回 EAGAIN, 结果却返回新的socket.
8. Thread B: 调用 accept()，期待是新socket，结果返回 EAGAIN.
9. Thread A: 循环调用 accept()直到返回EAGAIN
```

这种情况下，线程B没必要被唤醒。

下面例子中，线程可能会被饿死

```
1. Kernel: 同时接收两个连接，当前线程A and B在等待，ET下只有A被唤醒
2. Thread A: 完成 epoll_wait().
3. Thread A: 调用 accept()成功
4. Kernel: 接收第三个连接，因为内核缓冲区一直有连接，没有新事件产生
5. Thread A: 循环调用accept()，没有返回 EAGAIN
6. Kernel: 接收第四个连接
7. Thread A: 循环调用 accept()，没有返回EAGAIN
```

永远只有A在工作，负载均衡没有生效

### 解决

LT模式，Kernel 4.5+ 用 EPOLLEXCLUSIVE flag 可以保证一个事件只有一个线程被唤醒。

ET模式下，可以使用EPOLLONESHOT flag，但是每个事件要多调用一次epoll_ctl。这种方式下负载可以分到不同CPU中，但是同一时刻最多只能有一个线程在accept，限制了吞吐。

```
1. Kernel: 接收两个连接，ET下两个等待线程只有A被唤醒
2. Thread A: 完成 epoll_wait()
3. Thread A: 调用 accept() 成功
4. Thread A: 调用 epoll_ctl(EPOLL_CTL_MOD), 将会重置 EPOLLONESHOT，re-arm the socket
```

## scale out read()
### LT

防止惊群效应，采用EPOLLEXCLUSIVE flag

```
1. Kernel: 接收2047字节数据
2. Kernel: 两个等待的线程，因为采用EPOLLEXCLUSIVE，只唤醒A.
3. Thread A: 完成 epoll_wait()
4. Kernel: 接收2字节数据
5. Kernel: 只有一个线程B等待，唤醒B
6. Thread A: 调用 read(2048)，读取缓冲区2048字节数据
7. Thread B: 调用 read(2048)，读取1字节数据
```
数据分散到两个线程中，而且没有加锁保护，数据可能乱序

### ET

```
1. Kernel: 接收2048字节数据
2. Kernel: ET下两个等待线程，只有A被唤醒
3. Thread A: 完成 epoll_wait()
4. Thread A: 调用read(2048)，接收2048字节数据
5. Kernel: 内核缓冲区为空，事件触发
6. Kernel: 接收1字节
7. Kernel: 只有1个线程在等待，B被唤醒
8. Thread B: 完成 epoll_wait()
9. Thread B: 调用 read(2048)，接收1字节
10. Thread A: 重新调用 read(2048), 返回 EAGAIN
```

### 解决
解决方法是设置 EPOLLONESHOT flag，每次读完后调用epoll_ctl(EPOLL_CTL_MOD),重新触发事件

# reference
* [The method to epoll’s madness](https://medium.com/@copyconstruct/the-method-to-epolls-madness-d9d2d6378642)
* [Epoll is fundamentally broken 1/2](https://idea.popcount.org/2017-03-20-epoll-is-fundamentally-broken-12/)

