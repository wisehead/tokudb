---
title: TokuDB · 捉虫动态 · CREATE DATABASE 导致crash问题 · 数据库内核月报 · 看云
category: default
tags: 
  - www.kancloud.cn
created_at: 2020-05-12 18:54:08
original_url: https://www.kancloud.cn/taobaomysql/monthly/81114
---

# TokuDB · 捉虫动态 · CREATE DATABASE 导致crash问题

## 背景

TokuDB在启动的时候，会对datadir目录下的文件（和文件夹）进行遍历，根据规则找到所有的redo log文件，然后进行recover操作。  
redo文件的命名规则是:log\[index\].toku\[version\]，比如log000000000002.tokulog27。

## 问题

这个bug说起来有点“谜之”，因为它还跟编译器的版本有关。

如果使用gcc 4.8.2编译出一个TokuDB（加-O3编译参数），进行如下操作：

```plain
CREATE DATABASE log0002;
```

然后重启，整个TokuDB就可能凌乱了，出现如下crash信息：

```plain
storage/tokudb/ft-index/ft/logger/logfilemgr.cc:159 toku_logfilemgr_init: Assertion 'r==2' failed (errno=2)
Backtrace: (Note: toku_do_assert=0x0xcd50d0)
mysqld(_Z20toku_logfilemgr_initP15toku_logfilemgrPKcPm+0x26c)[0xc40cdc]
mysqld(_Z30toku_logger_open_with_last_xidPKcP10tokuloggerm+0x51)[0xc10ba1]
mysqld(_Z14tokuft_recover)
```

## 原因

先来看下TokuDB判断一个文件是否为redo log的代码：

```plain
static bool is_a_logfile_any_version (const char *name, uint64_t *number_result, uint32_t *version_of_log) {
	bool rval = true;
	uint64_t result;
	int n;
	int r;
	uint32_t version;
	r = sscanf(name, "log%" SCNu64 ".tokulog%" SCNu32 "%n", &result, &version, &n);
	if (r!=2 || name[n]!='\0' || version <= TOKU_LOG_VERSION_1) {
		//Version 1 does NOT append 'version' to end of '.tokulog'
		version = TOKU_LOG_VERSION_1;
		r = sscanf(name, "log%" SCNu64 ".tokulog%n", &result, &n);
		if (r!=1 || name[n]!='\0') {
			rval = false;
		}
	}
	if (rval) {
		*number_result  = result;
		*version_of_log = version;
	}

	return rval;
}
```

这段代码的逻辑很简单，调用sscanf函数获取相关参数, 高能区域为：

```plain
r = sscanf(name, "log%" SCNu64 ".tokulog%" SCNu32 "%n", &result, &version, &n);
```

如果name为log0002，调用完sscanf函数后，n和version的值是不确定的（因为函数内变量未被初始化），就可能导致函数 `is_a_logfile_any_version` 把log0002文件夹误认为是一个redo log。  
当`toku_logfilemgr_init`函数加载名称“log0002”时做二次校验，获取version值失败从而触发assert。

## 修复

大体思路是：

1.  由于redo log不可能是文件夹，所以增加只对文件进行判断的逻辑，遇到文件夹直接跳过；
2.  对`is_a_logfile_any_version`函数内变量进行初始化。

阿里云RDS TokuDB最新版本已经修复此问题，使用TokuDB的用户可以进行下小版本升级，以避免此问题影响业务。

上一篇：[PgSQL · 特性分析 · PostgreSQL Aurora方案与DEMO](https://www.kancloud.cn/taobaomysql/monthly/81113)下一篇：[PgSQL · 特性分析 · pg_receivexlog工具解析](https://www.kancloud.cn/taobaomysql/monthly/81115)

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/81114)

创建于: 2020-05-12 18:54:08

目录: default

标签: `www.kancloud.cn`

