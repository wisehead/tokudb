---
title: TokuDB · 引擎特性 · REPLACE 语句优化 · 数据库内核月报 · 看云
category: default
tags: 
  - www.kancloud.cn
created_at: 2020-05-27 20:11:46
original_url: https://www.kancloud.cn/taobaomysql/monthly/213794
---

# TokuDB · 引擎特性 · REPLACE 语句优化

## 背景

MySQL 在标准 SQL 外，会扩展一些好用的语法，本文关注的 REPLACE 和 INSERT IGNORE 就属于这类。这 2 个语法都是对 INSERT 的扩展，语义是向表中插入数据，不同之处在于遇到 PK 或者 UK 冲突时的处理：

1.  INSERT：报 duplicate key 的错误，数据不插入；
2.  REPLACE：删除掉老冲突记录，插入新的记录；
3.  INSERT IGNORE：不插入数据，但是不报错。  
    可以看到，REPLACE 的语义是用新数据取代老数据，INSERT IGNORE 的语义是保留老的数据。

本文将向大家介绍 TokuDB 引擎对这个2个语法的优化。

## 优化分析

我们先看下优化前，一次 REPLACE 和 INSERT IGNORE 都需要做哪些操作。

对于 REPLACE：  
1\. 先尝试做 insert，因为 TokuDB 插入是异步的，为了保证唯一性约束，会先做一次 search，看是否有已经有记录；  
2\. 如果老记录不存在，就直接插入；如果存在，就向 server 层报 dup key 错误；  
3\. server 层拿到 dup key 错误后，再向引擎发一次 search 操作，把老记录捞出来；  
4\. server 层有了老记录和要插入的数据，调引擎层的 update 接口做更新（其实这里应该做 delete + insert，server 层做了一个优先，只需要调一次引擎接口；如果有外键或者有delete触发器的话，还是会做 delete + insert 的，可以参考 `write_record()` 函数）。

对于 INSERT IGNORE：  
1\. 先尝试做 insert，因为 TokuDB 插入是异步的，为了保证唯一性约束，会先做一次 search，看是否有已经有记录；  
2\. 如果老记录不存在，就直接插入；如果存在，就向 server 层报 dup key 错误；  
3\. server 层拿到 dup key 错误后，发现设置了 ignore，就正常返回。

TokuDB 优化后是怎样的呢，REPLACE 和 INSERT IGNORE 只需要做一次插入即可，TokuDB 对写是做了优化的，只需要将 msg 放在 FT 的 root 节点即可，后台线程会异步将其应用到 leaf 节点（参考[TokuDB索引结构–Fractal Tree](http://mysql.taobao.org/monthly/2016/04/09/)），所以性能提升是比较明显的。

做优化的调用栈如下：

```plain
#0  do_ignore_flag_optimization()
#1  ha_tokudb::set_main_dict_put_flags()
#2  ha_tokudb::insert_row_to_main_dictionary()
#3  ha_tokudb::write_row()
#4  handler::ha_write_row()
#5  write_record()
#6  mysql_insert()
#7  mysql_execute_command()
```

主要代码逻辑在 `ha_tokudb::set_main_dict_put_flags()` 和 `do_ignore_flag_optimization()` 这2个函数中。

`do_ignore_flag_optimization()` 判断能否做这个优化：

```plain
static inline bool do_ignore_flag_optimization(
    THD* thd,
    TABLE* table,
    bool opt_eligible) {

    bool do_opt = false;
    if (opt_eligible &&
        (is_replace_into(thd) || is_insert_ignore(thd)) &&
        tokudb::sysvars::pk_insert_mode(thd) == 1 &&
        !table->triggers &&
        !(mysql_bin_log.is_open() &&
         thd->variables.binlog_format != BINLOG_FORMAT_STMT)) {
        do_opt = true;
    }
    return do_opt;
}
```

`ha_tokudb::set_main_dict_put_flags()` 根据 `do_ignore_flag_optimization()` 返回的结果和当前语句设置 put\_flag。

```plain
    else if (using_ignore_flag_opt && is_replace_into(thd)
            && !in_hot_index)
    {
        *put_flags = old_prelock_flags;
    }
    else if (opt_eligible && using_ignore_flag_opt && is_insert_ignore(thd)
            && !in_hot_index)
    {
        *put_flags = DB_NOOVERWRITE_NO_ERROR | old_prelock_flags;
    }
    else
    {
        *put_flags = DB_NOOVERWRITE | old_prelock_flags;
    }
```

`db_put()` 中会根据前面设置的 put\_flag，决定是调用 `toku_ft_insert_unique()`，还是`toku_ft_maybe_insert()`，前者会先调用 `toku_ft_lookup()` 做唯一性检查，然后再做插入；后者则直接插入。在最终调用 `toku_ft_root_put_msg()`，将 msg 放在root节点时，会根据之前的flag 生成不同 type 的msg，如 INSERT IGNORE 的 type 就设置为 FT\_INSERT\_NO\_OVERWRITE，表示msg类型是插入，但是如果有老记录时不覆盖，后台 apply 线程在应用时，就会根据 msg 的类型采取相应的处理。

## 性能测试对比

为了能够方便的开启和关闭这个优化，笔者在代码里加了一个开关。测试是用 sysbench，开32个线程，一个事务里就一条语句（REPLACE 或者 INSERT IGNORE），表结构就是sysbench默认的，但是去掉了二级索引，另外 binlog 是关闭的，结果如下：

1.  REPLACE
    
    | 模式 | TPS | CPU% |
    | --- | --- | --- |
    | 关闭优化 | 3438.21 | 900 |
    | 开启优化 | 6590.31 | 240 |
    
2.  INSERT IGNORE
    
    | 模式 | TPS | CPU% |
    | --- | --- | --- |
    | 关闭优化 | 6165.36 | 1000 |
    | 开启优化 | 6702.45 | 240 |
    

可以看到对于 REPLACE 的优化效果非常明显，用更低的 CPU 消耗获得了更高的 TPS；对于 INSERT IGNORE，CPU 消耗大大降低，TPS 有一定提升。  
INSERT IGNORE 优化效果没有 REPLACE 这么明显，是因为 INSERT IGNORE 本身的逻辑要比 REPLACE 简单，在优化前如果冲突记录存在的话，是直接返回的。

## 使用限制

需要注意的是，这个优化并不是通用的，具体的限制如下：

1.  只能有一个PK，不能其它任务用二级索引  
    PK所在的FT做插入时，是直接把 msg 放到 root 节点的，根本就没有取可能存在的老记录，所以二级索引的更新是没法做的。
    
2.  要求 binlog 用 statement 格式，或者关闭 binlog  
    如果 binlog 是 row 格式的话，会导致备库应用报错，所有的操作都记为 Write\_rows event，如果有记录冲突的话，备库执行时直接报 dup key 错误。
    
3.  表上不能有 triger  
    这个主要是因为优化后语义被改变了，replace 在冲突时没有 delete 操作，insert ignore 引擎层永远是返回成功的。
    

上一篇：[SQLServer · 最佳实践 · RDS for SQLServer 2012权限限制提升与改善](https://www.kancloud.cn/taobaomysql/monthly/213793)下一篇：[MySQL · 专家投稿 · InnoDB物理行中null值的存储的推断与验证](https://www.kancloud.cn/taobaomysql/monthly/213795)

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/213794)

创建于: 2020-05-27 20:11:46

目录: default

标签: `www.kancloud.cn`

