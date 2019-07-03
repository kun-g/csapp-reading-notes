Redis介绍

# Redis是什么
全称是 **Re**mote **D**ictionary **S**erver
也就是基于数据结构的远程字典服务器

# 提供的数据结构
## 基础
- [keys](https://redis.io/commands/keys)可以列出满足正则表达式匹配的所有键，不过注意生产环境千万别随便用这个命令，视数据量会导致一定时间的卡顿。
- [dbsize](https://redis.io/commands/dbsize)获取当前db的键的数量，也可以使用info keyspace看所有的db的键数量
- [exists](https://redis.io/commands/exists)检查某个键是否存在，存在则返回1，否则返回0
- [del](https://redis.io/commands/del)删除一个键，无论什么数据结构类型，都可以使用这个命令删除
### 优秀的数据类型设计
[type](https://redis.io/commands/type)返回一个键的类型，如果键不存在，则返回none
Redis数据结构内部有不同的编码，甚至不同的数据结构会使用相同的编码，这样的设计好处有两点：
1. 弱化外部数据结构和内部编码的耦合性
2. 可以对每一个数据类型在不同的场景下进行针对性的优化
![](DraggedImage.png "Redis数据结构和内部编码")
### 为什么快
1. 纯内存访问
2. 非阻塞I/O，使用epoll作为I/O多路复用技术的实现，Redis将epoll中的连接、读写、关闭都转换为时间，不在网络I/O上浪费过多时间。
3. 单线程避免了线程切换和竞态产生的消耗

### info指令
info命令的使用方法有以下三种：
- info: 部分Redis系统状态统计信息。
- info all: 全部Redis系统状态统计信息。
- info section: 某一块的系统状态统计信息，其中section可以忽略大小写。
section列表：
- server - 服务器信息
- clients - 客户端信息
- memory - 内存信息
- persistence - 持久化信息
- stats - 全局统计信息
- replication - 复制信息
- cpu - cpu消耗信息
- commandstats - 命令统计信息
- cluster - 集群信息
- keyspace - 数据库键统计信息

### 配置文件
配置文件有详细的注释，没必要另外翻译一遍

## 字符串 string
键都是字符串类型
字符串类型可以存放字符串、数字、二进制，数据最大不能超过512MB
### 命令
#### 设置与读取
[set](https://redis.io/commands/set)可以设置字符串类型，同时可以设定过期时间，还可以用nx/xx标志更新数据。[setxx](https://redis.io/commands/setxx)和[setnx](https://redis.io/commands/setnx)就是带xn/xx的快捷版命令，使用这两个命令，可以实现CAS，[分布式锁](http://redis.io/topics/distlock)是其中一种应用。
[get](https://redis.io/commands/get)获取键对应的值，如果键的值不存在，则返回nil
[mset](https://redis.io/commands/mset)与[mget](https://redis.io/commands/mget)可以进行批量操作，能有效提高通讯效率
[getset](https://redis.io/commands/getset)设置值并返回原来的值
#### 计数
[incr](https://redis.io/commands/incr)用于对值做自增操作，当原来的值是整数时会自增1，当值不存在时，会返回1，其它情况下会失败。还有一系列decr、incrby、decrby、incrbyfloat
#### 其它
[append](https://redis.io/commands/append)向字符串尾部追加字符串
[strlen](https://redis.io/commands/strlen)获取字符串长度
[setrange](https://redis.io/commands/setrange)设置指定位置的字符
[getrange](https://redis.io/commands/getrange)获取某个范围的字符串

### 内部编码
-  int： 8个字节的长整型
- raw：大于39个字节的字符串
- embstr：小于等于39个字节的字符串
使用object encoding命令，可以查看编码方式

### 典型使用场景
#### 缓存
```js
UserInfo getUserInfo(long id) {
    userRedisKey = "user:info:" + id
    value = redis.get(userRedisKey);
    UserInfo userInfo;
    if (value != null) {
         userInfo = deserialize(value);
     } else {
         userInfo = mysql.get(id);
         if (userInfo != null)             redis.setex(userRedisKey, 3600, serialize(userInfo));
    }
    return userInfo; 
}
```

内存使用达到maxmemory上限时，会触发内存溢出控制策略。

Redis的过期对象删除策略：
- 惰性删除，当客户端读取已经超时的键时，会执行删除并返回空
- 定时删除，按照一定的逻辑，定时删除过期键
	- 每个数据库空间随机抽查20个键
		- 是否超过25%的键过期？
		- 否-\>退出
		- 是 - 循环清除，直到超时或者比例下降到25%以下

内存溢出控制策略(maxmemory-policy)：
- noeviction：不会删除任何数据
- volatile-lru：LRU算法，会回收设置了超时属性的键
- allkeys-lru：LRU算法，无差别回收键
- allkeys-random：随机删除键
- volatile-random：随机删除有超时属性的键
- valatile-ttl：删除最近要过期的键

长时间未被访问的键：scan + object idletime

#### 计数器
```js
long incrVideoCounter(long id) {
     key = "video:playCount:" + id;
     return redis.incr(key);
}
```

#### 共享 Session
```js
phoneNum = "138xxxxxxxx";
key = "shortMsg:limit:" + phoneNum;
 // SET key value EX 60 NX
 isExists = redis.set(key,1,"EX 60","NX");
 if(isExists != null || redis.incr(key) <=5){
     // 通过
}else{
     // 限速
}
```

### Bitmaps
Bitmap是基于string实现的，可以把Bitmap想象成一个以位为单位的数组，数组的下标在Bitmap里叫偏移量

#### 命令
- setbit
- getbit
- bitcount
- bitop
	- and
	- or
	- not
	- xor
- bitpos

### HyperLogLog
HyperLogLog是一种基数算法，可以用很少的内存完成独立总数统计。

命令：
- pfadd
- pfcount
- pfmerge

HyperLogLog算法存在误差，Redis给出的数据是0.81%的误差

## 哈希 hash
### 命令
- hset
- hmset
- hget
- hmget
- hdel
- hlen
- hexists
- hkeys
- hvals
- hscan
- hincrby
- hincrybyfloat
- hstrlen
### 内部编码
#### hashtable
Hashtable的读写时间复杂度都是O(1)
同时满足两个条件时，使用这种编码：
- 当元素个数小于hash-max-ziplist-entries配置（默认512个）
- 所有值都小于hash-max-ziplist-value配置（默认64字节）
#### ziplist
结构更加紧凑，比hashtable方式更佳节约内存

### 使用场景
需要存的数据如果是字典类型，刚好可以使用hash来存储

## 列表 list
### 简介
List可以用来存储多个有序的字符串，列表长度最多为2^32-1。
List的两个特点：
- 有序。所以可以随机读取、修改
- 可重复。这一点和集合是有区别的
### 命令
#### 添加
- lpush
- rpush
- linsert
#### 查
- lrange
- lindex
- llen
#### 删除
- lpop
- rpop
- ltrim
#### 修改
- lset
#### 阻塞
- blpop
- blpush
### 内部编码
#### ziplist
同时满足两个条件时，使用这种编码：
- 当元素个数小于list-max-ziplist-entries配置（默认512个）
- 所有值都小于list-max-ziplist-value配置（默认64字节）

#### linkedlist
不满足ziplist条件时，用这种编码

#### quicklist
Redis3.2版本提供了[quicklist](https://matt.sh/redis-quicklist)

### 使用场景
#### 消息队列
lpush+brpop命令，可实现阻塞队列
#### 队列
lpush+rpop
#### 栈
lpush+lpop

## 集合 set
集合也是用来存储多个字符串元素，但是集合无需，而且不允许元素重复。

### 命令
#### 集合内操作
- sadd
- seem
- scard
- sismember
- srandmember
- spop
- smembers

#### 集合间操作
- sinter
- sinterstore
- sunion
- sunionstore
- sdiff
- sdiffstore
### 内部编码
#### intset
当集合同时满足以下条件时，会使用intset编码：
- 集合中的元素都是整数元素
- 元素个数小于set-max-intset-entries配置(512)

#### hashtable
当集合类型不能满足intset的条件时，使用hashtable编码

### 使用场景
#### 标签系统
一个用户可以有多个标签，一个标签下会有多个用户。
用并集可以得到两个用户共同的标签，也可以得到同时有两个标签的所有用户。

#### 抽奖
spop/srandmember = Random item

## 有序集合 zset
有序集合是在集合的基础上，为每个元素增加了score，然后所有的元素就可以排序了，因此有序集合可以获取指定范围的元素、指定分数范围内的元素，计算元素排名。

### 命令
#### 集合内
- zadd
- zcard
- zscore
- zrank
- zrevrank
- zrem
- zincryby
- zrange
- zrevrange
- zrangebyscore
- zrevrangebyscore
- count
- zremrangebyrank
- zremrangebyscore

#### 集合间
- zinterstore
- zunionstore

### 内部编码
#### ziplist
同时满足以下条件时：
- 有序集合的元素个数小于zset-max-ziplist-entries（默认128）
- 每个元素的值都小于zset-max-ziplist-value（默认64字节）

#### skiplist
不满足ziplist的条件时，使用skiplist

### 使用场景
#### 排行榜

## Stream (\>5.0)

# 功能
## 键管理
- 重命名：rename/renamenx
- 随机：randomkey
- 键迁移
	- move
	- dump+restore
	- migrate
- 键遍历
	- keys
	- scan
	- hscan
	- sscan
	- zscan
## 键过期功能
- [expire](https://redis.io/commands/expire)/pexpire/pexpireat可以对一个键设置过期时间，当超过过期时间后，会自动删除这个键
- persist清除键的过期时间
- 需要注意的是，set命令也会清除过期时间
- setex可以修改数据的同时设置过期时间
- [ttl](https://redis.io/commands/ttl)用于查询一个键的剩余时间，-1表示未设置过期时间，-2表示键不存在

## 慢查询分析
- 慢查询统计的是命令执行的时间
- 配置
	- slowlog-log-slower-than设置慢操作阈值，默认10000微秒
	- slowlog-max-len是慢日志的长度（条数）
- 命令
	- slowlog get
	- slowlog len
	- slowlog reset
- 结构
	- ID
	- time
	- duration
	- command+arguments
### 作用
可以基于这个机制实现监控功能

## Redis Shell
### redis-cli
- -r 表示执行命令多次
- -i 表示命令执行间隔
- -x 表示从stdin读取数据作为最后一个参数
- -c 是连cluster节点时的选项
- -a 提供密码
- --rdb 请求redis实例生成并发送RDB文件
- --bigkeys 使用scan命令进行采样，找到内存占用比较大的键值
- --eval用于执行指定Lua脚本
- --latency/latency-history/latency-list可以检测网络延迟
- --stat实时获取统计信息
- --no-raw/raw，返回的是否是二进制格式

### redis-benchmark
- -c 并发（默认50）
- -n 客户端请求总量（默认100000）
- -q 只显示request per seconde
- -r 生成更多的随机键
- -P 表示每个请求pipeline的数据量（默认1）
- -k 是否使用keepalive
- -t 对指定命令进行测试
- --csv 以csv格式输出结果

## 阻塞问题调查
### CPU饱和
前面有提到—stat实时获取统计信息，还有info commandstats也可以统计各命令的开销
[http://www.redis.io/topics/latency](http://www.redis.io/topics/latency)
#### CPU竞争
常用优化措施是给Redis进程绑定到某个CPU上，但是这种情况下，如果执行bgsave，子进程会和父进程争夺CPU资源，会导致父进程没有足够的CPU资源。
所以，如果需要执行bgsave的话，还是不要做绑定CPU的操作比较好。

#### 内存交换
使用以下命令查看内存交换情况：
```bash
cat /proc/[pid]/smaps | grep Swap
```
如果出现大量的内存交换，说明是有问题的，要导致性能大幅度下降。

要做到以下几点：
- 保证有充足的可用内存
- 要设置maxmemory
- 降低系统使用swap的优先级：echo 10 \> /proc/sys/vm/swappiness

#### 网络问题
- 网络闪断
- Redis连接数耗尽。这个我们再项目里遇到过，需要优化Redis的连接使用方法
- 连接溢出
	- 进程限制：ulimit -n
	- backlog队列：cron定时执行netstat -s | grep overflowed

## 内存
### 内存消耗
```bash
# 反应了内存碎片的严重程度，越大越严重。最好的情况是等于1
# 小于1时，说明内存被交换到硬盘了，这是很严重的问题
mem_fragmentation_ration = used_memory/used_memory_rss
```

内存消耗由以下几部分组成：
- 自身内存
- 对象内存
- 缓冲内存
- 内存碎片

其中自身内存非常小，在10MB以内，重点是其它三种内存

#### 对象内存
因为Redis里数据都是K-V对，所以每次都会至少生成两个类型对象：key和value。key的长度也要注意不要过长，也要合理预估value对内存的占用情况。

#### 缓冲内存
包括：客户端缓冲区、主从同步缓冲区、AOF缓冲区。

客户端缓冲区，是接入到Redis的客户端的输入输出缓冲区，如果客户端处理数据过慢，就会导致数据积压在缓冲区内，导致Redis内存占用上升。这个问题我们也遇到过，定位方法就是使用下面的命令找到缓冲区过大的client：
```bash
redis-cli client list | grep -v "omem=0"
```
然后顺藤摸瓜，找到有问题的逻辑，并针对性的改进。

主从同步缓冲区，就是主从同步时使用的缓冲区了。要注意的是全量同步时也是占用这个缓冲区，不够大的话是会同步失败的。这部分知识还是我们摔了几个大跟头才知道的。

AOF缓冲区，就是AOF写入命令还有重写AOF时使用的内存，这部分用户干预不了。

#### 内存碎片
内存碎片的产生，主要是来源于频繁的内存分配和释放，以及大小不一的内存块。
在Redis的使用上，就是频繁的数据更新和删除，还有就是剧烈的数据长度变化。

我们现在就面临这样的问题，两个最大的Redis实例的碎片比例分别是1.33和1.15，非常的浪费内存。
分析下来，我认为主要原因还是我们不同类型的数据放在一个实例里面，不同类型的数据长度差别非常大，而且一部分数据是常驻数据，一部分数据是缓存数据，完美的内存碎片产生场景。
所以，下一步我准备进一步拆分实例，把缓存放在一个实例，常驻数据放在一个实例里。



## Pipeline
RTT（Round Trip Time）往返时间，是发送命令和接收结果在网络上传输的时间。

有些命令带批量模式，可以整体上节约RTT，但是大部分的命令是不带批量模式的，于是有了Pipeline。Pipeline把多条指令一次性传输给Redis，然后把返回结果按顺序返回给客户端，只需要一次RTT。

注意：Pipeline过大，反而会导致请求过大或者返回过大，导致等待时间过长。

## 事务与Lua
### 事务
- multi
- exec
- discard

Redis的事务不支持回滚，命令执行错误不会影响事务的执行。
### Lua
- eval
- evalsha
- script load
- script exists
- script flush
- script kill

Redis-cli —eval xxx.lua aa bb可以测试脚本

## 发布订阅功能
命令：
- publish
- subscribe
- psubscribe - 按模式匹配订阅
- unsubscribe
- punsubscrib
- pubsub channels
- pubsub numsub

不支持历史功能

## GEO
命令：
- geoadd
- geopos
- geodist
- georadius
- georadiusbymember
- geohash
- zrem - 删除成员，因为底层是使用set实现的
## 客户端管理
### client list
Client list命令可以列出与Redis服务端相连的所有客户端连接信息，包含以下字段：
- id：唯一标识，Redis启动后从0开始，自增
- addr：ip和端口
- fd：socket的文件描述符，如果是-1就标识是Redis内部伪装的客户端
- name：名字，可以通过client setName和client getName操作
- age：连接的存活时间，单位是秒
- idle：连接的闲置时间，单位是秒
- flags：
	- A: connection to be closed ASAP
	- b: the client is waiting in a blocking operation
	- c: connection to be closed after writing entire reply
	- d: a watched keys has been modified - EXEC will fail
	- i: the client is waiting for a VM I/O (deprecated)
	- M: the client is a master
	- N: no specific flag set
	- O: the client is a client in MONITOR mode
	- P: the client is a Pub/Sub subscriber
	- r: the client is in readonly mode against a cluster node
	- S: the client is a replica node connection to this instance
	- u: the client is unblocked
	- U: the client is connected via a Unix domain socket
	- x: the client is in a MULTI/EXEC context
- db：当前的db
- psub：模式匹配的订阅数量
- multi：事务命令数量
- qbuf：客户端的输入缓冲区总容量，不能超过1G，超过的话会被关闭
- qbuf-free：客户端的输入缓冲区剩余容量
- obl：输出缓冲区长度
- oll：输出列表长度，排队的返回值会存在这里
- omem：输出缓冲区内存占用
- events：文件描述符事件
	- r: the client socket is readable (event loop)
	- w: the client socket is writable (event loop)
- cmd：最后一次执行的命令

这里的qbuf、qbuf-free、obl、oll、omem都是需要监控的对象

输出缓冲区可以使用client-output-buffer-limit配置来控制，输出缓冲区分普通客户端、发布订阅客户端、slave客户端

内存的占用也要监控，如果内存抖动频繁，可能是输出缓冲区过大导致的

配置中的格式：
client-output-buffer-limit \<class\> \<hard limit\> \<soft limit\> \<soft seconds\>
- class
	- normal
	- slave
	- pubsub
- hard limit 缓冲区大小超过这个值，client会被关闭
- 缓冲区大小超过soft limit持续soft seconds，client也会被关闭

## 持久化
### AOF
- Append Only File
- 记录Redis操作命令，重启时重新执行命令
- 文件同步
	- always 同步写入磁盘
	- everysec 异步，每秒同步磁盘一次
	- no 由系统同步磁盘
#### AOF重写流程
![](Screen%20Capture%20May%2027,%202019%20at%2022.22.22.jpg)
### RDB
- 当前进程数据的快照
- 优点
	- 紧凑，体积小
	- 加载比AOF快
- 缺点
	- 无法做到实时持久化/秒级持久化
	- 存在不同版本间格式不兼容的问题
- 触发方式
	- 手动触发：save/bgsave
	- 自动触发
		- 配置 save m n，m秒内n次修改时，触发bgsave
		- 主从节点全量复制，也会触发bgsave
		- shutdown命令会触发bgsave
		- debug reload会触发save
#### bgsave流程
![](Screen%20Capture%20May%2027,%202019%20at%2022.22.22-1.jpg)

### 重启加载流程
![](Redis%E9%87%8D%E5%90%AF%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B.jpg)
### 相关问题
#### fork操作耗时
- 内存量越大，fork操作越耗时
- 使用虚拟化技术也会导致fork操作时间变长
- info stats中latest-fork-usec是最近一次fork操作的耗时，单位微妙
#### 多实例部署
- 同时写磁盘会导致效率下降
- 可以通过监控Redis运行状态，外部执行存档操作


## 复制功能
- 配置方法
	- 配置文件 slaveof 
	- 命令 slaveof
	- 参数 —slaveof
	- slave的密码要和master保持一致

### 拓扑
- 1 master + 1 slave
- 1 master + n slave
- 树状主从结构

树状主从结构比较有意思，我之前没想过这种用法。之前项目里用的配置时一主三从，发生网络波动时，有很大的概率发生同步风暴。而树状结构就很好，能分摊同步压力。当然了，这样也会导致叶子节点的延迟更高了。

### 主从关系建立流程
1. 保存主节点信息
2. 建立socket连接
3. 发送ping命令
4. 权限验证
5. 全量同步
	1. 主节点bgsave
	2. 主节点传输rdb文件给从节点
	3. 从节点加载rdb文件
6. 增量同步
	1. 主节点根据从节点同步偏移量发送数据
	2. 从节点更新数据，并回复新的偏移量
> master_repl_offset - slave_offset 根据这个差值判定当前复制的健康度。_

### 读写分离
读写分离实现起来并不难，不过我想讨论另一个问题：Redis的读写分离是否有意义？
首先读写分离就面临几个新问题：
- 客户端要实现读写分离
- 主从之间有数据延迟，意味着读到的数据可能已经过期
- 更多的节点意味着更多的故障可能性
- 前面说提到的主从结构存在的风险

面临上面这些问题，就是为了提高读取效率。也就是说读取的QPS达到了单个实例的极限，那么我把实例进行分片不是一样能达到提高整体的QPS的效果么？而且还能提高整体的存储容量，反正本来的主从结构就是要有同样的内存占用。
当然，如果无法将查询请求平均切分，或者切分之后的QPS还是非常的高，那就只能使用读写分离了。

## Redis Sentinel
哨兵由哨兵结点、主数据结点、从数据结点、客户端结点组成，其中哨兵结点也分主从，会通过选举选出主哨兵结点。
哨兵结点负责：
- 监控各个数据结点状态
- 通知各个客户端数据结点的状态变化
- Failover，主节点挂掉之后，哨兵会把从节点提升为主节点
- 哨兵还会根据实际情况，更新redis的配置

### 部署
Sentinel结点要设置奇数个，因为需要投票确认主结点，奇数不会平票。
集群里一定要有主从结构

redis-sentinel redis-sentinel.conf

### 配置
#### sentinel monitor
```bash
sentinel monitor <master-name> <ip> <port> <quorum>
```
配置一个叫master-name的主节点，工作在ip和端口port，quorum是判断不可达时需要的票数

#### sentinel down-after-milliseconds
```bash
sentinel down-after-milliseconds <master-name> <times>
```
每个Sentinel结点都会发ping来判断Redis数据结点是否可达，如果超过times毫秒就判断不可达。

#### sentinel parallel-syncs
```bash
sentinel parallel-syncs <master-name> <nums>
```
发生故障转移时，同时发起复制操作的从节点数。

#### sentinel failover-timeout
```bash
sentinel failover-timeout <master-name> <times>
```
故障转移超时时间，作用于多个环节：
- 选出合适从节点
- 把选中的从节点升级为主节点 \<-
- 命令其它从节点复制新的主节点 \<-
- 等待原主节点恢复后命令他去复制新的主节点

#### sentinel auth-pass
```bash
sentinel auth-pass <master-name> <password>
```
设置主节点的访问密码

#### sentinel notification-script
```bash
sentinel notification-script <master-name> <script-path>
```
当发生一些事件时，触发指定的脚本，会发送响应的事件参数

#### sentinel client-reconfig-script
```bash
sentinel client-reconfig-script <master-name> <script-path>
```
故障转移结束后，触发指定的脚本，参数如下：
```bash
<master-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>
```
- master-name： 主节点名
- role： Sentinel的角色，leader和observer
- from-ip： 原主节点的ip
- from-port： 原主节点的端口
- to-ip： 新主节点的ip
- to-port： 新主节点的端口

有关sentinel notification-script和sentinel client-reconfig-script有几点需要注意: 
- \<script-path\>必须有可执行权限。
- \<script-path\>开头必须包含shell脚本头(例如#!/bin/sh)
- Redis规定脚本的最大执行时间不能超过60秒,超过后脚本将被杀掉。
- 如果shell脚本以exit 1结束,那么脚本稍后重试执行。如果以exit 2或者更高的值结束,那么脚本不会重试。正常返回值是exit 0。
- 如果需要运维的Redis Sentinel比较多,建议不要使用这种脚本的形式来进行通知,这样会增加部署的成本。

### 命令
#### sentinel masters
展示监控的主节点的状态和统计信息

#### sentinel master\<master name\> 
展示指定\<master name\>的主节点状态以及相关的统计信息

#### sentinel slaves\<master name\> 
展示指定\<master name\>的从节点状态以及相关的统计信息

#### sentinel sentinels\<master name\>
展示指定\<master name\>的Sentinel节点集合

#### sentinel get-master-addr-by-name\<master name\>
返回指定\<master name\>主节点的IP地址和端口

#### sentinel reset\<pattern\>
当前Sentinel节点对符合\<pattern\>(通配符风格)主节点的配置进行重置,包含清除主节点的相关状态(例如故障转移),重新发现从节点和Sentinel节点。

#### sentinel failover\<master name\>
对指定\<master name\>主节点进行强制故障转移(没有和其他Sentinel节点“协商”),当故障转移完成后,其他Sentinel节点按照故障转移的结果更新自身配置

#### sentinel ckquorum\<master name\> 
检测当前可达的Sentinel节点总数是否达到\<quorum\>的个数。

#### sentinel flushconfig
将Sentinel节点的配置强制刷到磁盘上,这个命令Sentinel节点自身用得比较多,对于开发和运维人员只有当外部原因(例如磁盘损坏)造成配置文件损坏或者丢失时,这个命令是很有用的。

#### sentinel remove\<master name\> 
取消当前Sentinel节点对于指定\<master name\>主节点的监控。

#### sentinel monitor\<master name\>\<ip\>\<port\>\<quorum\> 
这个命令和配置文件中的含义是完全一样的,只不过是通过命令的形式来完成Sentinel节点对主节点的监控。

#### sentinel set\<master name\> 
动态修改Sentinel节点配置选项

#### sentinel is-master-down-by-addr
Sentinel节点之间用来交换对主节点是否下线的判断,根据参数的不同,还可以作为Sentinel领导者选举的通信方式

### 三个定时任务
- 定时向主从节点发送info指令，获取拓扑结构
- 定时向`__sentinel__:hello`发送对于主节点的判断和自己的信息
- 定时向其他节点发送ping，判断存活状态

### 主观下线和客观下线
当ping的指令没有再down-after-millisenconds毫秒内回复，则判断该节点为主观下线
当主Sentinel判定一个节点主观下线时，会通过sentinel is-master-down-by-addr向其它Sentinel节点询问对该主节点的判断，当超过quorum个时，就会判断该节点是客观下线

### 选举
[Raft](https://raft.github.io/)

### 故障转移
1. 在从节点列表中选出一个节点作为新的主节点,选择方法如下: 
	- 过滤:“不健康”(主观下线、断线)、5秒内没有回复过Sentinel节点ping响应、与主节点失联超过down-after-milliseconds*10秒。
	- 选择slave-priority(从节点优先级)最高的从节点列表,如果存在则返回,不存在则继续。
	- 选择复制偏移量最大的从节点(复制的最完整),如果存在则返回,不存在则继续。
	- 选择runid最小的从节点。
2. Sentinel领导者节点会对第一步选出来的从节点执行slaveof no one命令让其成为主节点。
3. Sentinel领导者节点会向剩余的从节点发送命令,让它们成为新主节点的从节点,复制规则和parallel-syncs参数有关。
4. Sentinel节点集合会将原来的主节点更新为从节点,并保持着对其关注,当其恢复后命令它去复制新的主节点。

## Redis Cluster
### 数据分布
#### 常见分区方式
##### 哈希分区
特点：
- ✅离散度好
- ✅数据分布业务无关
- ❌无法顺序访问

代表产品：
- Redis Cluster
- Cassandra
- Dynamo

常见规则：
- 节点取余
	- 使用特定的数据，用公式 hash(key) % N计算出哈希值
	- 缺点：调整分区数量时，会导致数据的重新迁移
	- 优点：实现简单
- 一致性哈希分区（Distributed Hash Table)
	- 为每个节点分配一个0-2^32的token，组成一个hash环。数据读写执行节点查找操作时，先计算hash，然后顺时针找到第一个token节点。
	- ✅增删节点时，只影响相邻节点
	- ❌加减节点会造成部分节点要rehash
	- ❌节点比较少时，节点变化影响范围也大
	- ❌普通的一致性哈希分区在增减节点时需要增加一倍或减去一半节点才能保证数据和负载的均衡
- 虚拟槽分区
	- 把哈希空间分割成固定数量的槽，槽是集群内数据管理和迁移的基本单位
	- 每个节点会负责一定数量的槽
	- Redis的槽范围是0 - 16383
	- 计算公式：slot = CRC16(key) & 16383
		- 16383的二进制： 11111111111111
	- ❌批量操作只能在相同slot里操作
	- ❌key事务操作，只能在一个节点上操作
	- ❌key是最小粒度，不能把一个大的hash、list等映射到不同节点
	- ❌不支持多数据库空间，只有db0
	- ❌不支持树状复制结构，只支持单层复制结构
##### 顺序分区
特点：
- ❌离散度容易倾斜
- ❌数据分布和业务相关
- ✅可顺序访问

代表产品：
- BigTable
- HBase
- Hypertable

### 集群搭建
#### 准备节点
- 至少6个节点才能保证组成完整高可用集群
- 每个节点都需要开启cluster-enabled yes
```bash
	#节点端口
	port 6379 
	# 开启集群模式
	cluster-enabled yes 
	# 节点超时时间,单位毫秒
	cluster-node-timeout 15000 
	# 集群内部配置文件
	cluster-config-file "nodes-6379.conf"
	```
- 启动之后，nodes-6379.conf里会记录集群的节点ID等信息
#### 节点握手
- 节点握手：一批运行在集群模式下的节点通过Gossip协议彼此通信，达到感知对方的过程
- 由客户端发起命令：`cluster meet {ip} {port}`
- 过程
	- A创建B的节点信息对象，并发送meet消息
	- B接到meet信息之后，保存A的节点信息并回复Pong消息
	- 之后A和B定期通过ping/pong消息进行正常的节点通信
#### 分配槽
- 使用`cluster addslots {0...5461}`分配槽
- 使用`cluster replicate {nodeid}`来使当前节点成为nodeid的从节点
- 官方提供了redis-trib.rb工具来简化Redis集群管理
### 节点通信
#### 通信流程
- Gossip协议的原理就是节点间彼此不断通信交换信息，一段时间后所有的节点都会知道集群完整的信息，类似流言的传播。
- 集群中每个节点都会单独开辟一个TCP通道，用于节点间的彼此通信，通信端口号为基础端口+10000
- 每个节点在固定周期内通过特定规则选择几个节点发送ping消息
- 接受到ping消息的节点用pong消息作为响应
#### Gossip消息
Gossip消息分类：
- ping消息
	- 集群中每个节点之间互相交换的信息
	- 用于检测节点是否在线和交换彼此状态信息
- pong消息
	- 回复ping和meet信息的信息
	- 消息内部封装了自身状态数据
- meet消息
	- 通知新节点加入，发送者通知接受者加入到当前集群
- fail消息
	- 当节点判定集群内一个节点下线时，会向集群内广播一个fail消息
	- 其它节点接受到fail消息时，会把对应节点更新为下线状态
#### 节点选择
如果大量节点之间频繁的进行信息交换，势必加重带宽和计算的负担。但是简单的降低通信的节点，又会降低信息交换的频率，导致故障判定、新节点发现过慢。

- 每秒随机5次找出最久没有通信节点
- 最后通信时间大于node-timeout/2

### 集群伸缩

可以通过把槽和对应数据再不同节点之间移动，来实现伸缩。
首先，启动新节点。然后，使用cluster meet命令把新节点加入集群，也可以使用redis-trib.rb脚本来做这件事情。最后，就是迁移数据环节了，迁移数据之前，要规划好槽的分布，尽量保持所有节点的槽数量大致相等。迁移流程如下：
- 对目标节点发送`cluster setslot {slot} importing {sourceNodeId}`，让目标节点准备导入槽的数据
- 对源节点发送`cluster setslot {slot} migrating {targetNodeId}`，让源节点准备迁出槽的数据
- 源节点循环执行`cluster getkeysinslot {slot} {count}`，获取count个属于槽的键
- 在源节点执行`migrate {targetIp} {targetPort} "" 0 {timeout} keys {keys...}`，把获取的键迁移到目标节点
- 重复上面两个步骤，知道槽下所有数据完成迁移
- 向集群内所有主节点发送`cluster setslot {slot} node {targetNodeId}`，通知槽分配给目标节点。
- 当然，还是可以使用redis-trib.rb来自动完成

### 请求路由
在集群内，向其中一个节点发送操作时，如果该数据正好在该节点内，则直接执行操作，否则会返回MOVED错误，告知客户端该数据所在的节点信息。

大部分的客户端都实现了自己维护slot-\>node的映射关系，在本地就可以实现键到节点的查找，从而保证IO效率的最大化，而MOVED重定向负责协助Smart客户端更新slot-\>node的映射。

另外，在键转移过程中，访问到的键会返回ASK，此时是不需要更新映射的。

# 应用
## 能做什么
- 排行榜
- 缓存
- 关系管理
- 消息队列

## 不能做什么
### 数据规模过大的情况，不适合
Redis数据是全部存储在内存中的，这意味着内存的大小决定了能存多少数据。数据规模特别大的情况下，数据存储成本会非常高。
如果开启了RDB，做数据落地时，最大会需要当前Redis实例一倍的额外内存，即10G的数据，在落地的时候，需要额外的10G内存。也就是说要为了10G数据准备20G内存。
另外主从同步也会需要额外的内存存放通信数据，全同步的时候，最糟也会需要一倍的内存大小，如果你接入了两个Slave节点，就需要额外的两倍内存……
因此，如果使用这些特性的话，数据规模越大，你需要预留的内存就越多。即使你不启用这些特性，数据规模也受制于内存的大小。

### 冷数据不适合用Redis来存储
其实这样一点就是上面的衍生结论，因为内存很金贵，所以用来放一些不常用的数据是十分不经济的做法。合适的做法是引入数据过期机制，冷数据直接淘汰掉，提高热数据比例。

## 缓存
### 数据更新
数据发生变化时，直接更新缓存，这样不存在数据一致性问题。

一开始考虑使用idletime来判断数据是否已经落地，考虑了一下，这个方案还是不太稳定，所以最终决定使用set来保存已经落地的数据，当数据再变化时，从set里移除。后面以savedkeys指代这个集合。

### 空间回收
定时扫描savedkeys里的key，配合object debug或者object lru命令，回收idletime超过某个阈值的key。
这个阈值会随着剩余容量的降低而降低，但是会有一个下限，因为低于某个数字之后的数据变化概率是很高的，盲目回收反而会导致震荡的情况。当阈值接近下限时要引起警觉，要有预警机制。

另外回收空间的进程可以考虑使用unix socket进行本机通信，这样应该能进一步提高Redis工作效率。

### 问题：缓存穿透
对于我们游戏来说，缓存穿透只会发生在创建新角色。而这个动作是确定的，在查询发起之前就可以确定，这种情况就直接创建记录就可以了。

### 问题：无底洞问题
我们现在的游戏都是分区的，天然可以很好的分组。
即使是全球服游戏，也可以按玩家的ID做切分，只是会面临扩容时key的迁移问题，可以考虑翻倍扩容的方案。

其实我们的缓存系统里实现了`hash_tag`实现，是为了调试方便，可以强制把玩家导向一个指定服务器，放到一个具体的数据组里。

### 问题：雪崩
雪崩问题指的是缓存崩溃之后，大量的请求直接打到后端数据库上，导致后端数据库跟着一起崩溃的情况。
我们现在的解决方案是服务降级，会向上级服务报警，只服务现在在线的玩家，新玩家会收到“服务器繁忙”的通知。

### 问题：热点key失效
总是存在一些访问量特别大的key，当这些key过期时，有可能会同时发起多个更新请求，也会造成不必要的运算和负载。

常规解决思路：
- 互斥锁：只允许一个线程重建缓存
- “永不过期”：缓存数据不会因为时间到了而自动过期，于是就不会出现大规模刷新的请求。那么如何保证数据的同步呢？可以设置一个过程，定时去检查该key是否过期。

### 问题：bigkey
发现方法：
- redis-cli  --bigkeys
- scan + debug object

### 调优
#### THP
THP（Transparent Huge Pages）特性，会使得内存页从4KB变成2MB，子进程fork之后，因写操作引起的复制操作要复制的数据量变成了原来的512倍，会拖慢写操作的执行时间。因此最好不要开启这个特性。
```bash
echo never >  /sys/kernel/mm/transparent_hugepage/enabled
```

#### NTP
NTP（Network Time Protocol，网络时间协议）是用来保证不同机器的时钟一致的服务，对于分布式服务非常重要，需要定时区同步一下。
```bash
0 * * * * /usr/sbin/ntpdate ntp.xx.com > /dev/null 2>&1
```

#### ulimit -n + maxclients
`ulimit -n`是Linux系统中一个用户可以打开的最大文件数量。
在/etc/security/limits.conf中增加以下几行
```bash
* soft nofile 65536
* hard nofile 65536
```

#### rename-command
先看一个通过redis攻击的流程：
1. 获取redis的访问权限
```bash
	cat my.pub | redis-cli -h xxx -p xxx -x set crackit
	```
3. 登录上redis后，覆盖`/root/.ssh/authorized_keys`
	```bash
	config set dir /root/.ssh
	config set dbfilename authorized_keys
	save
	```
4. 攻击完成，现在可以登录被攻击的服务器了

为了避免上面的攻击，一方面要设置防火墙，另一方面，一些敏感的指令也都要改名：
- keys
- flushall/flushdb
- save
- debug
- config
- shutdown
- auth

命令改名方法是在配置文件里加上:
`rename-command keys asdfasdfa`
不过rename-command会对AOF和RDB有一定的影响，改命令之前如果用过这些命令，可能导致无法启动。另外还有一部分指令是无法改名的。
值得注意的是rename-command必须在配置文件里加，一定要一开始就把这些加上。

### maxmemory机制
内存使用达到maxmemory上限时，会触发内存溢出控制策略。

Redis的过期对象删除策略：
- 惰性删除，当客户端读取已经超时的键时，会执行删除并返回空
- 定时删除，按照一定的逻辑，定时删除过期键
	- 每个数据库空间随机抽查20个键
		- 是否超过25%的键过期？
		- 否-\>退出
		- 是 - 循环清除，直到超时或者比例下降到25%以下

内存溢出控制策略(maxmemory-policy)：
- noeviction：不会删除任何数据
- volatile-lru：LRU算法，会回收设置了超时属性的键
- allkeys-lru：LRU算法，无差别回收键
- allkeys-random：随机删除键
- volatile-random：随机删除有超时属性的键
- valatile-ttl：删除最近要过期的键
