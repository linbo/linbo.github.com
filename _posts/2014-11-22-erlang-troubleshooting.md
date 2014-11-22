---
layout: post
title: "Erlang 内存泄漏分析"
description: ""
category: IT
tags: 
    - erlang
---
{% include JB/setup %}

随着项目越来越依赖Erlang，碰到的问题也随之增加。前段时间线上系统碰到内存高消耗问题，记录一下troubleshooting的分析过程。线上系统用的是Erlang R16B02版本。

# 问题描述
有几台线上系统，运行一段时间，内存飙升。系统模型很简单，有网络连接，pool中找新的process进行处理。top命令观察，发现内存都被Erlang进程给吃完了，netstat命令查看网络连接数，才区区几K。问题应该是Erlang内存泄漏了。

# 分析方法
Erlang系统有个好处，可以直接进入线上系统，在生产现场分析问题。我们系统是通过[Rebar](https://github.com/rebar/rebar)管理的，可以用不同方法进入线上系统。

## 本机登录
可以直接登录到线上机器，然后通过以下命令attach到Erlang系统里面

```shell
$ cd /path/to/project
$ rel/xxx/bin/xxx attach
(node@host)> 
```
## 通过remote shell

**获取Erlang系统的cookie**

```shell
$ ps -ef |grep beam  %%找到参数 --setcookie
```

**新开一个shell，使用同样的cookie，不同的nodename**

```shell
$ erl --setcookie cookiename -name test@127.0.0.1
```

**用start remote shell进入系统**

```erlang
Erlang R16B02 (erts-5.10.3) [source] [64-bit] [smp:2:2] [async-threads:10] [hipe] [kernel-poll:false]

Eshell V5.10.3  (abort with ^G)
(test1@127.0.0.1)1> net_adm:ping('node@127.0.0.1').
pong
(test1@127.0.0.1)2> nodes().
['node@127.0.0.1']
(test1@127.0.0.1)3> 
User switch command
 --> h
  c [nn]            - connect to job
  i [nn]            - interrupt job
  k [nn]            - kill job
  j                 - list all jobs
  s [shell]         - start local shell
  r [node [shell]]  - start remote shell
  q                 - quit erlang
  ? | h             - this message
 --> r 'node@127.0.0.1'
 --> j
   1  {shell,start,[init]}
   2* {'node@127.0.0.1',shell,start,[]}
 --> c 2
```

# 分析流程
Erlang有很多工具，可以分析系统信息，比如[appmon](http://www.erlang.org/documentation/doc-5.6.1/pdf/appmon-2.1.9.pdf)，[webtool](http://erlang.org/doc/man/webtool.html)。但是系统内存严重不足，已经没有办法启动这些工具了，幸好还有Erlang shell。

Erlang shell自带了很多有用的[命令](http://www.erlang.org/doc/man/shell.html)，可以用help()方法查看

```erlang
> help().
```

## Erlang系统内存消耗情况

top结果显示，是内存问题，所以第一步可以先看看Erlang的系统内存消耗情况

```erlang
> erlang:memory().
```

[memory()](http://www.erlang.org/doc/man/erlang.html#memory-0)可以看到Erlang emulator分配的内存，有总的内存，atom消耗的内存，process消耗的内存等等。

## Erlang process创建数量
线上系统发现主要内存消耗都在process上面，接下来要分析，是process内存泄漏了，还是process创建数量太多导致。

```erlang
> erlang:system_info(process_limit).  %%查看系统最多能创建多少process
> erlang:system_info(process_count).  %%当前系统创建了多少process
```
[system_info()](http://www.erlang.org/doc/man/erlang.html#system_info-1)返回当前系统的一些信息，比如系统process，port的数量。执行上面命令，大吃一惊，只有2，3k的网络连接，结果Erlang process已经有10多w了。系统process创建了，但是因为代码或者其它原因，堆积没有释放。

## 查看单个process的信息
既然是因为process因为某种原因堆积了，只能从process里找原因了

先要获取堆积process的pid

```erlang
> i().  %%返回system信息
> i(0,61,886).  %% (0,61,886)是pid
```
看到有很多process hang在那里，查看具体pid信息，发现message_queue有几条消息没有被处理。下面就用到强大的[erlang:process_info()](http://erlang.org/doc/man/erlang.html#process_info-2)方法，它可以获取进程相当丰富的信息。

```erlang
> erlang:process_info(pid(0,61,886), current_stacktrace).
> rp(erlang:process_info(pid(0,61,886), backtrace)).
```
查看进程的backtrace时，发现下面的信息

```shell
0x00007fbd6f18dbf8 Return addr 0x00007fbff201aa00 (gen_event:rpc/2 + 96)
y(0)     #Ref<0.0.2014.142287>
y(1)     infinity
y(2)     {sync_notify,{log,{lager_msg,[], ..........}}
y(3)     <0.61.886>
y(4)     <0.89.0>
y(5)     []
```

process在处理Erlang第三方的日志库[lager](https://github.com/basho/lager)时，hang住了。

# 问题原因
查看lager的文档，发现以下信息

> Prior to lager 2.0, the gen_event at the core of lager operated purely in synchronous mode. Asynchronous mode is faster, but has no protection against message queue overload. In lager 2.0, the gen_event takes a hybrid approach. it polls its own mailbox size and toggles the messaging between synchronous and asynchronous depending on mailbox size.

> {async_threshold, 20},
> {async_threshold_window, 5}

> This will use async messaging until the mailbox exceeds 20 messages, at which point synchronous messaging will be used, and switch back to asynchronous, when size reduces to 20 - 5 = 15.

> If you wish to disable this behaviour, simply set it to 'undefined'. It defaults to a low number to prevent the mailbox growing rapidly beyond the limit and causing problems. In general, lager should process messages as fast as they come in, so getting 20 behind should be relatively exceptional anyway.

原来lager有个配置项，配置message未处理的数量，如果message堆积数超出，则会用 **同步** 方式处理！

当前系统打开了debug log，洪水般的log把系统给冲垮了。

老外也碰到类似问题，这个[thread](https://groups.google.com/forum/#!searchin/erlang-programming/waiting$20handle_info$20timeout/erlang-programming/JL8HVBjnWy0/nEoBDIhhMFUJ)给我们的分析带来很多帮助，感谢一下。

# 总结
Erlang提供了丰富的工具，可以在线进入系统，现场分析问题，这个非常有助于高效、快速的定位问题。同时，强大的Erlang OTP让系统有更稳定的保证。我们还会继续挖掘Erlang，期待有更多的实践分享。
