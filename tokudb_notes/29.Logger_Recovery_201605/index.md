---
title: MySQL · TokuDB · 日志子系统和崩溃恢复过程 · 数据库内核月报 · 看云
category: default
tags: 
  - www.kancloud.cn
created_at: 2020-05-27 19:55:00
original_url: https://www.kancloud.cn/taobaomysql/monthly/161897
---

# MySQL · TokuDB · 日志子系统和崩溃恢复过程

## TokuDB日志子系统

MySQL重启后自动加载InnoDB和其他的动态plugin，包括TokuDB。每一plugin在注册的时候指定init和deinit回调函数。TokuDB的init/deinit函数分别是`tokudb_init_func`和`tokudb_done_func`。  
MySQL重启过程中调用`tokudb_init_func`进行必要的初始化。在`tokudb_init_func`里面，调用`db_env_create`创建一个env实例，进行参数设置和callback设置。`db_env_create`是一个简单的封装，最终会调用`toku_env_create`来进行参数设置，callback设置和初始化的。`toku_env_create`初始化工作中一个很重要的事情就是调用`toku_logger_create`初始化TokuDB的日志子系统。  
在TokuDB中，日志子系统是由tokulogger数据结构管理的。下面仅列出了主要的数据成员。

```plain
struct tokulogger {
    struct mylock input_lock;             // 保护lsn和in_buf的mutex
    toku_mutex_t output_condition_lock;   // 保护written_lsn ，fsynced_lsn ，等待out_buf可用的条件变量的mutex
    toku_cond_t output_condition;         // 等待out_buf可用的条件变量
    bool output_is_available;             // 标志out_buf可用的条件
    bool is_open;                         // 标志logger是否已打开
    bool write_log_files;                 // 标示是否将redo log buffer写到redo log file
    bool trim_log_files;                  // 标志是否要trim redo log file
    char *directory;                      // redo log所在目录
    int lg_max;                           // redo log最大长度，缺省100M
    LSN lsn;                              // 下一个可用的lsn
    struct logbuf inbuf;                  // 接收redo log entry的buffer
    LSN written_lsn;                      // 最后一次写入的lsn
    LSN fsynced_lsn;                      // 最后一次fsync的lsn
    LSN last_completed_checkpoint_lsn;    // 最近一次checkpoint开始时刻的logger的lsn
    long long next_log_file_number;       // 下一个可用的redo log file的序列号
    struct logbuf outbuf;                 // 写入redo log file的buf
    int n_in_file;                        // 当前redo log file存储日志的字节数
    TOKULOGFILEMGR logfilemgr;            // log file manager的handle
    TXN_MANAGER txn_manager;              // txn manager的handle
};
```

### Logger初始化

Logger子系统在env->create阶段由`toku_logger_create`进行初步的初始化工作。代码片段如下：

```plain
int toku_logger_create (TOKULOGGER *resultp) {
    TOKULOGGER CALLOC(result);
    if (result==0) return get_error_errno();
    result->is_open=false;
    result->write_log_files = true;
    result->trim_log_files = true;
    result->directory=0;
    result->lg_max = 100<<20; // 100MB default
    // lsn is uninitialized
    result->inbuf  = (struct logbuf) {0, LOGGER_MIN_BUF_SIZE, (char *) toku_xmalloc(LOGGER_MIN_BUF_SIZE), ZERO_LSN};
    result->outbuf = (struct logbuf) {0, LOGGER_MIN_BUF_SIZE, (char *) toku_xmalloc(LOGGER_MIN_BUF_SIZE), ZERO_LSN};
    // written_lsn is uninitialized
    // fsynced_lsn is uninitialized
    result->last_completed_checkpoint_lsn = ZERO_LSN;
    // next_log_file_number is uninitialized
    // n_in_file is uninitialized
    toku_logfilemgr_create(&result->logfilemgr);
    *resultp=result;
    ml_init(&result->input_lock);
    toku_mutex_init(&result->output_condition_lock, NULL);
    toku_cond_init(&result->output_condition, NULL);
    result->output_is_available = true;
    return 0;
}
```

