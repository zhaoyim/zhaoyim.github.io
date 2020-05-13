---
layout:     post
title:      "Linux hostname and fqdn"
subtitle:   "Linux"
date:       2020-05-13 12:00:00
author:     "zhaoyim"
header-img: ""
catalog: true
tags:
    - Linux
---

设置Linux的hostname和fqdn，通过设置/etc/hosts, 第一列是IP，第二列是dqdn(主机名+域名)，第三列是主机名，次序明能乱

```
$ vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

10.1.236.197 host-10-1-236-197.zhaoyim.com host-10-1-236-197
```

Linux 命令：

```
[root@host-10-1-236-197 ~]# hostname
host-10-1-236-197
[root@host-10-1-236-197 ~]# hostname -f
host-10-1-236-197.zhaoyim.com
[root@host-10-1-236-197 ~]# dnsdomainname
zhaoyim.com
[root@host-10-1-236-197 ~]#
```

Python:

```
[root@host-10-1-236-197 ~]# python
Python 2.7.5 (default, Aug  4 2017, 00:39:18)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-16)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import socket
>>> print socket.getfqdn()
host-10-1-236-197.zhaoyim.com
>>> print socket.gethostname()
host-10-1-236-197
>>>
```

