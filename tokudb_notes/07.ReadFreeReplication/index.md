---
title: MySQL内核月报 2014.10-TokuDB·　主备复制·Read Free Replication-云栖社区-阿里云
category: default
tags: 
  - yq.aliyun.com
created_at: 2020-04-29 11:11:37
original_url: https://yq.aliyun.com/articles/50737?scm=20140722.184.2.173
---


# MySQL内核月报 2014.10-TokuDB·　主备复制·Read Free Replication-云栖社区-阿里云

## MySQL内核月报 2014.10-TokuDB·　主备复制·Read Free Replication


尽管MySQL 5.6和MariaDB 10.x在replication上已经做了不少优化，TokuDB 7.5也做了一个＂进一步＂的优化：Read Free Replication(RFR)，目的是提高备库(slave)重放速度，减少主备延迟。

RFR的原理比较简单：就是避免一些＂不必要＂的读来减少read IO。

**Read IO**

当在slave上执行：UPDATE/DELETE操作的时候，MySQL进行read-modify-write操作，可能会产生read IO操作，而INSERT的时候如果需要做＂唯一性＂检查，也可能会产生read IO。

什么条件下，TokuDB才可以read free呢？

1.  主库配置必须BINLOG_FORMAT=ROW
2.  主库上的操作不能违反唯一性约束（比如设置了＂unique_checks=OFF＂，否则主备同步会停止），这样到TokuDB备库的所有log event都默认不需要再做＂unique check＂
3.  备库只读

同时满足以上3个条件，TokuDB的备库认为从主库过来的INSERT/UPDATE/DELETE log event均是安全的，可以不做read，直接做write操作。

从[TokuDB官方测试](https://yq.aliyun.com/go/articleRenderRedirect?url=http%3A%2F%2Fwww.tokutek.com%2F2014%2F09%2Ftokudb-v7-5-read-free-replication-the-benchmark%2F)看效果还是不错的，主库~140TPS，备库~70TPS，当开启RFR的时候，备库可以飚到1400TPS，随后与主库基本持平。

进一步的优化还在继续，希望＂妈妈＂(DBA)再也不用担心我(read only slave)的性能了。

[![Master-vs-slave-cps.png](assets/1588129897-0ea26d54f5882bdddc3d282643d4b876.png)](https://yq.aliyun.com/go/articleRenderRedirect?url=http%3A%2F%2Fmysql.taobao.org%2Findex.php%3Ftitle%3D%25E6%2596%2587%25E4%25BB%25B6%3AMaster-vs-slave-cps.png)

  

本文为云栖社区原创内容，未经允许不得转载，如需转载请发送邮件至yqeditor@list.alibaba-inc.com；如果您发现本社区中有涉嫌抄袭的内容，欢迎发送邮件至：yqgroup@service.aliyun.com 进行举报，并提供相关证据，一经查实，本社区将立刻删除涉嫌侵权内容。



