---
layout: post
title: "poolboy的坑"
description: ""
category: IT
tags:
    - erlang
---

[poolboy](https://github.com/devinus/poolboy)是Erlang中运用非常广泛的进程池库，它有很多优点，使用简单，在很多项目中都能看到它的身影。不过，它也有一些坑，使用时候需要注意。（本文对poolboy的分析基于[1.5.1](https://github.com/devinus/poolboy/blob/1.5.1/src/poolboy.erl)版本）

# worker创建不能失败
当[poolboy初始化](https://github.com/devinus/poolboy/blob/1.5.1/src/poolboy.erl#L296)的时候，或者当前进程池的[worker数量超过默认值](https://github.com/devinus/poolboy/blob/1.5.1/src/poolboy.erl#L192)，都会[新建worker](https://github.com/devinus/poolboy/blob/1.5.1/src/poolboy.erl#L274)。我们看一下新建worker的代码：

```erlang  
new_worker(Sup) ->  
    {ok, Pid} = supervisor:start_child(Sup, []),  
    true = link(Pid),  
    Pid.  
```

可以看到，supervisor:start_child的时候是不能失败的，也就是说worker创建如果失败，会导致poolboy这个gen_server挂掉，导致整个进程池崩溃。

这会有什么影响呢，我们看一下用poolboy管理[eredis](https://github.com/wooga/eredis/)的例子，参考其中的一个实现[eredis_pool](https://github.com/hiroeorz/eredis_pool/)。

## 创建worker失败，pool无法启动

看[eredis初始化](https://github.com/wooga/eredis/blob/master/src/eredis_client.erl#L83)代码，如果连接失败，eredis直接退出。如果用eredis_pool的话，当redis没有起来，或者某些其它原因导致eredis初始化失败(只要一次失败)，会导致eredis_pool无法正常启动。

当然，如果redis无法正常工作，eredis_pool是不应该启动成功。但是如果进程池有100个worker，创建成功99个，第100个失败了，结果导致整个进程池退出，似乎有点太严格了。

所以有人对eredis提了个 [issue](https://github.com/wooga/eredis/issues/44)，应该就是针对这个问题的。

## 创建worker失败，pool异常退出

具体可以看 [create_pool](https://github.com/hiroeorz/eredis_pool/blob/master/src/eredis_pool_sup.erl#L52)的代码。

假如现在连接池配置有100个eredis client，当超过100个client时，poolboy会尝试启用overflow，新建eredis client。如果这时候因为某些原因，创建失败，结果也是一样，eredis_pool 异常退出。

观察网络连接，就会发现，这时候已有的redis client全部断链。当poolboy被重新拉起来的话，又会重新尝试建链。

根据上面分析可以看到，poolboy管理的worker有非常严格的规定，worker创建不能失败。如果失败，可能导致进程池无法正常启动，或者正常运行的进程池异常退出。

## 解决方法，加代理进程

在poolboy和进程之间加一个proxy process，proxy创建时不会去尝试建链，只做一些很简单的工作，确保进程初始化可以成功。在进行具体操作时，再去尝试建链。这样可以避免前面的问题，可以看 [epgsql_pool](https://github.com/interline/epgsql_pool/)或者
[phoenix](https://github.com/phoenixframework/phoenix/blob/v0.13.0/lib/phoenix/pubsub/redis.ex)，或者我们自己fork的[eredis_pool](https://github.com/yunba/eredis_pool)。

# proxy代理进程的问题

但是proxy有一个问题，那就是proxy进程里面的client不一定是正常的。看epgsql_pool和phoenix代码可以知道，proxy只保证自己创建的时候不会失败，至于它管理的client是不是正常的，只有在进行具体工作的时候，才可以知道。

这个大部分情况也没有什么问题，当新建worker，如果client连接有问题时，只会影响本次的poolboy调用，但是不会导致进程池崩溃。

当然可以在proxy进程里面加个定时器，定时去检查client的连接情况，如果失败，尝试重新建链。

但是深入代码时，会发现还是有一个坑。看poolboy [checkin](https://github.com/devinus/poolboy/blob/master/src/poolboy.erl#L310)代码。当checkin时候，如果这个时候进程池数量大于默认值，已经启用了overflow，那么它会尝试关闭这个worker，dismiss_worker代码如下：

```erlang
dismiss_worker(Sup, Pid) ->
    true = unlink(Pid),
    supervisor:terminate_child(Sup, Pid).
```

这个会有什么影响呢，我们分析一下这种情况。

poolboy默认配置100个worker，当worker超过100时，会启用overflow数量的worker。比如overflow为20，现在已经110个worker了。如果再次新建的client建链不成功，而同时110个worker已经有11个worker在checkin。这会导致10个worker被关闭，而这个不正常的worker checkin时可能没有被关闭。

换句话说，由于下面原因，导致正常的client被关闭，而不正常的client被保留。

1. worker启动不能失败
2. proxy不了解它管理的client是否正常
3. 当进程启用overflow后，poolboy checkin会关闭worker  

poolboy是一个简单，高效的进程池库，但是它对管理的worker有很严格的限制。例如管理redis client时，启动redis client不能失败，而且需要redis client自己管理链接，重连等等情况。即使采用proxy进程来管理redis client，仍然可能导致正常的redis client被关闭，而不正常的redis client存在pool中。