Logger子系统在env->open阶段，调用`toku_logger_open`函数进行进一步的初始化。函数`toku_logger_open`是`toku_logger_open_with_last_xid`的简单封装。Env->open最终调用`toku_logger_open_with_last_xid`解析redo log file获取下一个可用的lsn，下一个可用的redo log file的序列号index并打开相应redo log file。在env->open时，调用`toku_logger_open_with_last_xid`的最后一个参数last\_xid为TXNID\_NONE，表示由`toku_logger_open_with_last_xid`指定事务子系统初始化时最新的txnid。  
解析redo log file的过程在函数`toku_logfilemgr_init`实现，依次解析redo log目录下的每一个文件名符合特定格式的redo log file，从中读取最后一个log entry的lsn保存下来。Redo log文件名遵循”log$index.tokulog$version”格式，$index是64位无符号整数表示的redo log file的序列号index，$version是32位无符号整数表示版本信息。  
如果最新的redo log file最后一个log entry是LT\_shutdown（表示正常关闭不需要进行recovery），那么把对应的txnid记录在last\_xid\_if\_clean\_shutdown变量，作为TokuDB事务子系统初始化时最新的txnid。在解析redo log file的时候，还会用最新的redo log file的最后一个log entry的lsn更新logger的lsn，written\_lsn，fsynced\_lsn。接着，`toku_logger_find_next_unused_log_file`找到下一个可用的redo log文件的序列号，并创建新的redo log file。每个redo log file最开始的12个字节是固定的，首先是8个字节的magic字符串“tokulogg“，紧接着4个字节是log的版本信息。代码片段如下：

```plain
int
toku_logger_open_with_last_xid(const char *directory /* redo log dir */, TOKULOGGER logger, TXNID last_xid) {
    if (logger->is_open) return EINVAL;
    TXNID last_xid_if_clean_shutdown = TXNID_NONE;
    r = toku_logfilemgr_init(logger->logfilemgr, directory, &last_xid_if_clean_shutdown);
    if ( r!=0 )
        return r;
    logger->lsn = toku_logfilemgr_get_last_lsn(logger->logfilemgr);
    logger->written_lsn = logger->lsn;
    logger->fsynced_lsn = logger->lsn;
    logger->inbuf.max_lsn_in_buf  = logger->lsn;
    logger->outbuf.max_lsn_in_buf = logger->lsn;
    r = open_logdir(logger, directory);
    if (r!=0) return r;
    long long nexti;
    r = toku_logger_find_next_unused_log_file(logger->directory, &nexti);
    if (r!=0) return r;
    logger->next_log_file_number = nexti;
    r = open_logfile(logger);
    if (r!=0) return r;
    if (last_xid == TXNID_NONE) {
        last_xid = last_xid_if_clean_shutdown;
    }
    toku_txn_manager_set_last_xid_from_logger(logger->txn_manager, last_xid);
    logger->is_open = true;
    return 0;
}
```

