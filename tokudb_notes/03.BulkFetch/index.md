---
title: TokuDB· 性能优化·Bulk Fetch · 数据库内核月报 · 看云
category: Database
tags: 
  - www.kancloud.cn
created_at: 2020-04-29 10:48:31
original_url: https://www.kancloud.cn/taobaomysql/monthly/67114
---

# TokuDB· 性能优化·Bulk Fetch

Bulk Fetch是为了提升区间操作性能的，聊它之前，先简单唠叨下读取机制，TokuDB由两部分组成: [tokuFT](https://github.com/Tokutek/ft-index)和 [tokudb-engine](https://github.com/Tokutek/tokudb-engine) 。   
tokuFT是个支持事务的key/value存储层，tokudb-engine是MySQL API对接层，调用关系为:tokudb-engine ->tokuFT。   
tokuFT里的一个value，在tokudb-engine里就是一条row数据，底层存储与上层调用解耦，是个很棒的设计。   
在tokuFT是个key里，索引的每个node都是大块头(4MB)，node又细分为多个＂小块＂(internal node的叫做partition，leaf node的叫做basement)。   
从磁盘读取数据到内存的方式有２种：

1.  仅读一个＂小块＂的数据，反序列化到内存（提升point query性能，只读取需要的那部分数据即可)
    
2.  读取整个node数据，反序列化到内存（提升区间性能，一次读取整个node磁盘数据）
    

对于tokudb-engine层的区间操作（比如get_next等），tokuFT这层是无状态的，必须告诉当前的key，然后给你查找next，流程大体是:

```plain
 tokudb-engine::get_next(current_key) --> tokuFT::search_next(current_key) --> tokuFT::return next
```

这样，即使tokuFT缓存了整个node数据，tokudb-engine还是遍历着跟tokuFT要一遍：tokuFT每次都要根据当前key，多次调用compare操作最终查出next，路径太长了！   
有什么办法优化呢？这就是Bulk Fetch的威力: tokudb-engine向tokuFT一次要回整个node的数据，自己解析出next row数据，tokuFT的调用就省了:

```plain
 tokudb-engine::get_next(current_key) --> tokudb-engine::parse_next
```

从Tokutek的测试看，在使用Bulk Fetch后，能有2x-5x的性能提升。   
但并不是所有的区间操作都可以Bulk Fetch的(比如涉及update/delete)，TokuDB目前实现了:SELECT、CREATE\_TABLE、INSERT\_SELECT和REPLACE_SELECT的Bulk Fetch功能，预计发布在7.1.8版，更多Bulk Fetch介绍：   
[https://github.com/Tokutek/tokudb-engine/wiki/Bulk-Fetch](https://github.com/Tokutek/tokudb-engine/wiki/Bulk-Fetch)

上一篇： [MariaDB·分支特性·FusionIO特性支持](https://www.kancloud.cn/taobaomysql/monthly/67113)下一篇：[TokuDB· 数据结构·Fractal-Trees与LSM-Trees对比](https://www.kancloud.cn/taobaomysql/monthly/67115)

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/67114)

创建于: 2020-04-29 10:48:31

目录: Database

标签: `www.kancloud.cn`

