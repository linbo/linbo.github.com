---
layout: post
title: "Gnome-Shell 应用图标置顶"
description: ""
category: IT
tags: 
    - ubuntu
---

换了一台电脑，直接装了个Ubuntu 12.04 LTS。嫌原来Ubuntu自带的unity不好用，换成Gnome-Shell 3.4，感觉比unity清爽多了。

Ubuntu下面IM不少，比如pidgin，xchat啥的。QQ还是不愿意在Linux平台露面，所以用了一个pidgin-lwqq插件，是webqq协议的，凑合着用了

问题来了，Gnome-Shell默认情况下，应用图标都在底层托盘里，底层托盘是自动隐藏的，有人ping一下，通知不是很方便。搞了一下，把pidgin，xchat图标都置顶了，这样，别人找我，再也不会半天不回了

* 首先安装Gnome-Shell的一个[icon-manager](https://github.com/MrTheodor/gnome-shell-ext-icon-manager) extension，在项目的install文件里面有写怎么install的了   
* 按照README，打开dconf-editor，找到 org/gnome/shell/extensions/icon-manager/  
* 设置参数如下

```
top-bar: ['pidgin','xchat']
desaturation-factor: 0
```
* 打开 gnome-tweak-tool -> shell extensions -> enable "icon-manager" (不知道这步需不需要，我是做了)  
* Alt-F2, 敲入r，运行

重启xchat (pidgin不用重启)，这下他们可终于跑到顶层panel里面了。
