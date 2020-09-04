---
title: TokuDB · 特性分析· 日志详解 · 数据库内核月报 · 看云
category: default
tags: 
  - www.kancloud.cn
created_at: 2020-04-30 11:36:06
original_url: https://www.kancloud.cn/taobaomysql/monthly/67176
---


# TokuDB · 特性分析· 日志详解 · 数据库内核月报 · 看云

TokuDB的日志跟InnoDB不一样，它有两类文件:

1.  redo-log文件(以.tokulog\[序号\]为扩展名)
2.  rollback日志文件(tokudb.rollback)

接下来就简单唠唠这两类文件的内部细节。

**1) redo-log**

记录的不是页而是对Fractal-Tree索引的操作日志。 log格式:

```plain
| length | command | lsn | content | crc|
```

content里记录的是具体的日志内容，比如insert操作，content就是:

```plain
| file-no | txnid | key | value|
```

TokuDB在做恢复的时候，会找到上次checkpoint时的LSN位置，然后读取log逐条恢复。

为了确保log的安全性，redo-log也支持从后往前解析。

当一个log的MAX_LSN小于已完成checkpoint的LSN时，就表明这个log文件可以安全删除了。

那么问题来了：

如果用户执行了一个“大事务”，比如delete一个大表，耗时很长，log文件岂不是非常多，一直等到事务提交再做清理？

不用的，这就是tokudb.rollback的作用了。

**2) tokudb.rollback**

用户的事务操作(insert/delete/update写操作)都会写一条日志到tokudb.rollback，存储的格式是:

```plain
|txnid | key|
```

记录日志伪码如下：

```plain
void ft_insert(txn,...)
{
   if (txn) {
       toku_logger_save_rollback_cmdinsert(...);
   }

   if (do_logging && logger) {
       toku_log_enq_insert(....);
   }
}
```

如果是事务，每个操作会写2条日志(1条redo，1条rollback)。

如果用户执行了commit/rollback，TokuDB会根据txnid在tokudb.rollback里查到key(如果该entry不在cache里)，再根据key在索引文件里找到相应的事务信息并做相应的commit/rollback操作。

tokudb.rollback可以看做是一个事务的undo日志，记录的是的关系映射。

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/67176)

创建于: 2020-04-30 11:36:06

目录: default

标签: `www.kancloud.cn`

