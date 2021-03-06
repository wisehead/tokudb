---
title: MySQL · TokuDB · TokuDB Checkpoint机制 · 数据库内核月报 · 看云
category: default
tags: 
  - www.kancloud.cn
created_at: 2020-05-07 16:34:16
original_url: https://www.kancloud.cn/taobaomysql/monthly/67060
---

# MySQL · TokuDB · TokuDB Checkpoint机制

## 导读：TokuDB在“云端”的优势

为了降低用户数据存储成本，2015年４月份，云数据库(Aliyun RDS)增加了TokuDB引擎支持(MySQL5.6版本)，也是第一家支持TokuDB的RDS。

我们知道，当一个实例的数据空间超过TB级别时，空间存储和运维成本都是非常高的，尤其是做实例迁移和备份，整个过程耗时会非常长。

我们对线上一些大的InnoDB实例(TB级)平滑迁移到TokuDB后，数据空间从 2TB+ 直接降到 400GB ，空间成本仅为原来的五分之一，而且读写性能没有任何降低(写性能反而提升不少)。通过线上几个大实例(TB级)的使用，TokuDB的压缩比均在５倍以上，同样的空间大小，使用TokuDB后可以存５倍的容量！

TokuDB的另外一个特点就是低IO消耗，相同数据量下，IO花销基本是InnoDB的1/8，IO成本也降低了不少，同样的IOPS限制下，TokuDB写性能会更高。因为IOPS消耗较少，RDS已经在线上部署TokuDB+廉价SATA盘集群，配合“方寸山”(分库分表proxy)来提供低成本高性能的PB级日志存储能力。

这个集群使用廉价的SATA盘(IOPS能力~120)，单台物理机基本可提供30,000 TPS的写能力(tokudb\_fsync\_log\_period=0和sync\_binlog=1，针对类如日志型应用，对数据安全性要求不是很高的话，调整tokudb\_fsync\_log\_period=1000，sync\_binlog=1000，性能会更高)，利用TokuDB的大页(page size 4MB)压缩优势，尤其是对日志内容，压缩比基本在1/8以上，单机可提供160TB+的的存储能力，整个集群可轻松支持PB级。

使用TokuDB，让你随便任性！

本篇来探讨下TokuDB内部的一个重要机制: checkpoint。

TokuDB的checkpoint只有一种方式：sharp checkpoint，即做checkpoint的时候，需要把内存中所有的”脏页”都刷回磁盘。本篇就来介绍下TokuDB的sharp checkpoint的一些具体细节，使看官们对TokuDB的checkpoint有个大概了解。

## 为什么要checkpoint

TokuDB引擎里，数据存在于3个地方：

1.  Redo Log (disk)
2.  Buffer Pool (memory)
3.  Data File (disk)

为了性能，对“页”级的操作会先被Cache到Buffer Pool里，当触发某些条件后，才会把这些“脏页”刷到磁盘进行持久化，这样就带来一个问题：  
对于TokuDB来说，整个Fractal-Tree元素有两部分的“页”组成：(Buffer Pool里的“页” ) + （Data File里已持久化的”页”），如果TokuDB crash后，Buffer Pool里的“页”丢失，只剩Data File的“页”，此时的Fractal-Tree处于“混沌“状态(不一致状态)。

为了避免出现这种“混沌“状态，TokuDB需要定期(默认60s)做Checkpoint操作，把Buffer Pool里的“脏页”刷到磁盘的Data File里。

当TokuDB Crash后，只需从上次的一致性状态点开始“回放” Redo Log里的日志即可，恢复的数据量大概就是60s内的数据，快吧？嗯。

## TokuDB Checkpoint机制

TokuDB checkpoint分2个阶段：begin\_checkpoint 和 end\_checkpoint

大体逻辑如下：

begin_checkpoint：

```plain
C1, 拿到checkpoint锁
C2, 对buffer pool的page_list加read-lock
C3, 遍历page_list，对每个page设置checkpoint_pending flag
C4, 释放buffer pool page_list的读锁
```

end_checkpoint:

