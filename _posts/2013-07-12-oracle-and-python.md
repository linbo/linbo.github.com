---
layout: post
title: "Oracle与Python初探"
description: ""
category: IT
tags: 
    - 工
    - Python
    - Database
---
{% include JB/setup %}

对Oracle真的是一窍不通啊，不过最近正好有机会，用Python处理Oracle数据，于是记一下做过的事情(Linux版本是Centos 6.3 X64)

# Python的客户端cx_Oracle
要用Python处理Oracle数据库，需要Python的客户端cx_Oracle，这玩意很多人说很难装，奇怪，我怎么一下子就装好了，步骤是这样的

* 安装Oracle的oracle-instantclient，这个需要到[官方网站](http://www.oracle.com/technetwork/database/features/instant-client/index-097480.html)上下载，注意要选择正确的OS版本和Oracle版本，吐槽一下居然要注册才能下载

```bash
sudo rpm -ivh oracle-instantclient11.2-basic-11.2.0.3.0-1.x86_64.rpm
```
* 安装完后，需要设置环境变量，设置的结果如下

```bash
$ cat /etc/profile.d/oracle.sh
export ORACLE_HOME=/usr/lib/oracle/11.2/client64
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME/lib
```

* 然后就可以安装cx_Oracle客户端啦，去[主站](http://cx-oracle.sourceforge.net/)下载正确版本的安装文件，貌似在PyPI上面也有，估计用pip也可以安装的。再次吐槽一下，居然Mysql和Oracle的Python客户端还托管在Sourceforge上面，而且Sourceforge老是被墙，真是不科学啊，怎么也应该在Github或者bitbucket上面嘛

```bash
sudo rpm -ivh cx_Oracle-5.1.2-11g-py26-1.x86_64.rpm
```

* 最后一步还是设置环境变量

```bash
$ cat /etc/ld.so.conf
include ld.so.conf.d/*.conf

/usr/lib/oracle/11.2/client64/lib/
```

重新登录shell，大功告成，可以检查一下成功了没有

```bash
$ python -c 'import cx_Oracle; print cx_Oracle.version'
5.1.2
```

# sqlplus
Oracle的命令行客户端，首先要安装oracle-instant-client，并配置好环境变量，具体步骤看前面的前两步。然后到官方网站下载sqlplus安装文件

```bash
$ sudo rpm -ivh oracle-instantclient11.2-sqlplus-11.2.0.3.0-1.x86_64.rpm 

$ cat ~/.bashrc | grep PATH    #配置环境变量
export PATH=$PATH:/usr/lib/oracle/11.2/client64/bin

$ source ~/.bashrc

$ which sqlplus
/usr/lib/oracle/11.2/client64/bin/sqlplus
```
但是这货，这货太奇葩，居然键盘的方向键完全失灵，这是要多么的仇恨shell才做出的壮举啊。无奈，去找了个[wlrap](http://utopia.knoware.nl/~hlub/rlwrap/)。安装完成后还需要设置一下

```bash
$ cat ~/.bashrc | grep sqlplus
alias sqlplus='rlwrap sqlplus'
```
然后，然后sqlplus总算找得到方向了。

# 编码，编码
Python2比较让人诟病的是编码，字符串居然有两个类来表示，Unicode和str，第一次接触肯定晕，晕乎乎的云里雾里的，这个到时候在细说。其实很简单，登陆Oracle数据库，查找一下Oracle数据库的编码

```bash
SQL> SELECT * FROM nls_database_parameters t ;

找到这三列
NLS_LANGUAGE   NLS_TERRITORY   NLS_CHARACTERSET
AMERICAN       AMERICA         AL32UTF8
```
然后设置环境变量NLS\_LANGE为NLS\_LANGUAGE\_NSL\_TERRITORY.NLS\_CHARACTERSET，这个例子就是 AMERICAN\_AMERICA.AL32UTF8喽

在Python脚本中，也需要先设置这个环境变量，然后才能创建数据库的连接

```python
os.environ['NLS_LANG'] = 'AMERICAN_AMERICA.AL32UTF8'
```
这样的话，Python脚本中处理中文字符就没有任何问题了。

# 查找第n行数据
估计很少人会这样查找数据吧，我也是突发奇想，结果还问了2个高手才搞定，看来不能乱想

```sql
select * from ( select rownum rn, a.* from table_name a  where rownum <=10) where rn =10;
```

没了，暂时这么多估计够用了，不够再写吧

# cx_Oracle如何在cgi里面生效
用apache运行python cgi脚本，发现cx_Oracle无法使用，查了一下，原来环境变量没有在apache环境里生效

```bash
# cat /etc/ld.so.conf.d/oracle.conf 
/usr/lib/oracle/11.2/client64/lib

# lsconfig
```
然后就可以了
