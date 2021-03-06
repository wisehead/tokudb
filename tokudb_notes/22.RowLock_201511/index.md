---
title: MySQL · TokuDB · TokuDB 中的行锁 · 数据库内核月报 · 看云
category: default
tags: 
  - www.kancloud.cn
created_at: 2020-05-27 13:36:28
original_url: https://www.kancloud.cn/taobaomysql/monthly/81382
---


# MySQL · TokuDB · TokuDB 中的行锁 · 数据库内核月报 · 看云

## 前言

4月份月报有篇文章《[行锁(row-lock)与区间锁(range-lock)](http://mysql.taobao.org/monthly/2015/04/03/)》，介绍了 TokuDB 的行锁/区间锁是如何使用的。这篇文章是其姐妹篇，介绍TokuDB行锁的实现，大家可以对照着看。

## 行锁申请

与 InnoDB 类似，TokuDB 也支持行级锁用来协调多个 txn 对数据库表某一段数据的并发访问。一个表中所有已经 grant 的行锁是用一个 binary search tree 来表示的，TokuDB 的术语称它为 lock tree。lock tree 与数据库表之间是一一对应的关系。当打开 cursor 读写 TokuDB 数据库的时候，需要首先尝试申请row lock，成功后再调用 `db_put`/`ha_tokudb::read_range_first` 方法来读写数据。TokuDB 的 row lock 是用 range lock 表示的，一个 range lock 代表按key值连续的一个行锁区间。rangelock是一个同步锁，如果获取成功立即返回；若失败则会同步等待若干时间，等待超时整个操作就会失败返回。range lock的申请分为三个阶段，下面将逐个说明。

1.  创建range lock：在TokuDB中就是创建一个 lock\_request 对象  
    创建的过程很简单，主要是初始化，创建成功后调用 lock\_request 的 start 函数来申请锁；
    
2.  申请range lock：大部分事情都在这个阶段完成的  
    申请锁的时候需要指定五元组（lt，txn，type，left\_key，right\_key），分别表示数据库表对应的lock tree，txn结构，锁类型（read/write)，键值区间（left\_key, right\_key)。申请的时候会根据锁 类型来调用 locktree 的 `acquire_write`/`read_lock`方法来获取锁。这里有个 tricky 的地方：在 locktree 实现中隐式地将 read lock 升级成 write lock。  
    在获取锁的函数里面，首先调用了 `check_current_lock_constraints` 来进行 throttle 控制当前锁占用的内存，这块不展开讨论了。  
    locktree 有一个为 single txn 做的优化，当系统猜测当前是工作在 single txn 的方式下（不存在锁竞争的问题），所有的锁都会被 grant 并记录在 sto\_buffer 里面。如果不是 single txn 的模式，已经 grant 的锁则保存在 concurrent\_tree 里面，这个就是我们在前面提到的那个 binary search tree。Single txn 模式的判断是用启发式的方法，由两个因素控制 sto\_buffer 和 concurrent\_tree 的切换: 积分 score 和 sto\_buffer 长度，因篇幅有限这块也留给大家分析了。要提的一点是如果正处在 single txn 模式，遇到了一个新的 txn，那么 sto\_buffer 的锁会被转移到 concurrent\_tree 上。  
    我们重点讨论是 concurrent\_tree 的情况。函数 `acquire_lock_consolidated` 会根据五元组里面的 left\_key 和 right\_key 构造一个 request\_range，然后用这个 range 在 concurrent\_tree 上定位到与它存在 overlap 关系的最小子树，并把子树里面与 request\_range 存在 overlapped 关系的那些锁保存在一个变长数组里面。然后 iterate 这个数组看看是否存在锁冲突，冲突的条件是与五元组里的 txnid 不同但锁区间是 overlapped 的。如果不存在锁冲突，就可以立即 grant 这个锁申请了。  
    剩下的是些维护工作，就是依次把区间重叠的已经 grant 的锁和我们正在申请的锁进行区间 merge，保证 concurrent\_tree 里面的所有锁的区间都是不相交的（不overlapped的）。如果不幸，申请的锁和concurrent\_tree里面的锁有冲突，那么请求操作会失败。
    
3.  等待range lock：申请失败会把这个 range lock 放到 locktree 的 pending list 里等待以后重试。  
    锁申请失败可能是发生了死锁，还需要调用 deadlock\_exists 递归构造锁等待 DAG 图判断是否真的发生了死锁。
    

## 举例说明

上面描述比较枯燥，我们结合4月份月报里的例子一起看看吧。

