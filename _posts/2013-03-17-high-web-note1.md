---
layout: post
title: "[读书笔记]高性能网站建设指南1"
description: ""
category: IT
tags: 
    - web
    - 性能
    - 读书笔记
    - 工
---
{% include JB/setup %}

[**《高性能网站建设指南》**](http://book.douban.com/subject/3132277/)是前端优化的书，作者针对前端提出了14条优化建议，虽然有点琐碎，但是有立竿见影之效用。如果对前端性能优化有兴趣，不妨看看

#时间花在哪里？
1. 不到20%的时间花在HTML文档的传送上，其余时间花在页面组件上
2. 请求脚本时不会发生并行请求，浏览器在下载脚本时会阻塞额外的HTTP请求

# HTTP概述
## 条件GET请求
如果浏览器在其缓存中保留了组件的一个副本，但并不确定是否有效，就会产生一个条件GET请求

* 请求头 If-Modified-Since将最后修改时间发给服务器
* 服务器响应头带有修改时间 Last-Modified 
* 如果组件自生成日期以来从来没有改变过，服务器返回 "304 Not Modified" 并不再发送响应体

```
GET /index.html  
...  
If-Modified-Since: Wed, 22 Feb 2006 04:11:12 GMT
-----------------------------------------  

HTTP 1.1 304 Not Modified  
Content-Type: type/html  
Last-Modified: Wed, 22 Feb 2006 04:11:12 GMT
```


## Expires
条件请求和304让页面加载更快，但是仍需要一次请求/响应。响应头可以通过明确指出浏览器是否可以使用组件的缓存副本来消除这个需要

* 服务器可以通过Expires响应头，明确指出浏览器是否可以使用组件缓存  
* 浏览器会把expires头信息和相应的过期时间组件一起缓存。只要组件没有过期，浏览器就会使用缓存版本，而不会进行任何HTTP请求

```
Last-Modified: Wed, 22 Feb 2006 04:11:12 GMT  
Expires: Wed, 22 Feb 2010 08:11:12 GMT
```

## Keep-Alive

浏览器可以在一个单独的连接上进行多个请求，浏览器/服务器通过Connection头来指出对Keep-Alive的支持  
浏览器或服务器可以通过发送一个 Connection: close头来关闭连接

    Get /index.html  
    ...  
    Connection: keep-alive
    ------------------------  
    HTTP 1.1 200 OK  
    Connection: keep-alive

# Rule1  减少HTTP请求次数

## 图片地图

允许在一个图片上关联多个URL  

    <img usemap="#map1" src="/imags/tt.tif">
    <map name="map1>
      <area shape="rect" coords="0,0,31,31" href="home.html", title="Home">
      <area shape="rect" coords="36,0,66,31" href="gifts.html", title="Gifts">
    </map>  
缺点： 除了矩形无法定义其它形状，而且图片必须是连续的  
解决： 导航栏或其它超链接中使用多个图片

## CSS Sprites
使用CSS的background-position属性，可以将HTML元素放置到背景图片中期望的位置

缺点： 需要将多个图片合并到一张图片上面  
优点： 图片不需要是连续的  
解决： 页面中为背景、按钮、导航栏、链接中需要提供大量图片

## 内联图片
data: URL模式可以在页面中包含图片但无需任何额外的HTTP请求  

缺点： 
>不受IE7及以下浏览器的支持  
>可能在数据大小上有限制  
>在跨越不同页面时不会被缓存，而使用css并将内联图片作为背景可以缓存

优点： 可以用在任何需要指定URL的地方，包括script和a标签

##合并脚本和样式表
理想情况多个脚本应该合并为一个脚本,多个样式表合并为一个样式表

# Rule2 使用内容发布网站(CDN) (减少响应距离来减少响应时间)
让服务器与用户更近，减少HTTP请求响应时间  
CDN用于发布静态内容，比如图片、脚本、样式表等

# Rule3 添加Expires头 (通过缓存减少响应次数)
首次访问的时间并不是唯一要考虑的  
通过长久的Expires头，可以缓存图片、脚本、样式表等长期不变化的组件  

## 限制
Expires使用一个特定的时间，要求服务器和客户端的时钟严格同步  
过期时间需要经常检查，如果已过期，需要重新配置新的日期  
如果被缓存文件已经更改，但是还没有过期，则浏览器不检查任何更新，会一直使用缓存(通过修改html中组件文件名来解决)

## Cache-control的max-age指令
HTTP1.1 引入  
表示指定组件被缓存多久，以秒为单位  
如果请求离被缓存的时间小于max-age，浏览器就使用缓存的版本  
Expires和max-age同时存在，Expires将会被覆盖

    Expires: Sun, 16 Oct 2016 05:43:02 GMT  
    Cache-Control: max-age=31360000


#Rule4 压缩组件 (减少响应的大小来减少响应时间)
客户端通过HTTP请求头Accetp-Encoding来标识对压缩的支持(gzip压缩要比deflate好)

    Accept-Encoding: gzip, deflate

服务器接收到压缩标识头，就会使用对应的方法压缩响应，通过响应头Content-Encoding通知客户端

    Content-Encoding: gzip

应该压缩HTML，json，xml，样式表，脚本等文本文件，不应该压缩图片，pdf(因为它们已经被压缩了)

服务器需要额外的CPU周期完成压缩，客户端需要对压缩文件进行解压

## 代理缓存
>对某个URL发送到代理的第一个请求不支持gzip的浏览器，代理会转发到Web服务器，则没有被压缩的响应被代理缓存并发给浏览器  
>第二个对URL请求的浏览器支持gzip，代理仍然需要转发Web服务器，被压缩的响应被代理缓存并发给浏览器  
>如果顺序反了，第一个请求支持gzip，第二个请求不支持，则代理可能1将缓存中压缩版本发给后续的浏览器，而不管它们是否支持压缩

解决方法是在响应中添加Vary头，告诉代理根据一个或多个请求头来改变缓存的响应  
由于压缩的决定基于Accept-Encoding请求头的，因此Vary头需要包含Accept-Encodin

    Vary： Accetp-Encoding

这样代理将缓存响应的多个版本，为Accept-Encoding请求头的每个值缓存一份   
对应上面的例子，代理将缓存两份：压缩版本的和未压缩版本的，并根据请求头响应对应的缓存版本

## 边缘情形
原因
>一些old浏览器号称支持压缩，但是有缺陷 - 变态浏览器(IE 5.5 / IE 6.0SP1)  
>浏览器白名单只为已经证实过支持压缩的浏览器提供压缩内容  
>不能和代理共享浏览器白名单配置  

解决：  
通过**Cache-control: Private**为所有浏览器禁用代理缓存，避免了边缘情形的缺陷

# Rule5 将样式表放在顶部
组件是按照它们在文档中出现的顺序下载的  
将样式表放在底部，浏览器会延迟显示任何可视化组件，阻止内容逐步呈现，可能导致白屏出现  
样式表放在顶部所花时间更长，但是因为页面是逐步呈现，所以用户感觉页面显示更快  
## FOUC (Flash of Unstyled Content) 无样式内容的闪烁
如果样式表仍然在加载，构建呈现树是一种浪费，因为所有样式表加载并解析完毕之前无需绘制任何东西。否则，在样式表正确地下载并解析之后，已经呈现的组件要用新的样式重绘，导致FOUC的产生

# Rule6 脚本放在底部
## 并行下载
HTTP1.1规范**建议**浏览器从每个主机名并行地下载两个组件  
如果组件分别放在两个主机名下，可以并行下载4个组件  
脚本是**不能被并行下载的**

>1. 是脚本可能使用document.write来修改页面内容，因此浏览器得等待以确保页面能够恰当地布局  
>2. 保证脚本能够按照正确的顺序执行

# Rule7 避免CSS表达式
# Rule8 使用外部的JavaScript和CSS
理论上内联要比外置快
> 外置需要更多的HTTP请求  
> 外置的JavaScript和CSS可以被缓存，而内联不能
