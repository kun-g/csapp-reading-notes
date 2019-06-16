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

## Redis Cluster

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
