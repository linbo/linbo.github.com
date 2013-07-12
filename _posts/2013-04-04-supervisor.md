---
layout: post
title: "进程的守护神 - Supervisor"
description: ""
category: IT
tags: 
    - Python
    - 工
    - tool
---
{% include JB/setup %}

[Supervisor](http://supervisord.org/)是一个Python开发的client/server系统，可以管理和监控***nix**上面的进程。不过同[daemontools](http://linbo.org/blog/2013/02/24/daemontools/)一样，**它也不能监控daemon进程**

# 部件
Supervisor有不同的部件组成，部件分别负责不同的功能，对进程进行监控和管理。

* supervisord  
Supervisor的server部分称为supervisord。主要负责管理子进程，响应客户端的命令，log子进程的输出，创建和处理不同的事件

* supervisorctl  
Supervisor的命令行客户端。它可以与不同的supervisord进程进行通信，获取子进程信息，管理子进程

* Web Server  
Supervisor的web server，用户可以通过web对子进程进行监控，管理等等，作用与supervisorctl一致。

* XML-RPC interface  
XML-RPC接口，提供XML-RPC服务来对子进程进行管理，监控

# 安装
安装supervisor很简单，通过pip就可以安装

```bash
sudo pip install supervisor
```

安装完成之后，就可以用"echo_supervisord_conf"命令来生成配置文件，例如

```bash
echo_supervisord_conf > /etc/supervisord.conf  
echo_supervisord_conf > /path/to/supervisord.conf
```

# 配置
配置文件supervisord.conf是一个ini文件，可以对http_server、supervisord、supervisorctl和program进行配置。不过默认生成的文件已经对大部分进行配置，如果简单使用，只需要配置program的部分就可以了。

配置文件必须要有一个program配置项，这样supervisord才知道哪个program需要被管理和监控。例如下面tornado和nodejs应用的配置

```
[program:tornado_app]
command=python /home/vagrant/tornado/app.py
startsecs=0
stopwaitsecs=0
autostart=true
autorestart=true
stdout_logfile=/home/vagrant/tornado/log/app.log
stderr_logfile=/home/vagrant/tornado/log/app.err

[program:node_app]
command=node /home/vagrant/node/app.js
startsecs=0
stopwaitsecs=0
autostart=true
autorestart=true
stdout_logfile=/home/vagrant/node/log/app.log
stderr_logfile=/home/vagrant/node/log/app.err
```

#启动
配置文件生成之后，就可以启动supervisord了

```bash
supervisord     #默认使用/etc/supervisord.conf的配置文件
supervisord -c /path/to/supervisord.conf
```

当配置文件变化后，可以通过下面的命令reload conf，然后重启supervisord进程

```bash
kill -HUP `cat /tmp/supervisord.pid`   #不过试了一下没有成功
```

#通过supervisordctl管理进程
然后通过supervisorctl就可以监控管理program了

```bash
$ supervisorctl -c conf/app.conf  status
node_app                         RUNNING    pid 6916, uptime 0:00:00
tornado_app                      RUNNING    pid 6917, uptime 0:00:00

$ supervisorctl -c conf/app.conf  stop node_app
node_app: stopped

$ supervisorctl -c conf/app.conf  stop tornado_app
tornado_app: stopped

$ supervisorctl -c conf/app.conf  status
node_app                         STOPPED    Apr 04 02:34 AM
tornado_app                      STOPPED    Apr 04 02:35 AM

$ supervisorctl -c conf/app.conf  start all
node_app: started
tornado_app: started

$ supervisorctl -c conf/app.conf  status
node_app                         RUNNING    pid 8080, uptime 0:00:00
tornado_app                      RUNNING    pid 8079, uptime 0:00:00
```

#通过web管理进程
如果配置文件开启http server，那么就可以通过web界面来管理program了。

```bash
$ grep -A 3 "inet_http_server" conf/app.conf 
[inet_http_server]         ; inet (TCP) server disabled by default
port=0.0.0.0:8383        ; (ip_address:port specifier, *:port for all iface)
#username=user              ; (default is no username (open server))
#password=123               ; (default is no password (open server))
```

然后打开 [http://domain.com:8383](http://domain.com:8383) 就可以访问了

#问题
通过supervisord可以很方便的管理program，可以同时管理多个program，也可以管理一个program的多个进程。而且提供了命令行、web、xml-rpc的接口来管理和监控进程，通过配置文件，可以指定进程挂掉后如何处理(可以重启或者其它方式处理挂掉的进程)

但是，supervisord本身也是一个program，如果它自己挂掉了怎么办?
