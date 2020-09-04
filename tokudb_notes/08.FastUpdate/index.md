---
title: TokuDB · 引擎特性 · FAST UPDATES
category: default
tags: 
  - mysql.taobao.org
created_at: 2020-04-29 11:19:32
original_url: http://mysql.taobao.org/monthly/2014/11/09/
---

[

# 数据库内核月报 － 2014 / 11

](http://mysql.taobao.org/monthly/2014/11)

[‹](http://mysql.taobao.org/monthly/2014/11/08/)

[›](http://mysql.taobao.org/monthly/2014/11/10/)

*   [当期文章](#)

## TokuDB · 引擎特性 · FAST UPDATES

MySQL的update在执行时需要做read-modify-write：

1.  从底层存储中读取row数据(read row)
2.  对row数据做更改(modify row)
3.  把更改后的数据写回底层存储(write row)

操作路径还是比较长的，TokuDB提供了fast update语法，让＂某些＂场景下update更快，无需做read和modify直接write。 用法:

```sql
CREATE TABLE `t1` (
  `id` int(11) NOT NULL,
 `count` bigint(20) NOT NULL,
  PRIMARY KEY (`id`)) ENGINE=TokuDB;
```

NOAR语句：

```sql
INSERT NOAR INTO t1 VALUES (1,0) ON DUPLICATE KEY UPDATE count = count + 1;
```

语义是：插入一条记录，如果该记录存在(id为1)，就对count的值做加法操作，不存在则做插入。 注意： fast updates的条件是比较苛刻的，必须满足：

1.  表必须有主键，且只能有一个索引(包含主键)
2.  主键必须为： int, char或varchar类型，被更新的列也必须为三种类型之一
3.  WHERE子句必须为单行操作
4.  如果开启binlog，binlog_format必须为STATEMENT 看了这些苛刻的条件后，有种＂臣妾做不到＂的感觉了吧，可以看出TokuDB一直为细节而努力。

[阿里云RDS-数据库内核组](http://mysql.taobao.org/)  
[欢迎在github上star AliSQL](https://github.com/alibaba/AliSQL)  
阅读： -  
[![知识共享许可协议](assets/1588130372-8232d49bd3e964f917fa8f469ae7c52a.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)  
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

---------------------------------------------------


原网址: [访问](http://mysql.taobao.org/monthly/2014/11/09/)

创建于: 2020-04-29 11:19:32

目录: default

标签: `mysql.taobao.org`

