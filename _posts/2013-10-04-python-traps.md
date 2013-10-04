---
layout: post
title: "Python 的那些坑"
description: ""
category: IT
tags: 
    - Python
    - 工
---
{% include JB/setup %}

其实各个语言都有坑，用得少了，便远在天边，用得久了，便近在眼前。这不，Python的坑们也纷纷拱手相认，互道珍重了。

# Logging
Python的标准库提供了非常强大的Logging功能，Django于是借了东风，直接拿标准库的来用了。可是以前只是略知皮毛，忘记[RTFM](http://zh.wikipedia.org/zh-cn/RTFM)，结果惨象横生，差点尸体遍野了。

Django+uwsig那是标配(当然还有Gunicorn与之竞争)，所以每个uwsgi起了多个Django进程，粗心大意的用了TimedRotatingFileHandler，然后悲剧就发生了。

当天的日志那是相当的完美，一符一号，一子一句，一标一点都出现在了日志的文件里。可是以前的日志就惨了，只打印了第二天的部分日志，以前的日志全跑到九霄云外去了。

没办法，只好RTFM，终于看到了这么[一段话](http://docs.python.org/2/howto/logging-cookbook.html#logging-to-a-single-file-from-multiple-processes)

> Although logging is thread-safe, and logging to a single file from multiple threads in a single process is supported, **logging to a single file from multiple processes is not supported**, because there is no standard way to serialize access to a single file across multiple processes in Python.

难怪没了，呜呼，只能[另寻方法](http://stackoverflow.com/questions/18840785/timedrotatingfilehandler-doesnt-work-fine-in-django-with-multi-instance)了

# socket
Python写socket客户端那是太简单了，也就那么几行代码吧，可是就是那么几行代码，居然出问题了。

其实原来代码跑的好好的，我没忍住，加了一行代码，结果就出bug了。

客户报有数据丢失，查了日志没啥问题啊，于是和同事联调，发现大包发送时，居然只有部分数据发送过去了。起初没注意，觉得自己没改啥啊，后来同事眼尖，把我加的那行代码去掉了，问题就消失了。

奇了怪了，初始化socket后，我就加了一个socket超时设置啊

```python
s.settimeout(5)
```

RTMF，发现这么[一段话](http://docs.python.org/2/library/socket.html#socket.socket.settimeout)

>Some notes on socket blocking and timeouts: A socket object can be in one of three modes: blocking, non-blocking, or timeout. Sockets are always created in blocking mode. In blocking mode, operations block until complete or the system returns an error (such as connection timed out). In non-blocking mode, operations fail (with an error that is unfortunately system-dependent) if they cannot be completed immediately. In timeout mode, operations fail if they cannot be completed within the timeout specified for the socket or if the system returns an error. The setblocking() method is simply a shorthand for certain settimeout() calls.

>**Timeout mode internally sets the socket in non-blocking mode**. The blocking and timeout modes are shared between file descriptors and socket objects that refer to the same network endpoint. 

看到没，设置了超时后，居然就是non-blocking模式了，真是活见鬼了。只能在socket连接成功后，把Timeout给去掉了

这就是一行代码引发的惨案！

