---
layout: post
title: "Node.js如何做Profiling"
description: ""
category: IT
tags: 
    - 工
    - nodejs
    - profiling
---

最近玩一下Node.js，于是乎碰到一个性能问题(悲催，老是碰到性能问题)，研究了一下怎么做Profiling。忙活了好久，终于看到Output了，欢呼雀跃一下

官网博客上面有一篇介绍怎么[Profiling](http://blog.nodejs.org/2012/04/25/profiling-node-js/)，但是用的是DTrace，而且只支持Illumos内核。悲剧的CentOS，只能另寻方法了。

Node.js是运行在V8引擎上面的，[V8 wiki](https://code.google.com/p/v8/wiki/V8Profiler)介绍了如何做V8 Profiling的。找了一下Node.js的，发现这个项目[node-profiler](https://github.com/bnoordhuis/node-profiler)，貌似很容易的样子。

Node.js装的是最新的v0.10.15，用npm装了profiler之后，运行命令

```bash
$node --prof app.js
```
是可以看到v8.log了，但是使用nprof之后，发现输出结果出现很多

```bash
unknown code state: undefined
```
在网上找了半天，最后在Node.js的maillist里发现[node-tick](https://github.com/sidorares/node-tick)作者回的信息
>You can update tickprocessor.js manually from node source matching your version ( its in /deps/v8/tools/tickprocessor.js ). Unfortunately, there is no V8 version information in v8.log. I have same issue in node-tick - see discussion here: https://github.com/sidorares/node-tick/pull/3 
I hope to add target v8 version command line switch some time later.

按照他的[步骤](https://groups.google.com/forum/#!topic/nodejs/g_P97jsCMfw)

```bash
$ pwd
/home/vagrant/node_modules/profiler/tools 
$ mv v8 v8.old
$ cp -r ~/software/node-v0.10.15/deps/v8/tools/ v8
$ ./build-nprof
$ ../nprof  /path/to/v8.log
```
终于没有那个错误了

但是，但是，我看不懂输出结果啊
