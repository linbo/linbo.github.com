---
layout: post
title: "Linux神器 - tmux"
description: ""
category: IT
tags: 
    - tool
    - 工
---


#程序员的丰满生活
* 我要查log，好吧开个ssh客户端；我要看代码，好吧我再开一个；我要调代码，我再开一个；我要...

* 一边哼着曲儿，一边敲代码，好不惬意。突然，网络断了，死敲键盘的你，是否有着如同李泰祥大师写的这种心境：

>当我走在凄清的路上  
天空正漂着濛濛细雨  
在这寂寞暗淡的暮色里  
想起我们相别在雨中  
不禁悲从心中生

* 正在用gdb断点调试，女友一通电话，一顿臭骂，一撂狠话:再不回来xxx。一关机所有context都没有了，真是当断不断，欲哭无泪啊！

* 小女子雪地跪求:这个代码调不通，哪位大哥哥可以手把手教我？

作为程序员的救星，*nix怎么可能视而不见，袖手旁观，任尔等悲从心中生呢。于是神器tmux出现了

# tmux部件

* tmux命令启动一个服务
* 一个服务可以有多个session
* 一个session有以多个窗口
* 一个窗口有多个pane

还等什么，赶紧安装后启动tmux吧

# pane

如果用过VIM，命令**sp** / **vsp**一定用的顺风顺水吧，没事就把窗口大卸八块。tmux的pane做类似的事情，试试下面的命令看看?

**ctrl+b+其它键：先按ctrl+b一下，然后再按其它键 ** 

<table border="1"  width="600" >
    <tr>
        <th rowspan="7">ctrl+b</th>
        <th>"</th>
        <th>水平分割</th>
    </tr>
    <tr>
        <th>%</th>
        <th>竖直分割</th>
    </tr>
    <tr>
        <th>x</th>
        <th>关闭面板</th>
    </tr>
    <tr>
        <th>方向键</th>
        <th>选择面板</th>
    </tr>
    <tr>
        <th>ctrl+方向键</th>
        <th>1个单元格调整面板大小</th>
    </tr>
    <tr>
        <th>Alt+方向键</th>
        <th>5个单元格调整面板大小</th>
    </tr>
   <tr>
        <th>q</th>
        <th>显示面板编号</th>
    </tr>
</table>


# 窗口
还记得痛苦的不停开ssh客户端做不同事情吗？干嘛不用窗口呢。所谓窗口就是tab啦，浏览器的tab，VIM的tab，很熟悉吧。赶紧按下面的命令来亲切一下吧

<table border="1"  width="600" >
    <tr>
        <th rowspan="6">ctrl+b</th>
        <th>c</th>
        <th>创建新窗口</th>
    </tr>
    <tr>
        <th>&</th>
        <th>关闭窗口</th>
    </tr>
    <tr>
        <th>,</th>
        <th>改变窗口名字</th>
    </tr>
    <tr>
        <th>数字键</th>
        <th>选择窗口</th>
    </tr>
    <tr>
        <th>w</th>
        <th>按窗口列表选择窗口</th>
    </tr>
    <tr>
        <th>f</th>
        <th>在所有窗口中查找指定文本</th>
    </tr>
</table>

# session
session是一个什么概念了，可以认为是一个工作环境的上下文。官方概念是窗口就是一个伪终端，而session就是一组窗口（伪终端）。

session的好处是  
1.如果客户端掉线了，session仍然保持着  
2.多个客户端可以连到同一个session

如果你还对程序员丰满生活的2、3、4记忆犹新的话，那么session就派上用场了

```bash
tmux new -s session-id  #建立session
tmux ls  #列出session  
tmux attach -t session-d  #attach session
ctrl+b+d  #detach session
```

对问题4来说，小女子开个session，然后告诉大哥哥session-id。大哥哥attach一下session-id，这样，大哥哥的任何操作，小女子都可以看到了，既环保又高效，也就不用雪地求救了

# 配置
tmux好是好，可是那个ctrl+b也太距离产生美了，难道要留长指甲，练习一下九阴白骨爪？放心，既然是神器，怎么不可以配置呢？

配置文件是先读取 ~/.tmux.conf，如果没有找到，然后读取 /etc/tmux.conf。一般自定义的话，都是放在~/.tmux.conf，至于如何配置，就不浪费大家时间了，自行Google吧

reload配置文件可以通过下面命令，进入tmux session后

```bash
1. ctrl+b+: 
2. 输入source-file ~/.tmux.conf
```

# 帮助
如何获取帮助呢？  
1. man tmux  
2. 进入session后， ctrl+b+? 会列出所有命令