```plain
mysql> show create table t\G
--------------------------- 1. row ---------------------------
       Table: t
Create Table: CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=TokuDB DEFAULT CHARSET=latin1

mysql> set autocommit=off;
mysql> insert into t values (1),(10),(100);
mysql> select * from information_schema.tokudb_locks\G
--------------------------- 1. row ---------------------------
               locks_trx_id: 238
      locks_mysql_thread_id: 3
              locks_dname: ./test/t-main
             locks_key_left: 0001000000
            locks_key_right: 0001000000
        locks_table_schema: test
          locks_table_name: t
locks_table_dictionary_name: main
--------------------------- 2. row ---------------------------
               locks_trx_id: 238
      locks_mysql_thread_id: 3
             locks_dname: ./test/t-main
             locks_key_left: 000a000000
            locks_key_right: 000a000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
--------------------------- 3. row ---------------------------
               locks_trx_id: 238
      locks_mysql_thread_id: 3
              locks_dname: ./test/t-main
             locks_key_left: 0064000000
            locks_key_right: 0064000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
```

再看一个例子。id 是 primary key，c1 上有 index 。Insert 三条记录产生6个行锁，3个在primary key上，3个在c1 index上。primary key上锁的key值主要由pk值构成，非 unique 的 index 锁的 key 值主要由 index 上的 key 值和 primary key 值组成。

```plain
Create Table: CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c1` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c1` (`c1`)
) ENGINE=TokuDB DEFAULT CHARSET=latin1
1 row in set (0.00 sec)

mysql> alter table t2 add index (c1);
mysql> set autocommit=off;
mysql> insert into t values(1,2),(10,11),(100,101);
mysql> select * from information_schema.tokudb_locks\G
--------------------------- 1. row ---------------------------
               locks_trx_id: 451
      locks_mysql_thread_id: 1
             locks_dname: ./test/t-main
             locks_key_left: 0001000000
            locks_key_right: 0001000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
--------------------------- 2. row ---------------------------
               locks_trx_id: 451
      locks_mysql_thread_id: 1
             locks_dname: ./test/t-main
             locks_key_left: 000a000000
            locks_key_right: 000a000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
--------------------------- 3. row ---------------------------
               locks_trx_id: 451
      locks_mysql_thread_id: 1
             locks_dname: ./test/t-main
             locks_key_left: 0064000000
            locks_key_right: 0064000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: main
--------------------------- 4. row ---------------------------
               locks_trx_id: 451
      locks_mysql_thread_id: 1
             locks_dname: ./test/t-key-c1
             locks_key_left: 00010200000001000000
            locks_key_right: 00010200000001000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: key-c1
--------------------------- 5. row ---------------------------
               locks_trx_id: 451
      locks_mysql_thread_id: 1
             locks_dname: ./test/t-key-c1
             locks_key_left: 00010b0000000a000000
            locks_key_right: 00010b0000000a000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: key-c1
--------------------------- 6. row ---------------------------
               locks_trx_id: 451
      locks_mysql_thread_id: 1
             locks_dname: ./test/t-key-c1
             locks_key_left: 00016500000064000000
            locks_key_right: 00016500000064000000
         locks_table_schema: test
           locks_table_name: t
locks_table_dictionary_name: key-c1
```

## 问题探讨

前面描述中提到 locktree 会自动升级读锁为写锁，这会不会带来性能问题呢？我看看下面的例子，假设场景是 isolation 级别是 read commit，关闭autocommit。

例1：

*   txn1: select 执行 index range scan ==> 在 `read_range_first` 之前会尝试获取读锁 ==> locktree 自动把读锁升级为写锁
*   txn2: select 执行 index range scan，与 txn1 相同的 index，数据有 overlapped ==> 在 `read_range_first` 之前会尝试获取读锁 ==> locktree 自动把读锁升级为写锁

例2:

*   txn3: select 执行 index range scan ==> 在 `read_range_first` 之前会尝试获取读锁 ==> locktree 自动把读锁升级为写锁
*   txn4: insert 插入的数据落在 txn3 操作的区间内 ==> 在 `db_put` 之前会尝试获取写锁 ==> locktree 获取写锁

这样看起来 txn1 与 txn2，txn3 与 txn4 申请的 rangelock 存在 overlapped 关系，可能引起等待。但事实上，在 read commit 的隔离级别上，txn1&txn2，tx3&txn4 是不需要等待的。

TokuDB 中读数据申请 row lock 是在 `c_set_bounds` 函数实现的。`c_set_bounds` 有个tricky的处理：对于 READ\_UNCOMMITTED，READ\_COMMITTED 和 REPEATABLE\_READ 隔离级别下并且没有设置 DB\_RMW 标志的话，读数据是不需要去拿 range lock 的。

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/81382)

创建于: 2020-05-27 13:36:28

目录: default

标签: `www.kancloud.cn`

