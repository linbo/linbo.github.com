---
layout: post
title: "Vagrant改变boxes存放路径"
description: ""
category: IT
tags: 
    - Vagrant
    - tool
    - 工
---
{% include JB/setup %}


# box存放在哪里？

[上篇](http://www.linbo.org/blog/2013/01/06/intro-vagrant/)谈到Vagrang的一些基本内容，不过如果磁盘规划不好，而使用的boxes越来越多，悲剧的发现磁盘没空间了。那么Vagrant的boxes存放在哪里的呢？

翻看文档，发现boxes默认是放在~/.vagrant.d/boxes下面的，如果根目录空间不大，很快没空间了。怎么办？

#  修改box存放路径

赶紧SO和Google，还真发现两篇文章([一](http://stackoverflow.com/questions/14733681/vagrant-d-outside-of-the-home-folder)，[二](http://emptysquare.net/blog/moving-virtualbox-and-vagrant-to-an-external-drive/))讲这玩意，就简要说一下步骤好了

1. copy ~/.vagrant.d/下面的目录到新目录  

```bash
cp ~/.vagrant.d/   /path/to/vagrant_home/
```

2. 设置环境变量

```bash
$ grep 'VAGR' ~/.bashrc 
export VAGRANT_HOME='/path/to/vagrant_home'
```

就这样，重新登录shell后，boxes的存放目录就在  /path/to/vagrant_home/boxes 下面了


# 仅仅如此？

这个也太简单了吧，也太神秘了吧，但是为什么呢？ Vagrant是开源的，窥窥源代码去

当我们运行"vagrant box list"的时候，该命令可以列出所有boxes的名字。既然boxes存放地址已经知道了，查看存放路径可以发现，"vagrant box list"仅仅把该路径的文件夹的名字显示出来。可以试试在目录新建一个空文件夹，然后运行"vagrant box list"看看

既然这样，我们搜一下源代码看看

```bash
$ find . -name *.rb |xargs grep "\.vagrant\.d"
./embedded/gems/gems/vagrant-1.0.5/lib/vagrant/environment.rb:    DEFAULT_HOME = "~/.vagrant.d"
    ....
```

哈，第一条就是，赶紧看看啥玩意

```ruby
def setup_home_path
      @home_path = Pathname.new(File.expand_path(@home_path ||
                                             ENV["VAGRANT_HOME"] ||
                                             DEFAULT_HOME))
```

在代码里发现变量DEFAULT_HOME只在一个地方用到，虽然ruby语法不懂，但是看这代码还是很好理解的。这也是为什么我们设置了环境变量VAGRANT\_HOME后，"vagrant box list"命令首先查找VAGRANT_HOME的路径，如果没有找到才查找DEFAULT_HOME的路径。

终于真相大白了，其实也不是很神秘的
