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
### HyperLogLog

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

## Geo（\>3.2)

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
[expire](https://redis.io/commands/expire)/pexpire/pexpireat可以对一个键设置过期时间，当超过过期时间后，会自动删除这个键
persist清除键的过期时间
需要注意的是，set命令也会清除过期时间
setex可以修改数据的同时设置过期时间
[ttl](https://redis.io/commands/ttl)用于查询一个键的剩余时间，-1表示未设置过期时间，-2表示键不存在

## 发布订阅功能

## Lua脚本功能

## 简单的事务功能

## 流水线功能

## 持久化
### AOF
### RDB

## 复制功能

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
