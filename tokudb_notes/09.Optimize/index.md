---
title: TokuDB · 版本优化 · 7.5.0
category: default
tags: 
  - mysql.taobao.org
created_at: 2020-04-29 11:33:42
original_url: http://mysql.taobao.org/monthly/2014/11/08/
---

[

# 数据库内核月报 － 2014 / 11

](http://mysql.taobao.org/monthly/2014/11)

[‹](http://mysql.taobao.org/monthly/2014/11/07/)

[›](http://mysql.taobao.org/monthly/2014/11/09/)

*   [当期文章](#)

## TokuDB · 版本优化 · 7.5.0

TokuDB　7.5.0大版本已发布，是一个里程碑的版本，这里谈几点优化，以飨存储引擎爱好者们。

a) shutdown加速

有用户反馈TokuDB在shutdown的时候，半个小时还没完事，非常不可接受。 在shutdown的时候，TokuDB在干什么呢？在做checkpoint，把内存中的节点数据序列化并压缩到磁盘。 那为什么如此耗时呢？如果tokudb\_cache\_size开的比较大，内存中的节点会非常多，在shutdown的时候，大家都排队等着被压缩到磁盘（串行的）。 在7.5.0版本，TokuDB官方针对此问题进行了优化，使多个节点并行压缩来缩短时间。

详细commit见: [https://github.com/Tokutek/ft-index/commit/bd85b8ce7d152412755860976a871fdfc977115c](https://github.com/Tokutek/ft-index/commit/bd85b8ce7d152412755860976a871fdfc977115c) BTW: TokuDB在早期设计的时候已保留并行接口，只是一直未开启。

b) 内节点读取加速

在内存中，TokuDB内节点(internal node)的每个message buffer都有２个重要数据结构：

1) FIFO结构，保存{key, value} 2) OMT结构，保存{key, FIFO-offset} 由于FIFO不具备快速查找特性，就利用OMT来做快速查找(根据key查到value)。

这样，当内节点发生cache miss的时候，索引层需要做：

1) 从磁盘读取节点内容到内存 2) 构造FIFO结构 3) 根据FIFO构造OMT结构(做排序) 由于TokuDB内部有不少性能探(ji)针(shu)，他们发现步骤3)是个不小的性能消耗点，因为每次都要把message buffer做下排序构造出OMT，于是在7.5.0版本，把OMT的FIFO-offset(已排序)也持久化到磁盘，这样排序的损耗就没了。

c) 顺序写加速

当写发生的时候，会根据当前的key在pivots里查找(二分)当前写要落入哪个mesage buffer，如果写是顺序(或局部顺序，数据走向为最右边路径)的，就可以避免由＂查找＂带来的额外开销。 Fractal tree.png 如何判断是顺序写呢？TokuDB使用了一种简单的启发式方法(heurstic)：seqinsert_score积分式。 如果：

1) 当前写入落入最右节点，对seqinsert\_score加一分(原子) 2) 当前写入落入非最右节点，对seqinsert\_score清零(原子) 当seqinsert_score大于100的时候，就可以认为是顺序写，当下次写操作发生时，首先与最右的节点pivot进行对比判断，如果确实为顺序写，则会被写到该节点，省去不少compare开销。 方法简单而有效。

[阿里云RDS-数据库内核组](http://mysql.taobao.org/)  
[欢迎在github上star AliSQL](https://github.com/alibaba/AliSQL)  
阅读： -  
[![知识共享许可协议](assets/1588131222-8232d49bd3e964f917fa8f469ae7c52a.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)  
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

---------------------------------------------------


原网址: [访问](http://mysql.taobao.org/monthly/2014/11/08/)

创建于: 2020-04-29 11:33:42

目录: default

标签: `mysql.taobao.org`