到这里，TokuDB的logger子系统就初始化好了，在处理DDL或者DML或者TokuDB执行checkpoint的时候，都需要先写rollback（undo）log，redo log。Rollback在之前的月报[MySQL · TokuDB · 事务子系统和 MVCC 实现](http://mysql.taobao.org/monthly/2016/03/01/) 谈到过，这里不再赘述。

### 写redo log

下面我们一起看一下往redo log新加一条insert的过程。函数`toku_log_enq_insert`的第2，第5，第6，第7，第8参数表示描述一条insert的五元组(lsn, FT, xid, key, value)。代码片段如下：

```plain
void toku_log_enq_insert (TOKULOGGER logger, LSN *lsnp, int do_fsync, TOKUTXN txn, FILENUM filenum, TXNID_PAIR xid, BYTESTRING key, BYTESTRING value) {
  if (logger == NULL) {
     return;
  }
  if (txn && !txn->begin_was_logged) {
    invariant(!txn_declared_read_only(txn));
    // 记录txn begin
    toku_maybe_log_begin_txn_for_write_operation(txn);
  }
  if (!logger->write_log_files) {
    // logger->write_log_files为FALSE，表示不写redo，递增lsn就可以返回了。
    ml_lock(&logger->input_lock);
    logger->lsn.lsn++;
    if (lsnp) *lsnp=logger->lsn;
    ml_unlock(&logger->input_lock);
    return;
  }
  const unsigned int buflen= (+4 // log entry的长度，参与crc计算
                              +1 // log命令，对应insert来说是‘I’
                              +8 // lsn
                              +toku_logsizeof_FILENUM(filenum) // filenum，表示哪个FT文件
                              +toku_logsizeof_TXNID_PAIR(xid) // xid，表示txnid
                              +toku_logsizeof_BYTESTRING(key) // key
                              +toku_logsizeof_BYTESTRING(value) // data
                              +8 // crc + len // crc和log entry长度（不参与crc计算）
                             );
  struct wbuf wbuf;
  ml_lock(&logger->input_lock);
  toku_logger_make_space_in_inbuf(logger, buflen);
  wbuf_nocrc_init(&wbuf, logger->inbuf.buf+logger->inbuf.n_in_buf, buflen);
  wbuf_nocrc_int(&wbuf, buflen);
  wbuf_nocrc_char(&wbuf, 'I');
  logger->lsn.lsn++;
  logger->inbuf.max_lsn_in_buf = logger->lsn;
  wbuf_nocrc_LSN(&wbuf, logger->lsn);
  if (lsnp) *lsnp=logger->lsn;
  wbuf_nocrc_FILENUM(&wbuf, filenum);
  wbuf_nocrc_TXNID_PAIR(&wbuf, xid);
  wbuf_nocrc_BYTESTRING(&wbuf, key);
  wbuf_nocrc_BYTESTRING(&wbuf, value);
  wbuf_nocrc_int(&wbuf, toku_x1764_memory(wbuf.buf, wbuf.ndone));
  wbuf_nocrc_int(&wbuf, buflen);
  assert(wbuf.ndone==buflen);
  logger->inbuf.n_in_buf += buflen;
  toku_logger_maybe_fsync(logger, logger->lsn, do_fsync, true);
}
```

TokuDB的logger有两个buffer：inbuf和outbuf。Inbuf表示接收log entry的buffer，而outbuf表示写到redo log文件的buffer。这两个buffer是如何切换的呢？当inbuf满或者inbuf里的free space无法满足新来的log entry的存储需求时，需要触发redo buffer flush过程，即将inbuf日志flush到redo log文件里。这个过程比较耗时，而且很可能inbuf里面还有free space，只是由于当前这个log entry比较大而无法满足存储需求，TokuDB实现了output permission机制，使得需要free space的请求等待在output permission的条件变量上，其他client thread上下文的redo log请求可以继续使用inbuf写日志。等待上一个flush完成后（即条件变量被signaled），检查当前inbuf的free space，如果可以满足这条redo log entry就直接返回，说明别的线程帮我们flush好了。如果free space不够，需要在当前线程的上下文去做flush，实际上是把inbuf和outbuf互换，然后把outbuf写到redo log文件中。写完之后适当调整inbuf的大小使之满足当前redo log entry请求。最后唤醒等待inbuf提供足够空间的线程（阻塞在output permission上的线程）。简而言之，把redo log buffer拆分成inbuf和outbuf，最重要的作用是在redo log flush的时候不会阻塞新的log entry写入，感兴趣的朋友可以看一下函数`toku_logger_maybe_fsync`的实现，这里就不一一展开了。函数`toku_logger_make_space_in_inbuf`的代码片段如下：

```plain
void
toku_logger_make_space_in_inbuf (TOKULOGGER logger, int n_bytes_needed)
{
    if (logger->inbuf.n_in_buf + n_bytes_needed <= LOGGER_MIN_BUF_SIZE) {
        return;
    }
    ml_unlock(&logger->input_lock);
    LSN fsynced_lsn;
    // 等待前面的redo log flush完成
    grab_output(logger, &fsynced_lsn);

    ml_lock(&logger->input_lock);
    if (logger->inbuf.n_in_buf + n_bytes_needed <= LOGGER_MIN_BUF_SIZE) {
        // 其他线程帮助flush redo log，直接返回。
        release_output(logger, fsynced_lsn);
        return;
    }
    if (logger->inbuf.n_in_buf > 0) {
        // 交换inbuf，outbuf
        swap_inbuf_outbuf(logger);

        // 把outbuf里的日志写回
        write_outbuf_to_logfile(logger, &fsynced_lsn);
    }
    // 适当调整inbuf大小
    if (n_bytes_needed > logger->inbuf.buf_size) {
        assert(n_bytes_needed < (1<<30)); // redo log entry必须小于1G
        int new_size = max_int(logger->inbuf.buf_size * 2, n_bytes_needed);
        assert(new_size < (1<<30)); // inbuf必须小于1G
        XREALLOC_N(new_size, logger->inbuf.buf);
        logger->inbuf.buf_size = new_size;
    }
    // 唤醒等待flush redo log的线程
    release_output(logger, fsynced_lsn);
}
```

## TokuDB崩溃恢复过程

### 判断是否进行recovery

前面提到MySQL重启过程中会调用`db_env_create`创建env实例，进行参数设置和callback设置，然后调用env->open来做进一步初始化。同样env->open也是一个回调函数，它是在`db_env_create`设置的，指向env\_open函数。  
在env\_open里调用validate\_env判断是否需要进行recovery。validate\_env函数返回时表明这个env是否是emptyenv (env目录为空，且不存在rollback文件，不存在数据文件)，是否是newnev (env目录不存在)，是否是emptyrollback (env目录存在，rollback文件为空)。  
如果满足条件 !emptyenv && !new\_env && is\_set(DB\_RECOVERY) 就尝试进行recovery。简单地说recovery的条件就是env存在，log\_dir存在，redo log存在。  
判断是否真正做recovery的函数是`tokuft_needs_recovery`。代码如下：

```plain
int tokuft_needs_recovery(const char *log_dir, bool ignore_log_empty) {
    int needs_recovery;
    int r;
    TOKULOGCURSOR logcursor = NULL;

    r = toku_logcursor_create(&logcursor, log_dir);
    if (r != 0) {
        needs_recovery = true; goto exit;
    }

    struct log_entry *le;
    le = NULL;
    r = toku_logcursor_last(logcursor, &le);
    if (r == 0) {
        needs_recovery = le->cmd != LT_shutdown;
    }
    else {
        needs_recovery = !(r == DB_NOTFOUND && ignore_log_empty);
    }
 exit:
    if (logcursor) {
        r = toku_logcursor_destroy(&logcursor);
        assert(r == 0);
    }
    return needs_recovery;
}
```

`tokuft_needs_recovery`尝试读取最后一条redo log entry，如果不是LT\_shutdown，就需要真正做recovery。读取最后一条redo log entry的代码片段如下：

```plain
int toku_logcursor_last(TOKULOGCURSOR lc, struct log_entry **le) {
    // 打开最后一个redo log文件
    if ( !lc->is_open ) {
        r = lc_open_logfile(lc, lc->n_logfiles-1);
        if (r!=0)
            return r;
        lc->cur_logfiles_index = lc->n_logfiles-1;
    }
    while (1) {
        // 移到最后一个redo log的文件末尾
        r = fseek(lc->cur_fp, 0, SEEK_END);    assert(r==0);
        // 从当前位置（redo log末尾）向前读一个log entry
        r = toku_log_fread_backward(lc->cur_fp, &(lc->entry));
        if (r==0) // 读成功
            break;
        if (r>0) {
            // 读失败
            toku_log_free_log_entry_resources(&(lc->entry));
            // 从当前redo log头部开始向后scan直到找到第一非法log Entry的位置，并把redo log文件truncate到那个位置。
            r = lc_fix_bad_logfile(lc);
            if ( r != 0 ) {
                fprintf(stderr, "%.24s TokuFT recovery repair unsuccessful\n", ctime(&tnow));
                return DB_BADFORMAT;
            }
            // 重新读redo log entry
            r = toku_log_fread_backward(lc->cur_fp, &(lc->entry));
            if (r==0) // 读到好的redo log entry
                break;
        }
        // 当前redo log没有访问的log entry，切换到上一个redo log文件
        r = lc_close_cur_logfile(lc);
        if (r!=0)
            return r;
        if ( lc->cur_logfiles_index == 0 )
            return DB_NOTFOUND;
        lc->cur_logfiles_index--;
        r = lc_open_logfile(lc, lc->cur_logfiles_index);
        if (r!=0)
            return r;
    }
}
```

在读最后一个log entry的过程中，在读log entry出错的情况下（crash的时候把redo log写坏了）会调用`lc_fix_bad_logfile`尝试修复redo log文件。修复的过程很简单：从当前redo log头部开始向后scan直到找到第一非法log entry的位置，并把redo log文件truncate到那个位置。此时，文件指针也指向文件末尾。极端的情况是，修复完redo log，发现当前redo log中的所有entry都是坏的，那样需要切换到前面一个redo log文件。

### Recovery过程

如果需要做recovery，TokuDB会调用do\_recovery进行恢复，恢复的时候先做redo log apply，然后进行undo rollback。代码片段如下：

```plain
static int do_recovery(RECOVER_ENV renv, const char *env_dir, const char *log_dir) {
    r = toku_logcursor_create(&logcursor, log_dir);
    assert(r == 0);
    scan_state_init(&renv->ss);
    for (unsigned i=0; 1; i++) {
        // 读取前一个log entry，第一次读的是最后一个log entry
        le = NULL;
        r = toku_logcursor_prev(logcursor, &le);
        if (r != 0) {
            if (r == DB_NOTFOUND)
                break;
            rr = DB_RUNRECOVERY;
            goto errorexit;
        }
        // backward阶段处理log entry
        assert(renv->ss.ss == BACKWARD_BETWEEN_CHECKPOINT_BEGIN_END ||
               renv->ss.ss == BACKWARD_NEWER_CHECKPOINT_END);
        logtype_dispatch_assign(le, toku_recover_backward_, r, renv);
        if (r != 0) {
            rr = DB_RUNRECOVERY;
            goto errorexit;
        }
        if (renv->goforward)
            break;
    }

    for (unsigned i=0; 1; i++) {
        // forward阶段处理log entry，首先处理的是checkpoint begin的那个log entry
        assert(renv->ss.ss == FORWARD_BETWEEN_CHECKPOINT_BEGIN_END ||
           renv->ss.ss == FORWARD_NEWER_CHECKPOINT_END);
        logtype_dispatch_assign(le, toku_recover_, r, renv);
        if (r != 0) {
                rr = DB_RUNRECOVERY;
                goto errorexit;
        }

        // 读取下一个log entry
        le = NULL;
        r = toku_logcursor_next(logcursor, &le);
        if (r != 0) {
            if (r == DB_NOTFOUND)
                break;
            rr = DB_RUNRECOVERY;
            goto errorexit;
        }
    }

    // parse redo log结束
    assert(renv->ss.ss == FORWARD_NEWER_CHECKPOINT_END);

    r = toku_logcursor_destroy(&logcursor);
    assert(r == 0);

    // 重启logger
    toku_logger_restart(renv->logger, lastlsn);

    // abort所有未提交的事务
    recover_abort_all_live_txns(renv);
    // 在recovery退出前做一个checkpoint
     r = toku_checkpoint(renv->cp, renv->logger, NULL, NULL, NULL, NULL, RECOVERY_CHECKPOINT);
    assert(r == 0);
    return 0;
}
```

Scan log entry分别两个阶段：backward阶段和forward阶段。这两个阶段是由scan\_state状态机控制的。在scan开始之前在`scan_state_init`函数中把状态机ss的初始状态设置为BACKWARD\_NEWER\_CHECKPOINT\_END。

![](assets/1590580500-2bec0ada4c71f66147ef0122630a6397.jpg)

*   Backward阶段：从最后一个log entry开始向前读，直到读到checkpoint end。对在这个过程中读到的每一个log entry调用`logtype_dispatch_assign(le, toku_recover_backward_, r, renv)`。在这个阶段对于checkpoint以外的操作，toku\_recover\_backward\_前缀的处理函数都是noop。当读到checkpoint end的log entry时，会把ss状态设置为BACKWARD\_BETWEEN\_CHECKPOINT\_BEGIN\_END，并记录这个checkpoint的begin\_lsn和lsn。然后继续向前scan直到读到checkpoint begin的log entry，确保ss中记录的checkpoint\_begin\_lsn和log entry的lsn是相等的，然后 把ss的状态设置为FORWARD\_BETWEEN\_CHECKPOINT\_BEGIN\_END，并设置renv->goforward为TRUE。
*   Forward阶段：对当前的log entry调用`logtype_dispatch_assign(le, toku_recover_, r, renv)`重放redo log。然后向后scan直到读到checkpoint end，确保ss中记录的`checkpoint_begin_lsn`和`checkpoint_end_lsn`与log entry里面记录的`lsn_begin_checkpoint`和lsn是相等的，然后把ss的状态设置为FORWARD\_NEWER\_CHECKPOINT\_END。这样，崩溃之前的最后一个checkpoint就回放完成了。下面要做的事情就是，回放committed txn的redo log。代码片段如下：

```plain
static void scan_state_init(struct scan_state *ss) {
    ss->ss = BACKWARD_NEWER_CHECKPOINT_END;
    ss->checkpoint_begin_lsn = ZERO_LSN;
    ss->checkpoint_end_lsn = ZERO_LSN;
    ss->checkpoint_num_fassociate = 0;
    ss->checkpoint_num_xstillopen = 0;
    ss->last_xid = 0;
}
static int toku_recover_backward_end_checkpoint (struct logtype_end_checkpoint *l, RECOVER_ENV renv) {
    switch (renv->ss.ss) {
    case BACKWARD_NEWER_CHECKPOINT_END:
        renv->ss.ss = BACKWARD_BETWEEN_CHECKPOINT_BEGIN_END;
        renv->ss.checkpoint_begin_lsn.lsn = l->lsn_begin_checkpoint.lsn;
        renv->ss.checkpoint_end_lsn.lsn   = l->lsn.lsn;
        renv->ss.checkpoint_end_timestamp = l->timestamp;
        return 0;
    case BACKWARD_BETWEEN_CHECKPOINT_BEGIN_END:
        abort();
    default:
        break;
    }
    abort();
}
static int toku_recover_backward_begin_checkpoint (struct logtype_begin_checkpoint *l, RECOVER_ENV renv) {
    int r;
    switch (renv->ss.ss) {
    case BACKWARD_NEWER_CHECKPOINT_END:
        // incomplete checkpoint, nothing to do
        r = 0;
        break;
    case BACKWARD_BETWEEN_CHECKPOINT_BEGIN_END:
        assert(l->lsn.lsn == renv->ss.checkpoint_begin_lsn.lsn);
        renv->ss.ss = FORWARD_BETWEEN_CHECKPOINT_BEGIN_END;
        renv->ss.checkpoint_begin_timestamp = l->timestamp;
        renv->goforward = true;
        r = 0;
        break;
    default:
        abort();
        break;
    }
    return r;
}
static int toku_recover_begin_checkpoint (struct logtype_begin_checkpoint *l, RECOVER_ENV renv) {
    int r;
    TXN_MANAGER mgr = toku_logger_get_txn_manager(renv->logger);
    switch (renv->ss.ss) {
    case FORWARD_BETWEEN_CHECKPOINT_BEGIN_END:
        assert(l->lsn.lsn == renv->ss.checkpoint_begin_lsn.lsn);
        invariant(renv->ss.last_xid == TXNID_NONE);
        renv->ss.last_xid = l->last_xid;
        toku_txn_manager_set_last_xid_from_recovered_checkpoint(mgr, l->last_xid);
        r = 0;
        break;
    case FORWARD_NEWER_CHECKPOINT_END:
        assert(l->lsn.lsn > renv->ss.checkpoint_end_lsn.lsn);
        // Verify last_xid is no older than the previous begin
        invariant(l->last_xid >= renv->ss.last_xid);
        // Verify last_xid is no older than the newest txn
        invariant(l->last_xid >= toku_txn_manager_get_last_xid(mgr));
        r = 0; // ignore it (log only has a begin checkpoint)
        break;
    default:
        abort();
        break;
    }
    return r;
}
static int toku_recover_end_checkpoint (struct logtype_end_checkpoint *l, RECOVER_ENV renv) {
    int r;
    switch (renv->ss.ss) {
    case FORWARD_BETWEEN_CHECKPOINT_BEGIN_END:
        assert(l->lsn_begin_checkpoint.lsn == renv->ss.checkpoint_begin_lsn.lsn);
        assert(l->lsn.lsn == renv->ss.checkpoint_end_lsn.lsn);
        assert(l->num_fassociate_entries == renv->ss.checkpoint_num_fassociate);
        assert(l->num_xstillopen_entries == renv->ss.checkpoint_num_xstillopen);
        renv->ss.ss = FORWARD_NEWER_CHECKPOINT_END;
        r = 0;
        break;
    case FORWARD_NEWER_CHECKPOINT_END:
        assert(0);
        return 0;
    default:
        assert(0);
        return 0;
    }
    return r;
}
```

上面我们是TokuDB recovery的过程。对读redo log一笔带过。现在一起看看读log entry的过程：

*   向后读：从当前位置读4个字节的长度len1，然后读1个字节cmd。然后按照不同cmd的定义来读log entry。
*   向前读：从当前位置读nocrc的长度len2，把文件指针向前移动len2个字节。从那个位置向后读。
*   Verify：读的过程需要计算crc校验码。Len1是参与crc计算的，而len2不参与crc计算。计算得到的crc应该与log entry里面记录的crc相等。而且len1应该等于len2。

上一篇：[SQLServer · 最佳实践 · 透明数据加密在SQLServer的应用](https://www.kancloud.cn/taobaomysql/monthly/161896)下一篇：[MongoDB · 特性分析 · Sharded cluster架构原理](https://www.kancloud.cn/taobaomysql/monthly/161898)

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/161897)

创建于: 2020-05-27 19:55:00

目录: default

标签: `www.kancloud.cn`

