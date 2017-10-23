---
layout: post
title: "在centos上安装可以飘VIP的哨兵模式redis集群"
author: 王一刀
description: ""
category: 
tags: [redis]
---

因为系统迁移，需要在私有云上搭建各种基础服务组件。这两天我搭建了redis集群，搭建过程记录如下：

1.下载

在 http://download.redis.io/releases/ 下载redis-3.0.3.tar.gz 为了和原来系统保持一致，版本仍然用3.0.3

2.编译

将redis-3.0.3.tar.gz 上传到需要安装的机器，解压。
使用root用户 在源码目录 执行make &&　make install

3.为appadm用户配置无密码执行sudo

用root 用户执行：

{% highlight shell %}
echo -e "appadm\tALL=(ALL)\tNOPASSWD:/sbin/ip,NOPASSWD:/sbin/arping" > /etc/sudoers.d/appadm

sed -i "s|Defaults.*requiretty|#Defaults\trequiretty|" /etc/sudoers

chmod 440 /etc/sudoers.d/appadm

{% endhighlight %}

这个是为了在脚本中切换VIP。

4.配置redis.conf

master节点

{% highlight shell %}
daemonize yes
pidfile "/var/run/redis.pid"
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 0
loglevel notice
logfile "/app/allinpal/logs/redis/redis.log"
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename "dump.rdb"
dir "/home/appadm"
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events "gxE"
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-entries 512
list-max-ziplist-value 64
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
maxclients 4064
{% endhighlight %}

slave节点

比master的配置多一句：
{% highlight shell %}
slaveof xx.xx.xx.xx 6379  # master ip 地址
{% endhighlight %}
5.配置sentinel.conf

{% highlight shell %}
port 27000
logfile "/app/allinpal/logs/redis/sentinel.log"
sentinel monitor mymaster xx.xx.xx.xx 6379 2 #master ip 地址
sentinel down-after-milliseconds mymaster 3000
sentinel failover-timeout mymaster 60000
sentinel client-reconfig-script mymaster /app/allinpal/comm/redis-3.0.3/failover.sh
{% endhighlight %}


6.配置failover.sh

{% highlight shell %}
#!/bin/bash
MASTER_IP=${6}
MY_IP='xx.xx.xx.xx'　#根据自身机器ip配置
VIP='xx.xx.xx.xx'　　#虚ip地址
NETMASK='24'
INTERFACE='eth0'  
if [ ${MASTER_IP} = ${MY_IP} ]; then
        sudo /sbin/ip addr add ${VIP}/${NETMASK} dev ${INTERFACE}
        sudo /sbin/arping -q -c 3 -A ${VIP} -I ${INTERFACE}
        exit 0
else
        sudo /sbin/ip addr del ${VIP}/${NETMASK} dev ${INTERFACE}
        exit 0
fi
exit 1
{% endhighlight %}

7.启动

{% highlight shell %}
redis-server    /app/allinpal/comm/redis-3.0.3/redis.conf
redis-sentinel /app/allinpal/comm/redis-3.0.3/sentinel.conf  --sentinel &
{% endhighlight %}

第一次启动时需要手工设置虚ip
{% highlight shell %}
sudo /sbin/ip addr add xx.xx.xx.xx/24 dev eth0
sudo /sbin/arping -q -c 3 -A xx.xx.xx.xx -I eth0
{% endhighlight %}

８.测试

执行　redis-cli -p 27000 
用客户端工具连到redis
打info命令查看信息
{% highlight shell %}
[redis-3.0.3]$ redis-cli -p 27000
127.0.0.1:27000> info
# Server
redis_version:3.0.3
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:6211ab7aaf0da33e
redis_mode:sentinel
os:Linux 2.6.32-431.el6.x86_64 x86_64
arch_bits:64
multiplexing_api:epoll
gcc_version:4.4.7
process_id:1460
run_id:716da5768141ca04b188debf101fc51c831c2cad
tcp_port:27000
uptime_in_seconds:25746306
uptime_in_days:297
hz:10
lru_clock:15586700
config_file:/app/thfd/comm/redis-3.0.3/sentinel.conf

# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
master0:name=mymaster,status=ok,address=10.2.209.33:6379,slaves=2,sentinels=3
127.0.0.1:27000> 
{% endhighlight %}

可以将master的redis进程kill掉　看master节点和VIP有没有切换






