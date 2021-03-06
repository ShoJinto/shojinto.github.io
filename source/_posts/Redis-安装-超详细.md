---
title: Redis 安装 - 超详细
date: 2022-01-25 17:23:32 +0800
categories: [运维, 软件安装]
tags: [运维]     # TAG names should always be lowercase
author:
  name: ShoJinto
  link: https://shojinto.github.io
---

# Redis 安装 - 超详细

作为一名运维软件的安装应该是信手拈来的。尤其是对于redis这样的软件，还是那句话自己忘性太好了。不废话，上操作步骤：

```bash
> wget https://download.redis.io/releases/redis-4.0.14.tar.gz
> tar xf redis-4.0.14.tar.gz
> cd redis-4.0.14
> make 
```

根据官网描述到这里已经安装结束了，但是对于运维怎么可能有没有启动脚本呢。
其实redis的源码中已经包含后续步骤的所有脚本，我们只需要执行即可。好！我们开始。。。

```bash
> ls -l utils/
total 52
-rw-rw-r-- 1 root root  593 Mar 19  2019 build-static-symbols.tcl
-rw-rw-r-- 1 root root 1303 Mar 19  2019 cluster_fail_time.tcl
-rw-rw-r-- 1 root root 1070 Mar 19  2019 corrupt_rdb.c
drwxrwxr-x 2 root root   60 Mar 19  2019 create-cluster
-rwxrwxr-x 1 root root 2137 Mar 19  2019 generate-command-help.rb
drwxrwxr-x 3 root root   31 Mar 19  2019 graphs
drwxrwxr-x 2 root root   39 Mar 19  2019 hashtable
drwxrwxr-x 2 root root   70 Mar 19  2019 hyperloglog
-rwxrwxr-x 1 root root 9567 Mar 19  2019 install_server.sh
drwxrwxr-x 2 root root   63 Mar 19  2019 lru
-rw-rw-r-- 1 root root 1277 Mar 19  2019 redis-copy.rb
-rwxrwxr-x 1 root root 1352 Mar 19  2019 redis_init_script
-rwxrwxr-x 1 root root 1047 Mar 19  2019 redis_init_script.tpl
-rw-rw-r-- 1 root root 1762 Mar 19  2019 redis-sha1.rb
drwxrwxr-x 2 root root  135 Mar 19  2019 releasetools
-rwxrwxr-x 1 root root 3787 Mar 19  2019 speed-regression.tcl
-rwxrwxr-x 1 root root  693 Mar 19  2019 whatisdoing.sh
```

看名字就知道这些脚本是干啥用的，这里安装standard-land模式，直接进行如下操作即可

```bash
> find src -perm 755 -exec cp "{}" /usr/local/sbin/ \;
> utils/install_server.sh
Welcome to the redis service installer
This script will help you easily set up a running redis server

Please select the redis port for this instance: [6379] [Enter键]
Selecting default: 6379
Please select the redis config file name [/etc/redis/6379.conf] [Enter键]
Selected default - /etc/redis/6379.conf
Please select the redis log file name [/var/log/redis_6379.log] [Enter键]
Selected default - /var/log/redis_6379.log
Please select the data directory for this instance [/var/lib/redis/6379] [Enter键]
Selected default - /var/lib/redis/6379
Please select the redis executable path [/usr/local/sbin/redis-server] [Enter键]
Selected config:
Port           : 6379
Config file    : /etc/redis/6379.conf
Log file       : /var/log/redis_6379.log
Data dir       : /var/lib/redis/6379
Executable     : /usr/local/sbin/redis-server
Cli Executable : /usr/local/sbin/redis-cli
Is this ok? Then press ENTER to go on or Ctrl-C to abort
```

支持已经安装完成了，并且redis服务已经启动

```shell
> /etc/init.d/redis_6379 status
Redis is running (10511)
```

到这里就结束了哇。。我不！都2022了怎么可能没有systemd？于是继续。。。

去github上找一份system的模板，最好是redis的systemd模板。根据实际情况进行修改，最终的修改结果如下

```bash
> cat /usr/lib/systemd/system/redis.service
Description=Redis In-Memory Data Store
After=network.target

[Service]
Type=simple
User=redis
Group=redis
Environment=statedir=/run/redis
PermissionsStartOnly=true
PIDFile=/run/redis/redis.pid
ExecStartPre=/bin/touch /var/log/redis.log
ExecStartPre=/bin/chown redis:redis /var/log/redis.log
ExecStartPre=/bin/mkdir -p ${statedir}
ExecStartPre=/bin/chown -R redis:redis ${statedir}
ExecStart=/usr/local/sbin/redis-server /etc/redis/redis.conf
ExecStop=/bin/kill $MAINPID
ExecReload=/bin/kill -USR2 $MAINPID
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

对`install_sercver.sh`脚本生成的配置信息进行修正，使其与`systemd`配置文件中的配置项相符

```bash
> ln -sf /etc/redis/6379.conf /etc/redis/redis.conf
> sed -i 's/redis_6379/redis/g' /etc/redis/redis.conf
> sed -i 's/\(bind \)127.0.0.1/\10.0.0.0/g' /etc/redis/redis.conf
> sed -i 's#\(daemonize \)yes#\1no#g' /etc/redis/redis.conf
> sed -n '/redis_6379/p' /etc/redis/redis.conf
pidfile /var/run/redis_6379.pid
logfile /var/log/redis_6379.log
>  sed -i 's/\(redis\)_6379/\1/g' /etc/redis/redis.conf
> adduser -s /usr/sbin/nologin -d /var/lib/redis -r redis
> chow redis.redis -R /var/lib/redis 

```

重新以systemd启动redis

```bash
> pkill -9 redis-server
> systemctl daemon-reload
> systemctl start redis
> systemctl status redis
● redis.service
   Loaded: loaded (/usr/lib/systemd/system/redis.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-01-26 11:14:33 CST; 7s ago
  Process: 16587 ExecStop=/bin/kill $MAINPID (code=exited, status=0/SUCCESS)
  Process: 16599 ExecStartPre=/bin/chown -R redis:redis ${statedir} (code=exited, status=0/SUCCESS)
  Process: 16595 ExecStartPre=/bin/mkdir -p ${statedir} (code=exited, status=0/SUCCESS)
  Process: 16592 ExecStartPre=/bin/chown redis:redis /var/log/redis.log (code=exited, status=0/SUCCESS)
  Process: 16590 ExecStartPre=/bin/touch /var/log/redis.log (code=exited, status=0/SUCCESS)
 Main PID: 16602 (redis-server)
    Tasks: 4
   Memory: 976.0K
   CGroup: /system.slice/redis.service
           └─16602 /usr/local/sbin/redis-server 0.0.0.0:6379

Jan 26 11:14:33 apollo-yapi-210 systemd[1]: Starting redis.service...
Jan 26 11:14:33 apollo-yapi-210 systemd[1]: Started redis.service.
```

至此redis按照完成。