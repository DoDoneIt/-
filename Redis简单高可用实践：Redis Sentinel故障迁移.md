redis通过sentinel机制进行故障的主从切换，这个特性在2.8以上版本已经内置支持，配置和使用起来也很简单。但是有一点：发生故障后，主从自动切换，master的ip下线不可用，slave被升级为master，但是我们的client端中怎么获知server端新的master ip呢？java有强大的Jedis、Redisson比较方便。

##### 环境搭建
centos 6.8，安装redis-3.0.5

```
$ wget http://download.redis.io/releases/redis-3.0.5.tar.gz
$ tar xzf redis-3.0.5.tar.gz
$ cd redis-3.0.5
$ make&&make install
```

为了模拟主从，我们新建目录作为slave，复制一份redis.conf进去，并且更改文件中的port端口为6378和slaveof值

![image](https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/redis-slave-1.png)

master和slave、sentinel先后启动，启动时候加载不同的配置即可

1、启动master：redis-server  redis-3.0.5/redis.conf

![image](https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/redis-slave-2.png)

2、启动slave：redis-server redis-slave/redis.conf

![image](https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/redis-slave-3.png)

3、启动sentinel：redis-server redis-3.0.5/sentinel.conf –sentinel

![image](https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/sent启动.png)

4、进入当前master可以查看主从信息：info Replication

![image](https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/redis-slave-4.png)

##### 客户端感知故障迁移

###### 一、客户端主动获取

redis官方提供了php库[PSredis](https://github.com/jamescauwelier/PSRedis)，功能还是比较强大的，实现了多Sentinel 监控，其原理也比较简单：客户端连接Sentinel实例，通过Sentinel提供的命令来获取master：SENTINEL get-master-addr-by-name <master>

![image](https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/redis-slave-6.png)

###### 二、Sentinel推送客户端

在Sentinel的[配置文件](http://download.redis.io/redis-stable/sentinel.conf)中，提供了sentinel client-reconfig-script <master>参数：当主机因故障切换而更改时，可以调用指定脚本来通知客户端地址已经发生了变化。它会传递7个参数给脚本，具体解释如下：

```php
# CLIENTS RECONFIGURATION SCRIPT
#
# sentinel client-reconfig-script <master-name> <script-path>
#
# When the master changed because of a failover a script can be called in
# order to perform application-specific tasks to notify the clients that the
# configuration has changed and the master is at a different address.
# 
# The following arguments are passed to the script:
#
# <master-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>
#
# <state> is currently always "failover"
# <role> is either "leader" or "observer"
# 
# The arguments from-ip, from-port, to-ip, to-port are used to communicate
# the old address of the master and the new address of the elected slave
# (now a master).
#
# This script should be resistant to multiple invocations.
#
# Example:
#
# sentinel client-reconfig-script mymaster /var/redis/reconfig.sh
```

既然脚本可以获取新的master，那么就可以做许多事情了：可以去改client的redis 配置，可以去改写host配置，可以发送通知邮件….我写了一个简单记录日志的shell：

```php
#!/bin/bash
mastername=$1
role=$2
state=$3
formip=$4
fromport=$5
toip=$6
toport=$7
time="`date '+%Y-%m-%d %H:%M:%S'`"
log="${time},redis failover success,from ${formip}:${fromport} , to ${toip}:${toport}"
echo $log >> /var/redis/redisinfo.log
```

需要注意的是，shell脚本需要给可执行权限，不然sentinel启动会报错：

![image](https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/redis-slave-7.png)

##### 结语：
master如果被下线后再次上线加入集群，那么sentinel会把该实例做为slave而不是master，当然也可以强制恢复之前的master-slave关系；sentinel做迁移的过程中会rewrite各个实例的配置文件。

![image](https://raw.githubusercontent.com/DoDoneIt/Develop-blog-img/master/redis-slave-8.png)

redis sentinel内部实现原理和故障迁移原理还有许多知识要学，比如它是怎么实现的监控，怎么做出下线判断，多sentinel之间怎么联系，故障迁移的过程等等。。

参考：

http://download.redis.io/redis-stable/sentinel.conf

http://www.redis.cn/topics/sentinel.html

https://redis.io/topics/sentinel

http://www.cnblogs.com/gomysql/p/5040847.html

http://www.cnblogs.com/ssslinppp/p/5661419.html