---
layout: post
title: Redis+Keepalive
date: 2017-11-1 
tag: [Linux,Redis,Keepalive]
---


### 软件环境
Redis 3.2.6

CentOS 6.3

Keepalive 1.2.8

### 1.准备
master:172.16.154.141

slave:172.16.154.144

virtual ip:172.16.154.222

test ip:172.16.154.188


说明：

1.当master与slave均运作正常,master负责服务,slave负责备份。

2.当master挂掉,slave接管服务成为master,有写权限，同时关闭主从复制功能。

3.当master恢复正常,master降级成slave同步数据,开启主从复制,负责备份。


优缺点:

优点:成本低，实现简单

缺点:集群依赖单台Redis处理能力,可以部署多组,扩展性较差

### 2.配置Redis服务
```
# wget http://download.redis.io/releases/redis-3.2.6.tar.gz
# tar -xvzf redis-3.2.6.tar.gz 
# cd redis-3.2.6
# make
# make install
# mkdir /etc/redis
# cp redis.conf /etc/redis/6379.conf
# cp utils/redis_init_script /etc/init.d/redis
# vim /etc/init.d/redis
chkconfig: 2345 90 10
description: Redis is a persistent key-value database
# chmod +x /etc/init.d/redis
# chkconfig redis on

# vim /etc/redis/6379.conf 
daemonize yes
bind 0.0.0.0
port 6379
slaveof 172.16.154.141 6379

# /etc/init.d/redis start
# /etc/init.d/iptables stop
```

### 3.配置Keepalive服务
```
# yum -y install popt popt-devel
# yum -y install openssl-devel

# wget http://www.keepalived.org/software/keepalived-1.2.8.tar.gz
# tar -xvzf keepalived-1.2.8.tar.gz 
# cd keepalived-1.2.8
# ./configure --prefix=/usr/local/keepalived --sysconf=/etc
# make && make install

# cp /usr/local/keepalived/sbin/keepalived /bin/
# chkconfig --add keepalived
# chkconfig keepalived on

# /etc/init.d/keepalived start
Starting keepalived:                                       [  OK  ]

# mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf_bak
```

1.主节点配置
```
# touch /etc/keepalived/keepalived.conf

# vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived

vrrp_script chk_redis {
    script "/home/scripts/redis_check.sh"      ###监控脚本
    interval 2                                 ###监控时间
}

vrrp_instance VI_1 {
    state BACKUP                               ###设置为BACKUP
    nopreempt                                  ###不抢占MASTER
    interface eth0                             ###监控网卡
    virtual_router_id 51
    priority 100                               ###权重值
    authentication {
        auth_type PASS                         ###加密
        auth_pass abcdef                       ###密码
    }

    track_script {
         chk_redis
    }

    virtual_ipaddress {
       172.16.154.222                            ###virtual ip
    }

    notify_master /home/scripts/redis_master.sh
    notify_backup /home/scripts/redis_backup.sh
    notify_fault  /home/scripts/redis_fault.sh
    notify_stop   /home/scripts/redis_stop.sh
}

# vim redis_backup.sh 
#!/bin/bash 

REDISCLI="/usr/local/bin/redis-cli"

LOGFILE="/var/log/keepalived-redis-state.log"

echo "[backup]" >> $LOGFILE
date >> $LOGFILE
echo "Being slave...." >> $LOGFILE 2>&1

echo "Run SLAVEOF cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF 172.16.154.144 6379 >> $LOGFILE  2>&1 

# vim redis_check.sh
#!/bin/bash 

ALIVE=`/usr/local/bin/redis-cli PING`
if [ "$ALIVE" == "PONG" ]; then
  echo $ALIVE 
  exit 0
else
  echo $ALIVE 
  exit 1
fi

# vim redis_fault.sh 
#!/bin/bash 

LOGFILE=/var/log/keepalived-redis-state.log
echo "[fault]" >> $LOGFILE
date >> $LOGFILE

/bin/bash

REDISCLI="/usr/local/bin/redis-cli"

LOGFILE="/var/log/keepalived-redis-state.log"
echo "[master]" >> $LOGFILE
date >> $LOGFILE
echo "Being master...." >> $LOGFILE 2>&1

echo "Run SLAVEOF NO ONE cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF NO ONE >> $LOGFILE 2>&1

# vim redis_stop.sh 
#!/bin/bash 
LOGFILE=/var/log/keepalived-redis-state.log
echo "[stop]" >> $LOGFILE
date >> $LOGFILE

# chmod +x /home/scripts/*.sh
```

