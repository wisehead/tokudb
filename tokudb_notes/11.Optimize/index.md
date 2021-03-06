---
title: TokuDB·特性分析· Optimize Table · 数据库内核月报 · 看云
category: default
tags: 
  - www.kancloud.cn
created_at: 2020-04-29 21:54:09
original_url: https://www.kancloud.cn/taobaomysql/monthly/67166
---

# TokuDB·特性分析· Optimize Table

来自一个TokuDB用户的“投诉”:

[https://mariadb.atlassian.net/browse/MDEV-6207](https://mariadb.atlassian.net/browse/MDEV-6207)

现象大概是:

用户有一个MyISAM的表test_table:

```plain
 CREATE TABLE IF NOT EXISTS `test_table` (
   `id` int(10) unsigned NOT NULL,
   `pub_key` varchar(80) NOT NULL,
   PRIMARY KEY (`id`),
   KEY `pub_key` (`pub_key`)
 ) ENGINE=MyISAM DEFAULT CHARSET=latin1;
```

转成TokuDB引擎后表大小为92M左右:

```plain
 47M     _tester_testdb_sql_61e7_1812_main_ad88a6b_1_19_B_0.tokudb
 45M     _tester_testdb_sql_61e7_1812_key_pub_key_ad88a6b_1_19_B_1.tokudb
```

执行"OPTIMIZE TABLE test_table":

```plain
 63M     _tester_testdb_sql_61e7_1812_main_ad88a6b_1_19_B_0.tokudb
 61M     _tester_testdb_sql_61e7_1812_key_pub_key_ad88a6b_1_19_B_1.tokudb
```

再次执行"OPTIMIZE TABLE test_table":

```plain
 79M     _tester_testdb_sql_61e7_1812_main_ad88a6b_1_19_B_0.tokudb
 61M     _tester_testdb_sql_61e7_1812_key_pub_key_ad88a6b_1_19_B_1.tokudb
```

继续执行:

```plain
 79M     _tester_testdb_sql_61e7_1812_main_ad88a6b_1_19_B_0.tokudb
 61M     _tester_testdb_sql_61e7_1812_key_pub_key_ad88a6b_1_19_B_1.tokudb
```

基本稳定在这个大小。

主索引从47M-->63M-->79M，执行"OPTIMIZE TABLE"后为什么会越来越大？

这得从TokuDB的索引文件分配方式说起，当内存中的脏页需要写到磁盘时，TokuDB优先在文件末尾分配空间并写入，而不是“覆写”原块，原来的块暂时成了“碎片”。

这样问题就来了，索引文件岂不是越来越大？No, TokuDB会把这些“碎片”在checkpoint时加入到回收列表，以供后面的写操作使用，看似79M的文件其实还可以装不少数据呢！

嗯，这个现象解释通了，但还有2个问题:

1.  在执行这个语句的时候，TokuDB到底在做什么呢？

在做toku\_ft\_flush\_some\_child，把内节点的缓冲区(message buffer)数据刷到最底层的叶节点。

2.  在TokuDB里，OPTIMIZE TABLE有用吗？

作用非常小，不建议使用，TokuDB是一个"No Fragmentation"的引擎。

官方WIKI: [Optimize Table](https://github.com/Tokutek/tokudb-engine/wiki/Optimize-Table)

上一篇： [MySQL · 捉虫动态· replicate filter 和 GTID 一起使用的问题](https://www.kancloud.cn/taobaomysql/monthly/67165)下一篇：[数据库内核月报 － 2014/12](https://www.kancloud.cn/taobaomysql/monthly/67021)

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/67166)

创建于: 2020-04-29 21:54:09

目录: default

标签: `www.kancloud.cn`

