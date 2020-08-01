---
title: "Redis 知识点梳理"
catalog: true
date: 2020-7-27 23:51:24
subtitle: “学习笔记”
header-img: "/img/default.png"
tags:
- 后端
catagories:
---

### 什么是Redis

首先还是看一下官方的介绍：Redis 是一款开源的，基于内存的数据结构存储，可用作数据库，缓存和消息代理。支持的数据结构有：字符串（strings），哈希（hashes），列表（lists），集合（sets），有序集合（sorted sets）等，带有范围查询的排序集合，位图，超级日志，带有半径查询和流的地理空间索引。Redis内置了副本，Lua脚本，LRU算法，事务和不同级别的磁盘持久化功能，并通过Redis Sentinel和Redis Cluster自动分区来确保高可用性。

官方的介绍基本上已经把Redis的所有卖点都介绍了，对于后端开发人员而言，Redis基本是最常用的工具之一，而且也是面试必问题目。今天我们就来撸一遍Redis的常见知识点。

### 安装Redis

#### 下载及安装

我直接在[官网](https://redis.io/download)上下载最新稳定版 - 6.0.6, 下载到本地后解压
```shell
$ cd redis-6.0.6
$ make
$ sudo make install
```

#### 修改配置信息

由于redis默认不会以守护进程方式启动，因此我们需要修改配置
```shell
$ vim redis.conf 
```

找到`daemonize no`改为`deamonize yes`。

#### 启动Redis

进入`/usr/local/bin`目录下，先启动Server端，看到这个经典的欢迎界面就说明成功了。
```shell
$ cd /usr/local/bin
$ redis-server

4395:C 27 Jul 2020 23:43:00.640 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
4395:C 27 Jul 2020 23:43:00.640 # Redis version=6.0.6, bits=64, commit=00000000, modified=0, pid=4395, just started
4395:C 27 Jul 2020 23:43:00.640 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
4395:M 27 Jul 2020 23:43:00.641 * Increased maximum number of open files to 10032 (it was originally set to 256).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 6.0.6 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 4395
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

4395:M 27 Jul 2020 23:43:00.642 # Server initialized
4395:M 27 Jul 2020 23:43:00.642 * Ready to accept connections
```
然后再启动客户端:
```shell
$ redis-cli
127.0.0.1:6379> ping
PONG
```

### 使用Redis有哪些好处？

1. 速度快，由于数据存在内存中，而且是基于key哈希值进行存储，因此查询和操作时间复杂度都是O(1)。另外Redis是单线程，不会有线程切换的性能损耗。
2. 相比memcache提供了更加丰富的数据类型支持，如'string','hash', 'list', 'set', 'sorted set'。
3. 可以利用Redis实现分布式锁机制。
4. 丰富的特性，可用于热点数据缓存，计数器，LRU等功能。

### 什么样的场景使用Redis？

1. 热点数据，不经常修改的数据可以使用redis进行缓存处理，减轻数据库查询压力。
2. 分布式锁。
3. SSO Session，Token。
4. 排行榜，计数器
5. 消息队列（Pub/Subs）

### 如何使用Redis实现分布式锁？

实现分布式锁主要利用Jedis中setnx的API。SETNX key value。`SETNX`是`Set if not exists`的缩写，当key已经存在了，则存储不成功，返回0。当key不存在时，操作成功，返回1。

如下是一个schedule Job分布式锁的Java实现:
```java
public boolean isLocked(String locker, int expire) {
    Jedis jedis = null;
    try {
      jedis = jedisPool.getResource();
      if (jedis.setnx(locker, "LOCKED") == 1) {
        jedis.expire(locker, expire);
        return true;
      }
    } catch (Exception e) {
      throw new RuntimeException("Locking error", e);
    } finally {
      if (jedis != null) {
        jedis.close();
      }
    }
    return false;
  }
```

### Redis的过期策略和内存淘汰机制

Redis采用定期删除 + 惰性删除策略。首先要明白定时删除策略是要有定时器来监视所有key，对过期数据及时删除，虽然内存得到及时释放，但十分消耗CPU资源。在大并发请求下，Redis应将CPU多用在处理请求上，而不是去删除Key。因此Redis优化策略，每100ms抽样检查一下过期Key，有则删除。借此，Redis可以一定程度上减轻内存的压力，但肯定还会有很多Key被遗漏掉，这时再使用惰性删除机制来进行清理。也就是说当去获取Key时，Redis先检查Key是否过期，如果过期就会将数据删除。如果定期删除和惰性删除的策略无法及时清理出内存空间，Redis是不是就会爆仓了？不会，因为Redis中有内存淘汰机制。一旦数据爆仓，Redis会优先清除掉最近最少用的key，以此来保证Redis的高命中，高可用。
+ noeviction 当内存不足以容纳新写入数据时，新写入操作会报错。
+ volatile-lru 当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的 Key。这种情况一般是把 Redis 既当缓存，又持久化存储的时候才用。
+ volatile-ttl 当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的 Key 优先移除。
+ volatile-random 当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个 Key。
+ allkeys-lru 当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 Key。
+ allkeys-random 当内存不足以容纳新写入数据时，在键空间中，随机移除某个 Key。

### 缓存的穿透和雪崩问题

就我过去的项目经验还没有遇到过这两种问题，一般流量要几百万左右才可能出现。

**穿透** 即网站遭受到恶意攻击，不断请求数据库中不存在的数据，由于无法击中缓存，因此所有请求都会到达数据库，最终数据库不堪压力而连接异常。
解决方案如下：
+ 利用互斥锁，缓存失效的时候，先去获得锁，得到锁了，再去请求数据库。没得到锁，则休眠一段时间重试。
+ 采用异步更新策略，无论 Key 是否取到值，都直接返回。Value 值中维护一个缓存失效时间，缓存如果过期，异步起一个线程去读数据库，更新缓存。需要做缓存预热(项目启动前，先加载缓存)操作。
+ 提供一个能迅速判断请求是否有效的拦截机制，比如，利用布隆过滤器，内部维护一系列合法有效的 Key。迅速判断出，请求所携带的 Key 是否合法有效。如果不合法，则直接返回。

**雪崩** 即缓存同一时间大面积的失效，这个时候会有大量请求同时到达数据库，导致数据库连接异常。
+ 给缓存的失效时间，加上一个随机值，避免集体失效。
+ 互斥锁，但该方案吞吐量明显下降。

### Redis几种集群模式

+ 单机模式（Standalone）。
+ 主从复制（Master-Slave Replication），这是Redis最普遍和基础的集群模式，其工作原理：Slave从节点服务启动并连接到Master之后，它将主动发送一个SYNC命令。Master服务主节点收到同步命令后将启动后台存盘进程，同时收集所有接收到的用于修改数据集的命令，在后台进程执行完毕后，Master将传送整个数据库文件到Slave，以完成一次完全同步。而Slave从节点服务在接收到数据库文件数据之后将其存盘并加载到内存中。此后，Master主节点继续将所有已经收集到的修改命令，和新的修改命令依次传送给Slaves，Slave将在本次执行这些数据修改命令，从而达到最终的数据同步。
如果Master和Slave之间的链接出现断连现象，Slave可以自动重连Master，但是在连接成功之后，一次完全同步将被自动执行。
+ 哨兵模式（Sentinel）是从Redis的2.6版本开始提供的，但是当时这个版本的模式是不稳定的，直到Redis的2.8版本以后，这个哨兵模式才稳定下来，无论是主从模式，还是哨兵模式，这两个模式都有一个问题，不能水平扩容，并且这两个模式的高可用特性都会受到Master主节点内存的限制。
Sentinel(哨兵)进程是用于监控redis集群中Master主服务器工作的状态，在Master主服务器发生故障的时候，可以实现Master和Slave服务器的切换，保证系统的高可用。
+ Jedis sharding和利用中间件代理，将我们需要存入redis中的数据的key通过一套算法计算得出一个值。然后根据这个值找到对应的redis节点，将这些数据存在这个redis的节点中。但这两种集群方式会牺牲性能。但相对主从复制模式，更易水平拓展。

### Redis数据持久化

在思否上找到了一篇非常好的文章[《Redis持久化》](https://segmentfault.com/a/1190000002906345)介绍Redis两种持久化的方式。

+ 快照（RDB）：在指定的时间间隔保存数据快照，如每天0点。优点：持久化文件简洁，便于数据恢复，启动速度更快。缺点：很难及时备份，数据丢失更多。
+ 追加式文件（AOF）：默认每秒将内存中的数据刷到硬盘里，逐行追加。优点：相比RDB更可靠。缺点：性能会略有降低，AOF文件大小比RDB更大。

