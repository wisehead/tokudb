---
title: TokuDB· 引擎特性·压缩 · 数据库内核月报 · 看云
category: default
tags: 
  - www.kancloud.cn
created_at: 2020-04-29 11:03:55
original_url: https://www.kancloud.cn/taobaomysql/monthly/67136
---

# TokuDB· 引擎特性·压缩

TokuDB除了有着较好的写性能外，还有一个重要的特性：压缩，而且大部分情况下压缩效果很不错。

目前TokuDB有４种压缩算法：

```plain
tokudb_lzma:    压缩比高，CPU消耗也高
tokudb_zlib:    压缩比和CPU消耗持中(默认压缩算法)
tokudb_quicklz:    压缩比低，CPU消耗低
tokudb_uncompressed:无压缩
```

使用上也比较简单：

```plain
MariaDB:
CREATE TABLE `t_quicklz` (
  `x` int(11) DEFAULT NULL,
  `y` int(11) DEFAULT NULL
) ENGINE=TokuDB `compression`='tokudb_quicklz'
```

```plain
MySQL(Percona Server):
CREATE TABLE `t_quicklz` (
  `x` int(11) DEFAULT NULL,
  `y` int(11) DEFAULT NULL
) ENGINE=TokuDB DEFAULT CHARSET=latin1 ROW_FORMAT=TOKUDB_QUICKLZ
```

可能大家会有一个疑问：如果一个表创建的时候压缩算法是tokudb_quicklz，我可以通过ALERT TABLE改成其他算法吗？答案是：可以的！

TokuDB在底层实现上，用1byte来标记当前block的压缩算法，并持久化到磁盘，当压缩算法改变后，从磁盘读取数据然后解压缩的代码类似：

```plain
 switch (block_buffer[0] & 0xF) {
    case TOKU_NO_COMPRESSION:
        ...
        break;

    case TOKU_ZLIB_METHOD:
        ...
        break;

    case TOKU_QUICKLZ_METHOD:
        ...
        break;
    case TOKU_LZMA_METHOD:
        ...
        break;
｝
```

是不是很机智？

在使用TokuDB的过程中，一般不会改变压缩算法，除非默认的tokudb_zlib不给力，TokuDB是大block(4MB)，压缩效果较好，enjoy~

上一篇： [TokuDB· 主备复制·Read Free Replication](https://www.kancloud.cn/taobaomysql/monthly/213826)下一篇：[数据库内核月报 － 2014/09](https://www.kancloud.cn/taobaomysql/monthly/67018)

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/67136)

创建于: 2020-04-29 11:03:55

目录: default

标签: `www.kancloud.cn`

