---
title: TokuDB · 捉虫动态 · MRR 导致查询失败
category: default
tags: 
  - mysql.taobao.org
created_at: 2020-05-28 14:18:28
original_url: http://mysql.taobao.org/monthly/2017/04/08/
---

[

# 数据库内核月报 － 2017 / 04

](http://mysql.taobao.org/monthly/2017/04)

[‹](http://mysql.taobao.org/monthly/2017/04/07/)

[›](http://mysql.taobao.org/monthly/2017/04/09/)

*   [当期文章](#)

## TokuDB · 捉虫动态 · MRR 导致查询失败

## 问题背景

最近有用户在使用 TokuDB 时，遇到了一个查询报错的问题，这里给大家分享下。

具体的报错信息是这样的：

```plain
mysql> select * from t2 where uid > 1 limit 10;
ERROR 1030 (HY000): Got error 1 from storage engine
```

表结构如下：

```sql
CREATE TABLE `t2` (
  `id` bigint(20) NOT NULL,
  `uid` bigint(20) DEFAULT NULL,
  `post` text,
  `note` text,
  PRIMARY KEY (`id`),
  KEY `idx_uid` (`uid`)
) ENGINE=TokuDB DEFAULT CHARSET=utf8
```

## 问题分析

从报错信息来看，是引擎层返回错误的，难道是 TokuDB 数据出问题了么，我们首先要确认的是用户数据是否还能访问。

从表结构来看，出错的语句应该走了二级索引，那么我们强制走 PK 是否能访问数据呢。

```sql
select * from t2 force index(primary) where uid > 1 limit 3;
xxx
xxx
xxx
3 rows in set (0.00 sec)
```

上面的测试可以说明走 PK 是没问题呢，那么问题可能在二级索引。

同时我们在观察用户的其它 SQL 时发现，二级索引也是可以访问数据的。

比如下面这种：

```sql
select * from t2  where uid > 1 order by uid limit 3;
xxx
xxx
xxx
3 rows in set (0.00 sec)
```

都是走二级索引，为什么有的会报错呢，这 2 条语句有啥区别呢，explain 看下：

```plain
mysql> explain select * from t2  where uid > 1 limit 3;
+----+-------------+-------+-------+---------------+---------+---------+------+--------+-----------------------------------------------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref  | rows   | Extra                                         |
+----+-------------+-------+-------+---------------+---------+---------+------+--------+-----------------------------------------------+
|  1 | SIMPLE      | t2    | range | idx_uid       | idx_uid | 9       | NULL | 523677 | Using index condition; Using where; Using MRR |
+----+-------------+-------+-------+---------------+---------+---------+------+--------+-----------------------------------------------+
1 row in set (0.00 sec)

mysql> explain select * from t2  where uid > 1 order by uid limit 3;
+----+-------------+-------+-------+---------------+---------+---------+------+--------+------------------------------------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref  | rows   | Extra                              |
+----+-------------+-------+-------+---------------+---------+---------+------+--------+------------------------------------+
|  1 | SIMPLE      | t2    | range | idx_uid       | idx_uid | 9       | NULL | 523677 | Using index condition; Using where |
+----+-------------+-------+-------+---------------+---------+---------+------+--------+------------------------------------+
1 row in set (0.00 sec)
```

可以看到出错的语句，用到了 MRR（不了解 MRR 的可以看下我们之前的月报 [优化器 MRR & BKA](http://mysql.taobao.org/monthly/2016/01/04/)），这是优化器在走二级索引时，为了减少回表的磁盘 IO 的一个优化。

把这个优化关掉呢？

```plain
set optimizer_switch='mrr=off';
mysql> explain select id from t2  where uid > 1 limit 3;
+----+-------------+-------+-------+---------------+---------+---------+------+--------+--------------------------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref  | rows   | Extra                    |
+----+-------------+-------+-------+---------------+---------+---------+------+--------+--------------------------+
|  1 | SIMPLE      | t2    | range | idx_uid       | idx_uid | 9       | NULL | 523677 | Using where; Using index |
+----+-------------+-------+-------+---------------+---------+---------+------+--------+--------------------------+
1 row in set (0.00 sec)

select * from t2  where uid > 1 limit 3;
xxx
xxx
xxx
3 rows in set (0.00 sec)
```

可以看到，关掉优化器的 MRR 后，语句就返回正常了。因此基本可以判断是 MRR 导致的。

下面我们从源码层面分析下看，到底是怎么回事。

根据报错信息，来 gdb 跟踪，发现导致报错的栈是这样的，可以看到是在 mrr 执行初始化阶段：

```css
#0  DsMrr_impl::dsmrr_init()
#1  ha_tokudb::multi_range_read_init()
#2  QUICK_RANGE_SELECT::reset()
#3  join_init_read_record()
#4  sub_select()
#5  do_select()
#6  JOIN::exec()
#7  mysql_execute_select()
#8  mysql_select()
#9  handle_select()
#10 execute_sqlcom_select()
#11 mysql_execute_command()
...
```

具体在 DsMrr\_impl::dsmrr\_init 中的逻辑是这样的：

```php
// Transfer ICP from h to h2
if (mrr_keyno == h->pushed_idx_cond_keyno)
{
  if (h2->idx_cond_push(mrr_keyno, h->pushed_idx_cond))
  {
    retval= 1;
    goto error;
  }
}
```

我们对应看下 TokuDB 里条件下推接口实现：

```sql
// we cache the information so we can do filtering ourselves,
// but as far as MySQL knows, we are not doing any filtering,
// so if we happen to miss filtering a row that does not match
// idx_cond_arg, MySQL will catch it.
// This allows us the ability to deal with only index_next and index_prev,
// and not need to worry about other index_XXX functions
Item* ha_tokudb::idx_cond_push(uint keyno_arg, Item* idx_cond_arg) {
    toku_pushed_idx_cond_keyno = keyno_arg;
    toku_pushed_idx_cond = idx_cond_arg;
    return idx_cond_arg;
}
```

可以看到 `ha_tokudb::idx_cond_push` 是会将原条件在返回给 server 的。因此就导致了 `DsMrr_impl::dsmrr_init` 返回错误码 1 (Got error 1 from storage engine)。

而 `handler:idx_cond_push()` 接口是允许引擎层返回非 NULL 值的，引擎层认为自己没有完全过滤结果集，那么是可以返回条件给 server 层，让 server 层再做一次过滤的：

```php
/**
  Push down an index condition to the handler.

  The server will use this method to push down a condition it wants
  the handler to evaluate when retrieving records using a specified
  index. The pushed index condition will only refer to fields from
  this handler that is contained in the index (but it may also refer
  to fields in other handlers). Before the handler evaluates the
  condition it must read the content of the index entry into the
  record buffer.

  The handler is free to decide if and how much of the condition it
  will take responsibility for evaluating. Based on this evaluation
  it should return the part of the condition it will not evaluate.
  If it decides to evaluate the entire condition it should return
  NULL. If it decides not to evaluate any part of the condition it
  should return a pointer to the same condition as given as argument.

  @param keyno    the index number to evaluate the condition on
  @param idx_cond the condition to be evaluated by the handler

  @return The part of the pushed condition that the handler decides
          not to evaluate
 */

virtual Item *idx_cond_push(uint keyno, Item* idx_cond) { return idx_cond; }
```

因此这个问题是 MRR 在实现上的一个 bug，没有考虑引擎在ICP时返回非 NULL 的情况。

另外我们在查问题时发现，如果 mysqld 重启或者通过 flush table 关闭表的话，查询是不会出错的：

```plain
mysql> flush table t2;
mysql> explain  select * from t2  where uid > 1 limit 3;
+----+-------------+-------+-------+---------------+---------+---------+------+--------+------------------------------------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref  | rows   | Extra                              |
+----+-------------+-------+-------+---------------+---------+---------+------+--------+------------------------------------+
|  1 | SIMPLE      | t2    | range | idx_uid       | idx_uid | 9       | NULL | 523677 | Using index condition; Using where |
+----+-------------+-------+-------+---------------+---------+---------+------+--------+------------------------------------+
```

从 explain 结果看，是因为没有用到 MRR，这又是为什么呢？

我们看下优化器是如何选择是否用MRR优化的，在 `DsMrr_impl::choose_mrr_impl()` 这个函数里是有这样的逻辑的：

```plain
 /*
   If @@optimizer_switch has "mrr_cost_based" on, we should avoid
   using DS-MRR for queries where it is likely that the records are
   stored in memory. Since there is currently no way to determine
   this, we use a heuristic:
   a) if the storage engine has a memory buffer, DS-MRR is only
      considered if the table size is bigger than the buffer.
   b) if the storage engine does not have a memory buffer, DS-MRR is
      only considered if the table size is bigger than 100MB.
   c) Since there is an initial setup cost of DS-MRR, so it is only
      considered if at least 50 records will be read.
 */
 if (thd->optimizer_switch_flag(OPTIMIZER_SWITCH_MRR_COST_BASED))
 {
   /*
     If the storage engine has a database buffer we use this as the
     minimum size the table should have before considering DS-MRR.
   */
   longlong min_file_size= table->file->get_memory_buffer_size();
   if (min_file_size == -1)
   {
     // No estimate for database buffer
     min_file_size= 100 * 1024 * 1024;    // 100 MB
   }

   if (table->file->stats.data_file_length <
       static_cast<ulonglong>(min_file_size) ||
       rows <= 50)
     return true;                 // Use the default implementation
 }
```

可以看到，MRR 选择条件是这样的：

1.  如果引擎的 cache 比表大的话，是不会用 MRR 优化的；
2.  如果引擎没有 cache，默认用 100M，用于自己不管理 cache 引擎，如 MyISAM；
3.  如果要查询的行数不超过50的话，也是不会用 MRR 优化的；

这个 cache 对 InnoDB 来说，就是 `innodb_buffer_pool_size`；对 TokuDB 来说，就是 `tokudb_cache_size`。但是 TokuDB handler 层没有实现 `get_memory_buffer_size()` 这个接口，导致一直用 100M 做为 cache 来判断，这个是 TokuDB handler 实现的上的一个bug。

而 `data_file_length` 这个是值是内存信息，在表刚关闭重新打开的时候，是0，所以不会用MRR优化。

另外还有一个判断条件时，如果要求排序的话，也是不会用 MRR 优化的，这也就是为什么我们刚开始发现的，语句中用了 order by 后，explain 结果中就没有 MRR了。

## 问题影响和解决

从上面的分析来看，满足下面条件语句会被影响：

1.  语句访问的是 TokuDB 表，并且走的二级索引，有回表操作；
2.  表大小超过 100M；

简单的判断方法是，explain 结果中有 `Using index condition; Using where; Using MRR`，并且语句报错 Got error 1 from storage engine。

临时的解决方法是关闭优化器的 MRR 或者 ICP：

```sql
set optimizer_switch='mrr=off';
or
set optimizer_switch='index_condition_pushdown=off';
```

[阿里云RDS-数据库内核组](http://mysql.taobao.org/)  
[欢迎在github上star AliSQL](https://github.com/alibaba/AliSQL)  
阅读： -  
[![知识共享许可协议](assets/1590646708-8232d49bd3e964f917fa8f469ae7c52a.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)  
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

---------------------------------------------------


原网址: [访问](http://mysql.taobao.org/monthly/2017/04/08/)

创建于: 2020-05-28 14:18:28

目录: default

标签: `mysql.taobao.org`

