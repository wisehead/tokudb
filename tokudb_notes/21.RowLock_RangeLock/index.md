---
title: TokuDB · 特性分析 · 行锁(row-lock)与区间锁(range-lock)
category: default
tags: 
  - mysql.taobao.org
created_at: 2020-05-12 19:04:59
original_url: http://mysql.taobao.org/monthly/2015/04/03/
---

[

# 数据库内核月报 － 2015 / 04

](http://mysql.taobao.org/monthly/2015/04)

[‹](http://mysql.taobao.org/monthly/2015/04/02/)

[›](http://mysql.taobao.org/monthly/2015/04/04/)

*   [当期文章](#)

## TokuDB · 特性分析 · 行锁(row-lock)与区间锁(range-lock)

## 简介

TokuDB使用LockTree(ft-index/locktree)来维护事务的锁状态(row-lock和range-lock)，LockTree的数据结构是一个Binary Tree。  
本篇将通过几个“栗子”来谈谈TokuDB的row-lock和range-lock。  
表t:

```sql
mysql> show create table t\G
*************************** 1. row ***************************
       Table: t
Create Table: CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=TokuDB DEFAULT CHARSET=latin1
```

## row-lock

```sql
mysql> set autocommit=off;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t values (1),(10),(100);
Query OK, 3 rows affected (0.00 sec)
Records: 3  Duplicates: 0  Warnings: 0

mysql> select * from information_schema.tokudb_locks\G
*************************** 1. row ***************************
               locks_trx_id: 238
      locks_mysql_thread_id: 3
                locks_dname: ./test/t-main
             locks_key_left: 0001000000
            locks_key_right: 0001000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
*************************** 2. row ***************************
               locks_trx_id: 238
      locks_mysql_thread_id: 3
                locks_dname: ./test/t-main
             locks_key_left: 000a000000
            locks_key_right: 000a000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
*************************** 3. row ***************************
               locks_trx_id: 238
      locks_mysql_thread_id: 3
                locks_dname: ./test/t-main
             locks_key_left: 0064000000
            locks_key_right: 0064000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
3 rows in set (0.00 sec)

```

从tokudb\_locks表可以查询到，生成了3条row-lock(locks\_key\_left和locks\_key_right相等)。  
为了存储和显示方便，locks\_key\_left/locks\_key\_right取key的hash值。

## range-lock

```sql
mysql> set autocommit=off;
Query OK, 0 rows affected (0.00 sec)

mysql> delete from t where id<100;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from information_schema.tokudb_locks\G
*************************** 1. row ***************************
               locks_trx_id: 280
      locks_mysql_thread_id: 12
                locks_dname: ./test/t-main
             locks_key_left: -infinity
            locks_key_right: ff64000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
1 row in set (0.00 sec)
```

从tokudb_locks表可以查询到，where条件的rang-lock区间为\[-infinity, ff64000000\]，只要其他事务的锁区间跟这个有任何重叠，则需要等待。

## 锁冲突

client1执行如下操作:

```sql
mysql1> set autocommit=off;
Query OK, 0 rows affected (0.00 sec)

mysql1> insert into t values (1),(10),(100);
Query OK, 3 rows affected (0.00 sec)
Records: 3  Duplicates: 0  Warnings: 0

mysql1> select * from information_schema.tokudb_locks\G
*************************** 1. row ***************************
               locks_trx_id: 283
      locks_mysql_thread_id: 14
                locks_dname: ./test/t-main
             locks_key_left: 0001000000
            locks_key_right: 0001000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
*************************** 2. row ***************************
               locks_trx_id: 283
      locks_mysql_thread_id: 14
                locks_dname: ./test/t-main
             locks_key_left: 000a000000
            locks_key_right: 000a000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
*************************** 3. row ***************************
               locks_trx_id: 283
      locks_mysql_thread_id: 14
                locks_dname: ./test/t-main
             locks_key_left: 0064000000
            locks_key_right: 0064000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
3 rows in set (0.00 sec)
```

client2执行如下操作:

```sql
mysql2> set autocommit=off;
Query OK, 0 rows affected (0.00 sec)

mysql2> insert into t values (2),(100);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql2> select * from information_schema.tokudb_locks\G
*************************** 1. row ***************************
               locks_trx_id: 283
      locks_mysql_thread_id: 14
                locks_dname: ./test/t-main
             locks_key_left: 0001000000
            locks_key_right: 0001000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
*************************** 2. row ***************************
               locks_trx_id: 283
      locks_mysql_thread_id: 14
                locks_dname: ./test/t-main
             locks_key_left: 000a000000
            locks_key_right: 000a000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
*************************** 3. row ***************************
               locks_trx_id: 283
      locks_mysql_thread_id: 14
                locks_dname: ./test/t-main
             locks_key_left: 0064000000
            locks_key_right: 0064000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
*************************** 4. row ***************************
               locks_trx_id: 289
      locks_mysql_thread_id: 16
                locks_dname: ./test/t-main
             locks_key_left: 0002000000
            locks_key_right: 0002000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
4 rows in set (0.00 sec)

mysql2> select @@tokudb_last_lock_timeout;
+--------------------------------------------------------------------------------------------------------------------+
| @@tokudb_last_lock_timeout                                                                                         |
+--------------------------------------------------------------------------------------------------------------------+
| {"mysql_thread_id":16, "dbname":"./test/t-main", "requesting_txnid":289, "blocking_txnid":283, "key":"0064000000"} |
+--------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

锁等待超时了，通过参数tokudb\_last\_lock_timeout得知，hash为_0064000000_的row-lock已经被txnid为283（client1)抢占。

## 总结

在使用TokuDB过程中，如果`show processlist`里有锁等待语句，可以通过tokudb_locks表获取到当前所有事务的锁信息，以快速定位到问题。  
TokuDB提供tokudb\_lock\_timeout_debug参数，可以设置不同值(默认值为1)来记录锁冲突信息，说明如下：

```sql
tokudb_lock_timeout_debug = 0: No lock timeouts or lock deadlocks are reported.
tokudb_lock_timeout_debug = 1: A JSON document that describes the lock conflict is stored in the tokudb_last_lock_timeout session variable
tokudb_lock_timeout_debug = 2: A JSON document that describes the lock conflict is printed to the MySQL error log.
tokudb_lock_timeout_debug = 3: A JSON document that describes the lock conflict is stored in the tokudb_last_lock_timeout session variable and is printed to the MySQL error log. 
```

[阿里云RDS-数据库内核组](http://mysql.taobao.org/)  
[欢迎在github上star AliSQL](https://github.com/alibaba/AliSQL)  
阅读： -  
[![知识共享许可协议](assets/1589281499-8232d49bd3e964f917fa8f469ae7c52a.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)  
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

---------------------------------------------------


原网址: [访问](http://mysql.taobao.org/monthly/2015/04/03/)

创建于: 2020-05-12 19:04:59

目录: default

标签: `mysql.taobao.org`

