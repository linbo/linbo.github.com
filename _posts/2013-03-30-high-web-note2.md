---
layout: post
title: "[读书笔记]高性能网站建设指南2"
description: ""
category: IT
tags: 
    - web
    - 性能
    - 读书笔记
    - 工
---
{% include JB/setup %}

# Rule9 减少DNS查找
DNS缓存在浏览器和操作系统中，在浏览器找不到，才查找操作系统查找

## 影响DNS缓存的因素
DNS服务器查找返回的DNS记录包含了一个存活时间TTL，告诉客户端可以对该记录缓存多久  
操作系统会考虑TTL值，但浏览器缓存通常忽略TTL，并设置它自己的时间限制  
HTTP协议的Keep-Alive特性可以同时覆盖TTL和浏览器的时间限制，因为浏览器可以和服务器保持TCP连接，无需DDNS查找

**通过使用Keep-Alive和较少的域名来减少DNS查找**

# Rule10 精简Javascript
精简是从代码中移除不必要的字符以减少其大小，进而改善加载时间的实践。精简后，所有的注释以及不必要的空白字符都被移除  
混淆也会移除注释和空白，同时还会改写代码。作为改写一部分，函数和变量名都被转换为更短的字符串  
> 混淆更加复杂，过程本身可能引入错误  
> 混淆会改变Javascript符号，因此需要对任何不能改变的符合(API函数)进行标记，防止混淆器修改它们  
> 混淆后的代码很难阅读，在产品环境中调试更加困难

## tools
[yuicompressor](http://yui.github.com/yuicompressor/)

# Rule11 避免重定向
**重定向会使页面变慢**
##重定向类型
301 Moved Permanently  
302 Moved Temporarily  
304 Not Modified  

    HTTP 1.1 301 Moved Permanently
    Location: http://example.com/newuri
    Content-Type: text/html

浏览器自动跳转到location所指的URL上面，重定向的信息都在响应头上，响应体通常是空的  
不管是301还是302,响应都不会被缓存，除非有附加头，如Expires或者Cache-control  
Javascript的document.location可以执行重定向，不过最好用HTTP的3xx状态码，主要为了确保后退按钮能够正常工作

##缺少结尾的斜线
访问http://example.com/url如果结尾的斜线/没有出现，将产生一个301响应，包含到一个http://example.com/url/的重定向  
主机名缺少/不会发生重定向，比如http://example.com

# Rule12 移除重复脚本
1. 不必要的http请求，不会发生在firefox中，IE中如果包含两次且没有被缓存，会发生两个http请求
2. 对脚本进行多次求值

# Rule13 配置或移除ETag
如果缓存过期(或者用户明确地重新加载了页面，例如单击了reload和refresh按钮)，会产生条件GET请求

服务器检查缓存是否过期有两种方式

1. 比较最新的修改日期
2. 比较实体标签 ETag

实体标签是HTTP1.1引入的，唯一标识一个组件的一个特定版本的字符串，该字符串必须用引号引起来  
If-None-Match比If-Modified-Since具有更高的优先级

    GET /i/yahoo.gif HTTP 1.1
    Host: us.yimg.com

    -----------------------------

    HTTP 1.1 200 OK
    Last-modified: True, 12 Dec 2006 03:03:59 GMT
    ETag: "10c24bc-4ab-457e1c1f"
    Content-Length: 1195

浏览器会使用If-None-Match头将ETag传回原始服务器，如果匹配，则返回304

    GET /i/yahoo.gif HTTP 1.1
    Host: us.yimg.com
    If-Modified-Since: True, 12 Dec 2006 03:03:59 GMT
    If-None-Match: "10c24bc-4ab-457e1c1f"

    -----------------------------------------

    HTTP 1.1 304 Not Modified

## 问题
1. 通常使用组件的某些属性来构造，这些属性对于不同的服务器是不一样的，虽然组件是一样的。比如在Apache1.3和2.x是inode-size-timestamp，inode在不同服务器是不一样的
2. 降低代理缓存的效率，代理后面的用户缓存的ETag经常和代理缓存的ETag不匹配，导致不必要的请求被发送到原始服务器上面

可以通过自定义ETag解决部分问题

# Rule14 使Ajax可缓存

# 掩卷
本书很薄，才100多页，但是内容却很丰富，绝对是前端工程师居家旅行的必备良药。本书出版时间为2008年，但IT技术又那么的日新月异，不可避免的遗漏了一些东西。翻完后，至少有一些些不足

1. 浏览器只讨论了IE和Firefox，但是今天的浏览器不可缺少Chrome，Safari
2. HTTP服务器只讨论了IIS和Apache，今天当然是Nginx的天下了
3. 本书讨论的全球最大的十家网站在今天来说已经不适用了。对于现在最大的网站来说，可能页面的结构已经发生了很大的变化，比如Facebook和Twitter，也许对于这些网站来说，性能优化会有其它新的方向
4. HTML5？移动互联网？

