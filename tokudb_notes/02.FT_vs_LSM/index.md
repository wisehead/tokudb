---
title: TokuDB· 数据结构·Fractal-Trees与LSM-Trees对比 · 数据库内核月报 · 看云
category: Database
tags: 
  - www.kancloud.cn
created_at: 2020-04-29 10:10:46
original_url: https://www.kancloud.cn/taobaomysql/monthly/67115
---

# TokuDB· 数据结构·Fractal-Trees与LSM-Trees对比

最近，TokuDB的创始人Dr. Bradley Kuzmaul发表了一篇文章: [A Comparison of Log-Structured Merge (LSM) and Fractal Tree Indexing](http://forms.tokutek.com/acton/attachment/6118/f-0039/1/-/-/-/-/lsm-vs-fractal.pdf)，从write amplification(WAMP), read amplification(RAMP), and space amplification三个方面对B-Trees，LSM-Trees(LSM)以及Fractal-Trees(FT)进行了详细的分析和对比。

Dr. Bradley Kuzmaul的结果是(页13):   
![Lsmft.png](assets/1588126246-a783a42acbb2f50bc0c48da0329517e0.png)

从结果来看：

```plain
在WAMP上，FT跟LSM(leveled)是相同的
在RAMP(range)上，LSM(leveled)的复杂度明显要高不少(FT的O(logN/B)倍)
```

不过，RAMP这块的分析有个小问题：   
LSM(leveled)在实现上(比如LevelDB)，可以通过meta-info打＂锚点＂的方式，把RAMP(range)降低甚至做到跟FT一样，如果是point queries的RAMP，则可以通过Bloom filter来降低。

具体的推导过程请阅读原作，下面简单分析下FT的RAMP为啥比LSM的要低。   
FT的读方式比较＂特殊＂，由于每个节点都有个message buffer，当有读请求时，需要把inner node的message buffer数据（部分）推(apply)到leaf node，最后只在leaf node上做二分查找，所以RAMP基本就是树的高度。

另外，在数据流向上(compaction过程中数据走向)，LSM强调"level"(横向)，从level-L根据规则选取部分数据merge到level-(L+1)，如果选取数据的策略不好，会抢占磁盘带宽，容易引起性能抖动，而FT强调"root-to-leaf"(纵向)，数据从root有序的逐层merge到leaf节点，每条数据的merge路径是很明确的。