2.从节点配置
```
# touch /etc/keepalived/keepalived.conf

# vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived

vrrp_script chk_redis {
    script "/home/scripts/redis_check.sh"      ###监控脚本
    interval 2                                 ###监控时间
}

vrrp_instance VI_1 {
    state BACKUP                               ###设置为BACKUP
    nopreempt                                  ###不抢占MASTER
    interface eth0                             ###监控网卡
    virtual_router_id 51
    priority 10                                ###权重值
    authentication {
        auth_type PASS                         ###加密
        auth_pass abcdef                       ###密码
    }

    track_script {
         chk_redis
    }

    virtual_ipaddress {
       172.16.154.222                            ###virtual ip
    }

    notify_master /home/scripts/redis_master.sh
    notify_backup /home/scripts/redis_backup.sh
    notify_fault  /home/scripts/redis_fault.sh
    notify_stop   /home/scripts/redis_stop.sh
}

# vim redis_backup.sh
#!/bin/bash 

REDISCLI="/usr/local/bin/redis-cli"

LOGFILE="/var/log/keepalived-redis-state.log"

echo "[backup]" >> $LOGFILE
date >> $LOGFILE
echo "Being slave...." >> $LOGFILE 2>&1

echo "Run SLAVEOF cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF 172.16.154.141 6379 >> $LOGFILE  2>&1

# vim redis_check.sh
#!/bin/bash 

ALIVE=`/usr/local/bin/redis-cli PING`
if [ "$ALIVE" == "PONG" ]; then
  echo $ALIVE 
  exit 0
else
  echo $ALIVE 
  exit 1
fi

# vim redis_fault.sh 
#!/bin/bash 

LOGFILE=/var/log/keepalived-redis-state.log
echo "[fault]" >> $LOGFILE
date >> $LOGFILE

/bin/bash

REDISCLI="/usr/local/bin/redis-cli"

LOGFILE="/var/log/keepalived-redis-state.log"
echo "[master]" >> $LOGFILE
date >> $LOGFILE
echo "Being master...." >> $LOGFILE 2>&1

echo "Run SLAVEOF NO ONE cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF NO ONE >> $LOGFILE 2>&1

# vim redis_stop.sh 
#!/bin/bash 
LOGFILE=/var/log/keepalived-redis-state.log
echo "[stop]" >> $LOGFILE
date >> $LOGFILE

# chmod +x /home/scripts/*.sh
```

### 4.验证
```
1.启动主节点Redis
# /etc/init.d/redis start
Starting Redis server...

2.启动从节点Redis
# /etc/init.d/redis start
Starting Redis server...

3.重启动主节点Keepalive
# /etc/init.d/keepalived restart
Stopping keepalived:                                       [  OK  ]
Starting keepalived:                                       [  OK  ]

4.重启动从节点Keepalive
# /etc/init.d/keepalived restart
Stopping keepalived:                                       [  OK  ]
Starting keepalived:                                       [  OK  ]

5.连接vip
# redis-cli -h 172.16.154.222 -p 6379
172.16.154.222:6379> info
# Replication
role:master
connected_slaves:1
slave0:ip=172.16.154.144,port=6379,state=online,offset=253,lag=0

6.插入数据测试
# redis-cli -h 172.16.154.222 -p 6379 SET a 123
OK
# redis-cli -h 172.16.154.222 -p 6379 get a
"123"

7.主节点插入数据、读取数据
# redis-cli -h 172.16.154.141 -p 6379 set master 1
OK
# redis-cli -h 172.16.154.141 -p 6379 get master
"1"

8.从节点插入数据、读取数据
# redis-cli -h 172.16.154.144 -p 6379 set slave 2
(error) READONLY You can't write against a read only slave.
# redis-cli -h 172.16.154.144 -p 6379 get slave
(nil)

9.模拟主节点挂了，从节点接管主节点
# /etc/init.d/redis stop
Stopping ...
Redis stopped
# redis-cli -h 172.16.154.222 -p 6379
172.16.154.222:6379> info
role:master
connected_slaves:0

10.主节点挂了,从节点接管主节点,写入,读取数据
172.16.154.222:6379> keys *
1) "kkk"
2) "a"
3) "master"
4) "b"
172.16.154.222:6379> flushall
OK
172.16.154.222:6379> set 9999 1111
OK
172.16.154.222:6379> get 9999
"1111"
172.16.154.222:6379> keys *
1) "9999"

11.从节点接管主节点,原有主节点启动,变成从节点
172.16.154.222:6379> info 
role:master
connected_slaves:1
slave0:ip=172.16.154.141,port=6379,state=online,offset=15,lag=1
172.16.154.222:6379> set 1 2
OK
172.16.154.222:6379> get 1
"2"
# redis-cli -h 172.16.154.141 -p 6379 set 3 3
(error) READONLY You can't write against a read only slave.
# redis-cli -h 172.16.154.141 -p 6379 get a
(nil)
# redis-cli -h 172.16.154.144 -p 6379 set o o
OK
# redis-cli -h 172.16.154.144 -p 6379 get o
"o"

12.从节点接管主节点挂掉，原有主节点变成从节点继续接管主节点
# redis-cli -h 172.16.154.141 -p 6379
172.16.154.141:6379> keys *
1) "9999"
2) "o"
3) "1"
172.16.154.141:6379> set a b
OK
172.16.154.141:6379> get a
"b"
# redis-cli -h 172.16.154.222 -p 6379 get a
"b"
# redis-cli -h 172.16.154.144 -p 6379 get a
Could not connect to Redis at 172.16.154.144:6379: Connection refused
Could not connect to Redis at 172.16.154.144:6379: Connection refused

13.主节点、从节点全部挂掉,vip挂掉
# redis-cli -h 172.16.154.222 -p 6379
Could not connect to Redis at 172.16.154.222:6379: No route to host
Could not connect to Redis at 172.16.154.222:6379: No route to host
not connected> 

14.从节点先启动,集群恢复
# redis-cli -h 172.16.154.222 -p 6379
172.16.154.222:6379> keys *
1) "9999"
2) "1"
3) "o"
172.16.154.222:6379> set b 2
OK
172.16.154.222:6379> flushall
OK
172.16.154.222:6379> keys *
(empty list or set)

15.主节点启动,原有从节点变成主节点,原有主节点变成从节点
172.16.154.222:6379> info 
role:master
connected_slaves:1
slave0:ip=172.16.154.141,port=6379,state=online,offset=169,lag=1
# redis-cli -h 172.16.154.141 -p 6379 set b 2
(error) READONLY You can't write against a read only slave.
# redis-cli -h 172.16.154.144 -p 6379 set b 2
OK
```
