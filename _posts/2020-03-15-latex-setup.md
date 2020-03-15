---
layout: post
title: "LaTeX环境配置"
description: ""
category: 随笔
tags: 
    - 文
---

> *距离上次写LaTeX，已经过去整整6年了*  

一直用`Markdown`写东西，业余时间学[《SQL基础教程》](https://book.douban.com/subject/27055712/)，随手在纸上画的内容想搬到电脑上，`Markdown`有点捉襟见肘。不想用`Office`那一套庞然大物，好像`iPad`上的`Goodnotes`和`Notability`不错，手边没有，所以目光转到`LaTeX`上。

6年前曾经玩过`LaTeX`，写过一点东西，后面荒废了，几天前又重拾起来，体验挺好。受益于各种技术的发展，`LaTeX`的门槛又比6年前降低了很多。

工欲善其事，必先利其器，有一套快捷友好的配套工具，是用户愿意尝试的第一步。记得6年前，在`Linux`机器上折腾好久`texlive`，而且怎么写中文也被虐的不行。现在借助`Docker`，`VS Code`等利器，搭建一套`LaTeX`使用环境，是分分钟的事情。

原来想在Mac上安装一套`MacTex`，想着安装这么一个大件有点抗拒，网上一搜`texlive`居然有`Docker`镜像，配合`VS Code`挺香，于是便折腾了起来。

# Docker环境
首先机器上需要安装`Docker`，然后直接拉镜像，启动完事。为什么用这个镜像？好像下载量最高

```bash
$ docker pull mirisbowring/texlive_ctan_full:2019
$ docker run -dit 51fa8d279d30 #一串数字是image id
```

建议最好能mount目录到容器里面，存放所写的内容。不过当时直接一顿操作了，幸好镜像里面带了`Git`，所以也无所谓了。

拷贝容器内容

```bash
$ docker cp a64a398683e3:/data/tex/demo.pdf ./
```

# VS Code环境
首先要安装`Remote-Containers`和`Latex Workshop`两个扩展。

`VS Code`配置如下（里面很多配置项应该可以删掉，网上拷贝的，没去研究了）


```json
// Latex workshop
  "latex-workshop.latex.tools": [
        {
          "name": "latexmk",
          "command": "latexmk",
          "args": [
          "-synctex=1",
          "-interaction=nonstopmode",
          "-file-line-error",
          "-pdf",
          "%DOC%"
          ]
        },
        {
          "name": "xelatex",
          "command": "xelatex",
          "args": [
          "-synctex=1",
          "-interaction=nonstopmode",
          "-file-line-error",
          "%DOC%"
            ]
        },          
        {
          "name": "pdflatex",
          "command": "pdflatex",
          "args": [
          "-synctex=1",
          "-interaction=nonstopmode",
          "-file-line-error",
          "%DOC%"
          ]
        },
        {
          "name": "bibtex",
          "command": "bibtex",
          "args": [
          "%DOCFILE%"
          ]
        }
      ],
  "latex-workshop.latex.recipes": [
        {
          "name": "xelatex",
          "tools": [
          "xelatex"
                      ]
                },
        {
          "name": "latexmk",
          "tools": [
          "latexmk"
                      ]
        },

        {
          "name": "pdflatex -> bibtex -> pdflatex*2",
          "tools": [
          "pdflatex",
          "bibtex",
          "pdflatex",
          "pdflatex"
                      ]
        }
      ],
  "latex-workshop.view.pdf.viewer": "tab",  
  "latex-workshop.latex.clean.fileTypes": [
      "*.aux",
      "*.bbl",
      "*.blg",
      "*.idx",
      "*.ind",
      "*.lof",
      "*.lot",
      "*.out",
      "*.toc",
      "*.acn",
      "*.acr",
      "*.alg",
      "*.glg",
      "*.glo",
      "*.gls",
      "*.ist",
      "*.fls",
      "*.log",
      "*.fdb_latexmk"
    ]
```

试试通过`VS Code`连接到容器里码`LaTeX`的快乐吧。每次保存`tex`文件后，自动编译成`pdf`，快捷键`option+command+B`编译，`option+command+V`预览pdf文件。

碰到一个代码高亮的问题，`command+K,T`调出选择颜色主题框，把`Dark(Visual studio)`改成`Dark+`，高亮语法出现。