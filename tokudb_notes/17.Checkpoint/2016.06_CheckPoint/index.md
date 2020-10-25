---
title: MySQL · TokuDB · checkpoint过程 · 数据库内核月报 · 看云
category: default
tags: 
  - www.kancloud.cn
created_at: 2020-05-27 20:03:47
original_url: https://www.kancloud.cn/taobaomysql/monthly/181782
---

# MySQL · TokuDB · checkpoint过程

TokuDB的buffer pool（在TokuDB中被称作cachetable）维护几个后台工作线程定期处理一些任务。

其中有一个工作线程叫做checkpointer线程，每60秒启动一次把cachetable中所有脏页写回到磁盘上。

TokuDB只支持这一种checkpoint方式，用MySQL术语来说就是sharp checkpoint。

每次checkpoint过程写的脏页数目可能会比较多，而且在写回的过程中需要一直持有节点的读写锁，因此，checkpoint时索引的访问性能会受到一定程度的影响。

为了降低checkpoint对性能影响，TokuDB对每个脏页clone一份用于写回，在clone的过程中是持有节点的读写锁的，clone结束会放掉读写锁。

TokuDB checkpoint过程分为如下五个步骤：

*   获取全局的checkpoint锁
*   Begin checkpoint
*   End checkpoint
*   清理redo日志
*   释放全局的checkpoint锁

下面我们一起看一下begin checkpoint和end checkpoint的详细过程。

## Begin checkpoint

在checkpoint开始时刻要做一些准备工作，诸如：

1.  pin FT  
    给CACHEFILE对应的FT加pinned\_by\_checkpoint标记，保证CACHEFILE不会从内存里移除。CACHEFILE记录了索引包含的数据节点列表和描述索引对应文件的相关信息。
    
2.  对每个CACHEFILE加for\_checkpoint标记  
    标识此CACHEFILE数据属于当前的checkpoint。
    
3.  记redo日志  
    记录checkpoint开始时刻的lsn。  
    写redo日志：begin checkpoint日志项，checkpoint打开索引文件日志项，live txn日志项
    
4.  对每个PAIR（数据页）加checkpoint\_pending标记  
    遍历cachetable里面每个数据页，如果数据页对应的索引文件（CACHEFILE）属于checkpoint，对数据页加checkpoint\_pending标记，并加入到全局m\_pending\_head双向链表里面。
    
5.  更新checkpoint header信息  
    clone一份FT header，记做ft->checkpoint\_header,记录checkpoint开始时刻BTT（Block Translation Table）的位置。  
    TokuDB每次checkpoint都会把数据写到一个新的地方，索引逻辑页号（或者块号）到索引文件offset的映射关系记录在BTT里面。  
    ft->checkpoint\_header的类型为FT\_CHECKPOINT\_INPROGRESS，lsn为checkpoint开始时刻的lsn。
    
6.  克隆BTT，BTT里面有个translation表，记录逻辑页号到索引文件offset的映射关系。这个表有三个版本：
    

*   \_current（当前的，类型为TRANSLATION\_CURRENT）
*   \_inprogress（checkpoint开始时刻的，类型为TRANSLATION\_INPROGRESS）
*   \_checkpointed（上次checkpont的，类型为TRANSLATION\_CHECKPOINTED）   
    就是把TRANSLATION\_CURRENT复制一份，并把类型设置为TRANSLATION\_INPROGRESS。

注：  
1, 2阶段在m\_cf\_list->read\_lock保护下进行  
4, 5阶段在此过程在pair list的锁和m\_cf\_list->read\_lock保护下进行。。  
注意是拿了pair list上所有的锁，m\_list\_lock读锁，m\_pending\_lock\_expensive写锁，m\_pending\_lock\_cheap写锁，保证不能向pair list添加/删除数据页；不能把pair list的数据页evict出内存；同时也阻止在get\_and\_pin的过程中client线程池帮助写回属于checkpoint的脏页。这三个锁都是保护pair list的，按照不同的功能拆分成三个锁。

```plain
void checkpointer::begin_checkpoint() {
    // 1\. Initialize the accountability counters.
    m_checkpoint_num_txns = 0;

    // 2\. Make list of cachefiles to be included in the checkpoint.
    m_cf_list->read_lock();
    m_cf_list->m_active_fileid.iterate<void *, iterate_note_pin::fn>(nullptr);
    m_checkpoint_num_files = m_cf_list->m_active_fileid.size();
    m_cf_list->read_unlock();

    // 3\. Create log entries for this checkpoint.
    if (m_logger) {
        this->log_begin_checkpoint();
    }

    bjm_reset(m_checkpoint_clones_bjm);

    m_list->write_pending_exp_lock();
    m_list->read_list_lock();
    m_cf_list->read_lock(); // needed for update_cachefiles
    m_list->write_pending_cheap_lock();

    // 4\. Turn on all the relevant checkpoint pending bits.
    this->turn_on_pending_bits();

    // 5\. Clone BTT and FT header
    this->update_cachefiles();
    m_list->write_pending_cheap_unlock();
    m_cf_list->read_unlock();
    m_list->read_list_unlock();
    m_list->write_pending_exp_unlock();
}
```

