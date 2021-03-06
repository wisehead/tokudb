---
title: TokuDB· 参数故事·数据安全和性能 · 数据库内核月报 · 看云
category: Database
tags: 
  - www.kancloud.cn
created_at: 2020-04-29 10:58:38
original_url: https://www.kancloud.cn/taobaomysql/monthly/67125
---

# TokuDB· 参数故事·数据安全和性能

TokuDB里可调优的参数不多，今天把＂最重要＂的几个拉出来晒晒。

与性能相关的参数及说明:

```plain
 tokudb_cache_size(bytes):
 缓存大小，读写时候，数据会首先会缓存到这里。
 默认大小为机器物理内存的一半。
 tokudb_commit_sync(ON/OFF):
 当事务提交的时候，是否要fsync log到磁盘。
 默认开启(ON)，如果设置为OFF，性能会提升，但可能会丢失事务(commit记录到log buffer，但是未fsync到磁盘的事务)。
 tokudb_directio(ON/OFF):
 是否开启Direct I/O功能，TokuDB在写盘的时候，无论是否开启Direct I/O，都是按照512字节对齐的。
 默认为OFF。
 tokudb_fsync_log_period(ms):
 多久fsync一下log buffer到磁盘，TokuDB的log buffer总大小为32MB且不可更改。
 默认为0ms(此时做fsync的后台线程一直处于wait状态)，此时受tokudb_commit_sync开关控制是否要fsync log到磁盘(checkpoint也会fsync log buffer的，默认为1分钟)。
```

针对不同的使用场景：

1.  对数据要求较高(不允许丢失数据，事务ACID完整性)，只需根据内存调整tokudb\_cache\_size大小即可，建议开启tokudb_directio。
    
2.  对数据要求不太高(允许部分数据丢失，不要求事务ACID完整性)，可配置:
    

```plain
 tokudb_commit_sync=OFF
 tokudb_fsync_log_period=1000 #1s
```

在此配置下，每1秒对log buffer做下fsync，可充分利用log的group commit功能，如果TokuDB挂掉，则可能会丢失最多1秒的数据。

