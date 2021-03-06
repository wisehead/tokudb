---
title: MySQL · TokuDB · 疯狂的 filenum++ · 数据库内核月报 · 看云
category: default
tags: 
  - www.kancloud.cn
created_at: 2020-05-12 18:40:39
original_url: https://www.kancloud.cn/taobaomysql/monthly/67077
---

# MySQL · TokuDB · 疯狂的 filenum++

## 问题描述

收到一枚RDS TokuDB实例crash导致HA切换的报警，上去一看错误如下：

```plain
tokudb/ft-index/ft/cachetable/cachetable.cc toku_cachetable_openfd_with_filenum: Assertion `filenum.fileid != FILENUM_NONE.fileid' failed
/bin/mysqld(_Z19db_env_do_backtraceP8_IO_FILE+0x1b)[0xc57ddb]
/bin/mysqld(_Z35toku_cachetable_openfd_with_filenumPP9cachefileP10cachetableiPKc7FILENUMPb+0x223)[0xbb49b3]
/bin/mysqld(_Z19toku_ft_handle_openP9ft_handlePKciiP10cachetableP7tokutxn+0x135)[0xbf3c05]
/bin/mysqld(_Z20toku_ft_handle_clonePP9ft_handleS0_P7tokutxn+0xb5)[0xbf42f5]
/bin/mysqld(_Z29toku_db_lt_on_create_callbackPN4toku8locktreeEPv+0x2a)[0xb801ba]
/bin/mysqld(_Z18toku_db_open_inameP9__toku_dbP13__toku_db_txnPKcji+0x276)[0xb805b6]
/bin/mysqld(_ZN9ha_tokudb20open_main_dictionaryEPKcbP13__toku_db_txn+0x1ab)[0xb50a0b]
/bin/mysqld(_ZN9ha_tokudb16initialize_shareEPKci+0x2c8)[0xb70848]
/bin/mysqld(_ZN9ha_tokudb4openEPKcij+0x5e9)[0xb71349]
/bin/mysqld(_ZN7handler7ha_openEP5TABLEPKcii+0x33)[0x5e74b3]
```

这个错误信息在RDS上第一次碰到，隐隐感到这是一个“可遇不可求”的bug导致，开始捉虫。

## 问题分析

每个表(索引)文件被打开后，TokuDB都会为这个文件赋予一个唯一id，即filenum。

filenum有什么作用？  
TokuDB在写redo log的时候，每个事务里会带一个filenum属性，用来标示该事务属于哪个表文件，在崩溃恢复的时候，会根据这个filenum回放到相应的表里。

filenum在什么时候被分配？  
表(索引)文件被打开的时候会被分配。

filenum如何分配？  
为了保证唯一性，TokuDB维护了一个filenum数据结构(类似binary tree) : m\_active\_filenum，分配算法：

```plain
uint32_t m_next_filenum_to_use;  //全局变量，用来标识已分配的最大filenum
lock();
retry:
int ret = m_active_filenum.find(m_next_filenum_to_use);
if (ret == 0) {
  //m_next_filenum_to_use被占用
  m_next_filenum_to_use++;
  goto retry;
}
filenum = m_next_filenum_to_use; //得到我们想要的filenum
m_next_filenum_to_use++;
unlock();
```

这样问题就来了，如果用户有非常多的表(索引)文件，不停的被打开和关闭，m\_next\_filenum\_to\_use会一直递增下去，由于是uint32\_t类型，小宇宙终于爆发了，filenum 递增到4294967295(UINT\_MAX)，从而导致assert失败。

## 问题修复

当一些表(索引)文件被close后，这些filenum可以被回收再利用，所以当filenum递增到UINT_MAX后，重置到0即可:

```plain
uint32_t m_next_filenum_to_use;  //全局变量，用来标识已分配的最大filenum
lock();
retry:
int ret = m_active_filenum.find(m_next_filenum_to_use);
if (ret == 0) {
  //m_next_filenum_to_use被占用
  m_next_filenum_to_use++;
  goto retry;
}
// 从0开始重新获取未被使用的filenum
if (m_next_filenum_to_use == UINT_MAX) {
  m_next_filenum_to_use = 0;
  goto retry;
}
filenum = m_next_filenum_to_use; //得到我们想要的filenum
m_next_filenum_to_use++;
unlock();
```

RDS版本已修复此问题，官方patch戳[这里](https://github.com/percona/PerconaFT/commit/1b289f788c6b50956321f4d977ac1cb63bd72013)。

上一篇：[MySQL · 答疑解惑 · open file limits](https://www.kancloud.cn/taobaomysql/monthly/67076)下一篇：[MySQL · 功能分析 · 5.6 并行复制实现分析](https://www.kancloud.cn/taobaomysql/monthly/67078)

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/67077)

创建于: 2020-05-12 18:40:39

目录: default

标签: `www.kancloud.cn`