## End checkpoint

在end checkpoint的阶段

*   把所有的CACHEFIlE记录在checkpoint\_cfs数组里面，为后面的步骤做准备。
*   然后调用checkpoint\_pending\_pairs函数把m\_pending\_head双向链表的数据页写回到磁盘上。  
    Checkpoint\_pending\_pairs遍历m\_pending\_head链表，对每个数据页判断是否真的需要写回。  
    因为一次checkpoint的时间比较长，有的数据页可能是被client线程池帮忙写回了，这里就不需要再做一次写回操作。如果需要写回，就调用clone\_callback克隆一份。  
    在clone的过程中是持有数据页的读写锁和disk\_nb\_mutex（mutex语义，表示有I/O在进行），克隆结束后，释放读写锁，只持有disk\_nb\_mutex锁，由checkpointer线程把数据页写回（cloned副本）。写回结束后，释放disk\_nb\_mutex。  
    如果数据页没有设置clone\_callback（缺省是都会设置的），由checkpointer线程把数据页（注意，是数据页本身）写回，写回过程中是持有读写锁和disk\_nb\_mutex的。  
    写回结束后清除checkpoint\_pending标记和dirty标记。  
    函数checkpoint\_pending\_pairs把所有的数据页写回到磁盘上，后面要做的就是metadata的修改。
*   对checkpoint\_cfs数组的每个CACHEFILE调用checkpoint\_userdata回调函数（实际上是ft\_checkpoint函数）把BTT（Block Translation Table）和ft->checkpoint\_header序列化到磁盘上。  
    BTT的rootnum  
    在FT索引文件里有两个位置可以保存ft->header：偏移0和偏移4096。  
    TokuDB采用round robin的方式，把奇数次（1,3,5…）checkpoint的header存储在偏移为0的地方;  
    把偶数次（2,4,6,…）checkpoint的header存储在偏移为4096的位置上。  
    然后更新ft->h->checkpoint\_lsn为checkpoint开始时刻的lsn。
*   写redo日志：end checkpoint日志项。
*   通知logger子系统logger->last\_completed\_checkpoint\_lsn为checkpoint开始时刻的lsn。
*   对checkpoint\_cfs数组保存的每个CACHEFILE调用end\_checkpoint\_userdata回调函数（实际上是ft\_end\_checkpoint）把\_checkpointed记录的上次checkpoint写回的数据页所占用空间释放掉。并且把这次checkpoint的BTT保存在\_checkpointed，然后清空\_inprogress，表示checkpoint结束，当前没有正在进行的checkpoint。在ft\_end\_checkpoint里面还做了一个事情就是把ft->checkpoint\_header释放并置为空，到这里checkpoint的工作就完成了。
*   unpin FT

```plain
void checkpointer::end_checkpoint(void (*testcallback_f)(void*),  void* testextra) {
    toku::scoped_malloc checkpoint_cfs_buf(m_checkpoint_num_files * sizeof(CACHEFILE));
    CACHEFILE *checkpoint_cfs = reinterpret_cast<CACHEFILE *>(checkpoint_cfs_buf.get());

    this->fill_checkpoint_cfs(checkpoint_cfs);
    this->checkpoint_pending_pairs();
    this->checkpoint_userdata(checkpoint_cfs);
    // For testing purposes only.  Dictionary has been fsync-ed to disk but log has not yet been written.
    if (testcallback_f) {
        testcallback_f(testextra);
    }
    this->log_end_checkpoint();
    this->end_checkpoint_userdata(checkpoint_cfs);

    // Delete list of cachefiles in the checkpoint,
    this->remove_cachefiles(checkpoint_cfs);

}
```

## Checkpoint的redo日志

下面我们一起看一下checkpoint过程记录的redo日志：

*   Begin\_checkpoint：表示begin checkpoint的日志项
*   Fassociate：表示打开的索引的日志项
*   End\_checkpoint：表示end checkpoint的日志项

```plain
./tdb_logprint < data/log000000000002.tokulog27
begin_checkpoint         'x': lsn=88 timestamp=1455623796540257 last_xid=153 crc=470dd9ea len=37
fassociate               'f': lsn=89 filenum=0 treeflags=0 iname={len=15 data="tokudb.rollback"} unlink_on_close=0 crc=8606e9b1 len=49
fassociate               'f': lsn=90 filenum=1 treeflags=4 iname={len=18 data="tokudb.environment"} unlink_on_close=0 crc=92dc4c1c len=52
fassociate               'f': lsn=91 filenum=3 treeflags=4 iname={len=16 data="tokudb.directory"} unlink_on_close=0 crc=86323b7e len=50
end_checkpoint           'X': lsn=92 lsn_begin_checkpoint=88 timestamp=1455623796541659 num_fassoc
```

上一篇：[MariaDB · 新特性 · 窗口函数](https://www.kancloud.cn/taobaomysql/monthly/181781)下一篇：[MySQL · 特性分析 · 内部临时表](https://www.kancloud.cn/taobaomysql/monthly/181783)

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/181782)

创建于: 2020-05-27 20:03:47

目录: default

标签: `www.kancloud.cn`