```plain
C5, 遍历page_list，对checkpoint_pending为true且为“脏”的page尝试获取write-lock
C6, 如果拿到write-lock，clone出来一份，释放write-lock，然后把clone的page刷回磁盘
```

以上就是整个checkpoint大概的逻辑，整个过程中，只有C6的任务最“繁重”，在这里面有几个“大活”：

1.  clone的时候如果是leaf“页”，会对原“页”重做数据均分(leaf里包含多个大小均匀的子分区) –CPU消耗
2.  刷盘前做大量压缩 –CPU消耗
3.  多线程并发的把page刷到磁盘 –IO操作

以上3点会导致checkpoint的时候出现一点点的性能抖动，下面看组数据：

```plain
[ 250s] threads: 32, tps: 5095.80, reads/s: 71330.94, writes/s: 20380.38, response time: 8.14ms (95%)
[ 255s] threads: 32, tps: 4461.80, reads/s: 62470.82, writes/s: 17848.80, response time: 10.03ms (95%)
[ 260s] threads: 32, tps: 4968.79, reads/s: 69562.25, writes/s: 19873.96, response time: 8.49ms (95%)
[ 265s] threads: 32, tps: 5123.61, reads/s: 71738.31, writes/s: 20494.03, response time: 8.06ms (95%)
[ 270s] threads: 32, tps: 5119.00, reads/s: 71666.02, writes/s: 20475.61, response time: 8.11ms (95%)
[ 275s] threads: 32, tps: 5117.00, reads/s: 71624.40, writes/s: 20469.00, response time: 8.07ms (95%)
[ 280s] threads: 32, tps: 5117.39, reads/s: 71640.26, writes/s: 20471.56, response time: 8.08ms (95%)
[ 285s] threads: 32, tps: 5103.21, reads/s: 71457.54, writes/s: 20414.24, response time: 8.11ms (95%)
[ 290s] threads: 32, tps: 5115.80, reads/s: 71608.46, writes/s: 20461.42, response time: 8.11ms (95%)
[ 295s] threads: 32, tps: 5121.98, reads/s: 71708.73, writes/s: 20484.72, response time: 8.09ms (95%)
[ 300s] threads: 32, tps: 5115.01, reads/s: 71617.00, writes/s: 20462.46, response time: 8.08ms (95%)
[ 305s] threads: 32, tps: 5115.00, reads/s: 71611.76, writes/s: 20461.79, response time: 8.11ms (95%)
[ 310s] threads: 32, tps: 5100.01, reads/s: 71396.90, writes/s: 20398.03, response time: 8.13ms (95%)
[ 315s] threads: 32, tps: 4479.20, reads/s: 62723.81, writes/s: 17913.40, response time: 10.02ms (95%)
[ 320s] threads: 32, tps: 4964.80, reads/s: 69496.00, writes/s: 19863.60, response time: 8.63ms (95%)
[ 325s] threads: 32, tps: 5112.19, reads/s: 71567.45, writes/s: 20447.56, response time: 8.12ms (95%)
```

以上为sysbench测试数据(读写混合)，仅供参考，可以看到在\[255s\]和\[315s\]checkpoint的时候，性能有个抖动。

喵，另外一个问题来了：如果TokuDB在end_checkpoint C6的时候crash了呢，只有部分“脏页”被写到磁盘？此时的数据库(Fractal-Tree)状态岂不是不一致了？

TokuDB在这里使用了copy-on-write模型，本次checkpoint的“脏页”在刷到磁盘的时候，不会覆写上次checkpoint的文件区间，保证在整个checkpoint过程中出现crash，不会影响整个数据库的状态。

本篇只是大概介绍了TokuDB的checkpoint机制，还有非常多的细节，感兴趣的同学可阅读[ft/cachetable](https://github.com/Tokutek/ft-index/tree/master/ft/cachetable)代码。

上一篇：[MySQL · 引擎特性 · Innodb change buffer介绍](https://www.kancloud.cn/taobaomysql/monthly/67059)下一篇：[PgSQL · 特性分析 · 时间线解析](https://www.kancloud.cn/taobaomysql/monthly/67061)

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/67060)

创建于: 2020-05-07 16:34:16

目录: default

标签: `www.kancloud.cn`

