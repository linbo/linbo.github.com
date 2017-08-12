---
layout: post
title: "Vagrant升级至1.x，无法找到box"
description: ""
category: IT
tags: 
    - Vagrant
    - tool
    - 工
---

Vagrant 1.x出现有一段时间了，看了一下，增加了VMware和AWS(以custom plugin形式提供)的Provider，看来Vagrant功能越来越强大了。

升级了一下，结果出现了box无法载入的问题，根据[这篇文章](http://www.wizonesolutions.com/2013/04/18/fixing-the-box-could-not-be-found-in-the-new-vagrant-1-1/)，手动修改几个文件就可以了。

首先在1.x之前，boxes的目录形式是这样的

```bash
$ ls
box-disk1.vmdk  box.ovf  Vagrantfile
```
到了1.x之后，变成这样了

```bash
$ tree virtualbox/
virtualbox/
├── box-disk1.vmdk
├── box.ovf
├── metadata.json
└── Vagrantfile
```
对比之下，发现1.x 因为增加了不同的Provider，所以在box目录下面新建了不同的Provider目录，各个Provideer目录才存放真正的templates。

除此之外，还多了个metadata.json文件

```bash
$ cat virtualbox/metadata.json  |python -mjson.tool
{
    "provider": "virtualbox"
}
```
于是清楚了变化之后，就好办多了。

* 在box下面新建Provider目录，比如Virtualbox就输入如下命令

```bash
$ mkdir virtualbox
```
* copy文件到Provider目录下面

```bash
$ cp box* Vagrantfile virtualbox
```
* 创建metadata.json文件

好了，现在vagrant up就可以正常工作了。官方文档说1.x 与 1.0.x 兼容，但是忘记以前是什么版本了，如果以前版本的box目录没有Provider目录，那么按上述方法应该就能搞定问题了。

Vagrant真是个好东西，留贴纪念一下
