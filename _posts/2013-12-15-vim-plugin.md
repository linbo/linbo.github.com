---
layout: post
title: "Vim 插件"
description: ""
category: IT
tags: 
    - vim
    - tool
---
{% include JB/setup %}

一直有IDE恐惧症，在大学的时候用Visual Studio，也玩过Eclipse，Netbeans。每次打开这些巨兽时，似乎都要洗身净手，正襟危坐，泡一杯茶，养一会儿神，等IDE启动后，才开始工作。毕业后，甚幸工作的环境是Linux，所以自然而然的接触了Vim。现在想想，估计还是当年我电脑太垃圾了，而那些IDE胃口又太大了，以致我失去了耐心。

喜欢Vim，因为它轻，快，灵活，强大，但是Vim只是一款编辑器。

Vim有很多很强大的插件，武装后，似乎也还有IDE的影子，但是我更喜欢Vim的定位，它只是一款编辑器。

不过呢，工欲善其事，必先利其器。一些好的插件，试试也未尝不可。何不聊聊个人觉得还可以玩玩的插件呢

# 插件管理
回想当年装Vim插件，掩面而泣，说多了都是泪。好汉不提当年泪，但是看了sublime的插件管理，看多了又是泪。唉，都新世纪了，直奔主题，说说现代Vim的插件管理吧

## [pathogen](https://github.com/tpope/vim-pathogen)
pathogen可以安装和管理在私有目录下面的vim插件(亲，还记得当年安装插件，在各个目录拷贝文件吗？)

**安装**  

```bash
$ mkdir -p ~/.vim/autoload ~/.vim/bundle;
$ curl -Sso ~/.vim/autoload/pathogen.vim \
    https://raw.github.com/tpope/vim-pathogen/master/autoload/pathogen.vim

## windows .vim目录改为C:\Users\Administrator\vimfiles
```

**设置**  

```bash
$ grep 'pathogen' ~/.vimrc  # windows 为vim安装目录下面的_vimrc
execute pathogen#infect()
```

**安装插件**  

```bash
$ cd ~/.vim/bundle
$ git clone https://github.com/kien/ctrlp.vim.git
```

**卸载插件**  
直接bundle目录下面的插件目录删除即可了

## [vundle](https://github.com/gmarik/vundle)
这玩意更神了，直接在vimrc文件里配置要安装的插件，就可以对插件进行管理了

**安装**  

```bash
#windows用户还是放在vimfiles下面
$ git clone https://github.com/gmarik/vundle.git ~/.vim/bundle/vundle 
```

**设置**  
可以参考github上面的Readme

**使用**  
运行vim后，输入:BundleInstall就可以安装vimrc上面配置的所有插件了，哈哈，好省事啊

# 其它插件  
下面插件只是介绍了，具体使用和安装，可以参考他们主页说明  

* [emmet](https://github.com/mattn/emmet-vim/)  
这玩意前身叫zen-coding，如果你要写html和css，你怎么可能不用这插件？如果你第一次看emmet视频，准备好目瞪口呆吧。顺便说一下，emmet支持许许多多的编辑器

* [ctrcp.vim](https://github.com/kien/ctrlp.vim)  
用过sublime的人，一定深深的爱着Ctrl+P。其实Vim这款插件，与之相比，也毫不逊色，安装完Ctrl+P看看?

* [nerdtree](https://github.com/scrooloose/nerdtree)  
目录浏览的插件，额，这个真的很方便进行目录跳转

* [vim-multiple-cursors](https://github.com/terryma/vim-multiple-cursors)  
用过sublime的人，多光标操作，爽歪歪。哦，这个就是vim版本的了，还没有研究过呢，惭愧

* [ag](https://github.com/rking/ag.vim)  
grep？ack？都不用了，用ag吧

# 结论  
Vim插件实在是太多了，个人觉得，试用一下，觉得合适就多用用，不适合的，uninstall也费不了多少功夫的

对了，还有好多插件了，等用的感觉不错了，再写续篇吧
