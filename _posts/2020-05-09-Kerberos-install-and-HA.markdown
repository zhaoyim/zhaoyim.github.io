---
layout:     post
title:      "Kerberos install and HA"
subtitle:   "Database"
date:       2020-05-09 12:00:00
author:     "zhaoyim"
header-img: ""
catalog: true
tags:
    - Security
---

最近由于客户提出Kerberos的一些问题，为了尝试，自己搭建了Kerberos的主从HA，这里做简单记录。

1. 主节点安装kerberos server
```
$ yum -y install krb5-server krb5-libs krb5-workstation
```

2. 配置kerberos server
```
$ vim /etc/krb5.conf

[logging]
  default = FILE:/var/log/krb5kdc.log
  admin_server = FILE:/var/log/kadmind.log
  kdc = FILE:/var/log/krb5kdc.log

[libdefaults]
  default_realm = ZHAOYIM.COM
  renew_lifetime = 7d
  forwardable = true
  ticket_lifetime = 24h
  dns_lookup_realm = false
  dns_lookup_kdc = false

[realms]
  ZHAOYIM.COM = {
    admin_server = host-10-1-236-197.zhaoyim.com
    kdc = host-10-1-236-197.zhaoyim.com
  }

[domain_realm]
  .zhaoyim.com = ZHAOYIM.COM
  zhaoyim.com = ZHAOYIM.COM
```

3. 配置kdc
 
/var/kerberos/krb5kdc/kadm5.acl
```
$ vim /var/kerberos/krb5kdc/kadm5.acl

*/admin@ZHAOYIM.COM	*
```
/var/kerberos/krb5kdc/kdc.conf
```
$ vim /var/kerberos/krb5kdc/kdc.conf

[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 ZHAOYIM.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```

4. 创建Kerberos数据库
```
[root@host-10-1-236-197 krb5kdc]# kdb5_util create –r ZHAOYIM.COM -s
Loading random data
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'ZHAOYIM.COM',
master key name 'K/M@ZHAOYIM.COM'
You will be prompted for the database Master Password.
It is important that you NOT FORGET this password.
Enter KDC database master key:
Re-enter KDC database master key to verify:
```

5. 启动kdc和kadmin服务
```
$ service krb5kdc start
$ service kadmin start
```

6. 添加管理账号
```
$ kadmin.local
kadmin.local:  addprinc admin/admin@ZHAOYIM.COM
```

7. 检查Kerberos
```
[root@host-10-1-236-197 krb5kdc]# kinit admin/admin@ZHAOYIM.COM
Password for admin/admin@ZHAOYIM.COM:
[root@host-10-1-236-197 krb5kdc]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: admin/admin@ZHAOYIM.COM

Valid starting       Expires              Service principal
2020-05-09T14:41:43  2020-05-10T14:41:43  krbtgt/ZHAOYIM.COM@ZHAOYIM.COM
[root@host-10-1-236-197 krb5kdc]#

```

8. 备节点安装Kerberos
```
$ yum -y install krb5-server krb5-libs krb5-workstation
```

9. 主节点修改/etc/krb5.conf
```
[logging]
  default = FILE:/var/log/krb5kdc.log
  admin_server = FILE:/var/log/kadmind.log
  kdc = FILE:/var/log/krb5kdc.log

[libdefaults]
  default_realm = ZHAOYIM.COM
  renew_lifetime = 7d
  forwardable = true
  ticket_lifetime = 24h
  dns_lookup_realm = false
  dns_lookup_kdc = false

[realms]
  ZHAOYIM.COM = {
    admin_server = host-10-1-236-197.zhaoyim.com
    kdc = host-10-1-236-197.zhaoyim.com
    kdc = host-10-1-236-198.zhaoyim.com

  }

[domain_realm]
  .zhaoyim.com = ZHAOYIM.COM
  zhaoyim.com = ZHAOYIM.COM
```

10. 重启主节点kdc和kadmin服务
```
$ service krb5kdc start
$ service kadmin start
```

11. 创建主从同步账号，并为账号生成keytab文件(默认在/etc/krb5.keytab)
```
$ kadmin.local
kadmin.local:  addprinc -randkey host/host-10-1-236-197.zhaoyim.com
kadmin.local:  addprinc -randkey host/host-10-1-236-198.zhaoyim.com
 
kadmin.local:  ktadd host/host-10-1-236-197.zhaoyim.com
kadmin.local:  ktadd host/host-10-1-236-198.zhaoyim.com
```

12. 复制主节点/etc/krb5.conf和/etc/krb5.keytab到备Kerberos服务器的/etc目录下

13. 复制/var/kerberos/krb5kdc目录下的.k5.CLOUDERA.COM，kadm5.acl和krb5.conf文件到备Kerberos服务器的/var/kerberos/krb5kdc目录

14. 在备节点/var/kerberos/krb5kdc/kpropd.acl配置文件中添加同步账户
```
$ vim /var/kerberos/krb5kdc/kpropd.acl

host/host-10-1-236-197.zhaoyim.com@ZHAOYIM.COM
host/host-10-1-236-198.zhaoyim.com@ZHAOYIM.COM
```

15. 备节点启动kprop服务
```
$ service kprop start
```

16. 主节点使用kdb5_util命令导出Kerberos数据库文件
```
$ kdb5_util dump /var/kerberos/krb5kdc/master.dump
```

17. 主节点同步数据库到备节点
```
$ kprop -f /var/kerberos/krb5kdc/master.dump -d -P 754 host-10-1-236-198.zhaoyim.com
```

18. 备节点启动kdc服务
```
$ service krb5kdc start
```

19. 停止主节点kdc服务并检查kerberos是否可以认证
```
[root@host-10-1-236-197 krb5kdc]# service krb5kdc stop
Redirecting to /bin/systemctl stop krb5kdc.service
[root@host-10-1-236-197 krb5kdc]# kinit admin/admin@ZHAOYIM.COM
Password for admin/admin@ZHAOYIM.COM:
[root@host-10-1-236-197 krb5kdc]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: admin/admin@ZHAOYIM.COM

Valid starting       Expires              Service principal
2020-05-09T14:59:02  2020-05-10T14:59:02  krbtgt/ZHAOYIM.COM@ZHAOYIM.COM
[root@host-10-1-236-197 krb5kdc]#
```

__NOTE:__ 可通过crontab来周期性进行kerberos数据库dump和同步操作，这里不再过多叙述。由于kadmind服务是提供远程管理kerberos账号的，因此此处为单点，这里的主从设置无法提供kadmind服务的HA，如果要解决admind的单点问题可以考虑是用kerberos的LDAP database module，请参考官网 _If the KDC database uses the LDAP database module, kadmin.local can be run on any host which can access the LDAP server._(作者未配置过，仅供有需要时参考)


reference：http://web.mit.edu/kerberos/krb5-latest/doc/admin/install.html