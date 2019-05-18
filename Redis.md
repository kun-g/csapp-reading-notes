Redis介绍

# Redis是什么
全称是 *Re*mote *D*ictionary *S*erver
也就是基于数据结构的远程字典服务器

# 提供的数据结构
## 字符串
### Bitmaps
### HyperLogLog

## 哈希

## 列表

## 集合

## 有序集合

## Stream (>5.0)

## Geo（>3.2)

# 功能
## 键过期功能
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
- 数据规模过大的话，不适合Redis
- 冷数据不适合用Redis来存储
