---
layout: post
title: "Vagrant初探"
description: "Vagrant 介绍"
category: IT
tags: 
    - Vagrant
    - tool
    - 工
    - ruby
---
{% include JB/setup %}

如果你是Virtualbox的用户，那么[Vagrant](http://www.vagrantup.com/ "Vagrant")绝对值得一看。Vagrant是用ruby写的工具，封装了VirtualBox一些操作，通过几条命令，一个配置文件，可以很方便的部署你想要的虚拟机。

**快速入门**

Vagrant是利用Virtualbox的template创建虚拟机的，所以首先需要加载template，Vagrant里叫box。

```bash
$ vagrant box add lucid32 http://files.vagrantup.com/lucid32.box
```

命令结束后，lucid32.box就被下载下来，默认放在~/.vagrant.d/boxes目录下面。这个box是全局的，可以被不同项目使用。

Vagrant是以项目形式组织你的虚拟机群的。首先设置你的项目目录

```bash
$ mkdir vagrant_guide
$ cd vagrant_guide
$ vagrant init
```

执行完命令后ï   发现当前目录多了一个文件：Vagrantfile。这个文件就是虚拟机的配置文件，它用ruby代码来配置虚拟机的各种参数，包括配置多台虚拟机，每台虚拟机使用哪个box，虚拟机的名字、网络等等。

根据具体需求修改配置文件后，通过命令启动虚拟机。如果虚拟机不存在，则导入配置文件里定义好的box，然后创建虚拟机并启动。如果虚拟机已经存在，直接启动。虚拟机的安装文件目录默认是~/VirtualBox VMs，当然可以通过VirtualBox改变VMs存放目录。

```bash
$ vagrant up
```

虚拟机启动  之后，可以通过ssh命令登陆到虚拟机当中

```bash
$ vagrant ssh vmname
```

当然你也可以暂停、恢复、停止、删除虚拟机。与在VirtualBox手工操作，是不是很方便呢？


**Why**

为什么要用Vagrant呢，主要原因是，很多场景我们需要一堆有相应的配置的机器做一些工作。比如测试一些项目，可能就要求与开发的环境一样，包括运行环境，数据库等等。这时候Vagrant就可以很快速、方便的搭建开发测试环境。那Vagrant是怎么做到的呢？

1. 通过配置文件一次性配置多台虚拟机。

2. Vagrant可以和配置管理工具如puppet, chef紧密结合，在配置文件里做相应配置，就可以在VM启动之后，做一些环境的相应部署。

3. Vagrant默认情况下，项目目录是项目里所有虚拟机的共享目录。所以很多情况下，可以把代码放在项目目录下，这样开发/测试环境都共享同一套的代码，这样保证开发/测试的环境尽可能一致。

**资源**

[官网](http://www.vagrantup.com/ "官网")

[Vagrant boxes](http://www.vagrantbox.es/ "Vagrant boxes")

**问题**

当项目启动多台虚拟机时，每台虚拟机的NAT网络的网卡都一样，mac地址和IP一样。通过参数配置config.vm.base_mac，并不能改变eth0的mac地址，在[SO](http://stackoverflow.com/questions/14050795/vagrant-config-vm-base-mac-doesnt-work "SO")上问了也没有人回

有点忘记VirtualBox的NAT网络了，实际运用过程中，项目有两台虚拟机，都额外配置了Bridge网络，有一台Bridge经常有问题，要重启网络才恢复正常，不知道是不是这个原因引起的。
