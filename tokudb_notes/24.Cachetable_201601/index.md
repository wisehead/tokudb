---
title: MySQL · TokuDB · Cachetable 的工作线程和线程池 · 数据库内核月报 · 看云
category: default
tags: 
  - www.kancloud.cn
created_at: 2020-05-27 15:26:47
original_url: https://www.kancloud.cn/taobaomysql/monthly/117961
---

# MySQL · TokuDB · Cachetable 的工作线程和线程池

## 介绍

TokuDB也有类似InnoDB的buffer pool叫做cachetable，存储数据节点（包括叶节点和中间节点）和rollback段，本文中为了表达简单，叶节点，中间节点和rollback段统称数据节点。Cachetable是全局唯一的，它与MySQL实例存在一一对应的关系。TokuDB没有采用常见的BTREE(BTREE+，BTREE\*)表示索引，而是采用Fractal Tree，简称FT。FT跟BTREE+类似，维护了一个树形的有序结构，中间节点存储pivot（TokuDB的中间节点还包含message buffer），叶节点存储数据。

数据库启动的时候会去初始化cachetable。Client线程（调用栈上下文所在的线程）要访问某个数据节点会首先在cachetable里面查找，找到就立即返回；否则会在cachetable申请一个cache项，然后从磁盘上加载数据到那个cache项。TokuDB里表示cache项的数据结构叫做pair，记录(节点块号/页号，数据节点）的对应关系。在MySQL的缺省引擎InnoDB中，数据和索引是存储在一个文件里的，而TokuDB中每个索引对应一个单独的磁盘文件。

Cachetable是一个hash表，每个bucket里面包含多个pair，共1024\*1024个bucket。属于相同索引的pair由cachefile来管理。TokuDB有一个优化在后面会涉及到，这里先简单提一下。当server层显示关闭某个TokuDB表时FT层会调用`toku_cachefile_close`关闭表或者索引，并把缓存的数据节点从cachetable删除；但这些数据节点仍然保留在cachefile中（保留在内存中）。这种cachefile会被加到的stale列表里面，它包含的数据节点会在内存里呆一段时间。近期再次访问这个索引时，首先会在active列表里查找索引对应的cachefile。若没有找到会尝试在stale列表查找并把找到的cachefile的数据节点重新加到cachetable里去。近期再次访问相同的数据集就不必从磁盘上加载了。

## Cachetable的工作线程(worker thead)

Cachetable创建了三个工作线程：

1.  evictor线程：释放部分cachetable内存空间；
2.  cleaner线程：flush中间节点的message buffer到叶节点；
3.  checkpointer线程：写回dirty数据。

## Cachetable的线程池

Cachetable创建了三个线程池：

1.  client线程池：帮助cleaner线程flush中间节点的message buffer；
2.  cachetable线程池：
    *   帮助client线程fetch/partial fetch数据节点
    *   帮助evictor线程evict/partial evict数据节点
    *   从cachetable删除时，后台删除数据节点
3.  checkpoint线程池：帮助client线程写回处于checkpoint\_pending状态的数据节点。

## Cachetable的几个主要队列

1.  m\_clock\_head：新加载的数据节点除了加入hash方便快速定位，也会加入此队列。可以理解成cachetable的LRU队列；
2.  m\_cleaner\_head：指向m\_clock\_head描述LRU队列，cleaner线程从这个位置开始扫描找到memory pressure最大的中间节点发起message buffer flush操作；
3.  m\_checkpoint\_head：指向m\_clock\_head描述LRU队列，checkpointer线程在begin checkpoint阶段从这个位置开始扫描，把每个数据节点加到m\_pending\_head队列；
4.  m\_pending\_head：checkpointer线程在end checkpoint阶段从这个位置开始扫描，把ditry数据节点写回到磁盘上。

## Evictor线程

随着数据逐渐加载到cachetable，其消耗的内存空间越来越大，当达到一定程度时evictor工作线程会被唤醒尝试释放一些数据节点。Evitor线程定期运行(缺省1秒)。Evictor定义四个watermark来评价当前cachetable消耗内存的程度：

1.  m\_low\_size\_watermark: 达到此watermark以后，evictor线程停止释放内存空间。通俗的说，这就是cachetable消耗内存的上限；
2.  m\_low\_size\_hysteresis： 达到此watermark以后，client线程（也就是server层线程）唤醒evictor线程释放内存。一般是m\_low\_size\_watermark的1.1倍；
3.  m\_high\_size\_hysteresis: 达到此watermark以后，阻塞的client线程会被唤醒。一般是m\_low\_size\_watermark的1.2倍；
4.  m\_high\_size\_watermark：达到此watermark以后，client线程会被阻塞在m\_flow\_control\_cond条件变量上等待evictor线程释放内存。一般是m\_low\_size\_watermark的1.5倍。

### Evictor线程被唤醒的时机

1.  添加新pair；
2.  Get pair时，需要fetch或者partial fetch数据节点；
3.  Evictor destroy时，唤醒等待的client线程；
4.  释放若干数据节点后，Evictor判断是否要继续运行。

铺垫了这么多，下面一起来看一下evictor线程的主体函数`run_eviction`。`run_eviction`是一个while循环调用`eviction_needed`判断是否要进行eviction。如下所示：m\_size\_current表示cachetable的当前size，m\_size\_evicting表示当前正在evicting的数据节点消耗的内存空间。两者的差就是这次eviction运行前，cachetable最终能到达的size。  
伪码如下：

```plain
bool eviction_needed() {
    return (m_size_current - m_size_evicting) > m_low_size_watermark;
}
void run_eviction(){
    uint32_t num_pairs_examined_without_evicting = 0;
    while (eviction_needed()) {
        if (m_num_sleepers > 0 && should_sleeping_clients_wakeup()) {
            /* signal the waiting client threads */
        }
        bool some_eviction_ran = evict_some_stale_pair();
        if (!some_eviction_ran) {
            get m_pl->read_list_lock;
            if (!curr_in_clock) {
                /* nothing to evict */
               break;
            }
            if (num_pairs_examined_without_evicting > m_pl->m_n_in_table) {
                /* everything is in use */
                break;
            }
            bool eviction_run = run_eviction_on_pair(curr_in_clock);
            if (eviction_run) {
                // reset the count
                num_pairs_examined_without_evicting = 0;
            }
            else {
                num_pairs_examined_without_evicting++;
            }
            release m_pl->read_list_lock;
        }
    }
}
```

eviction\_needed 返回true时evictor尝试释放内存。它首先看一下当前的cachetable是否降到m\_high\_size\_hysteresis以下，若是就唤醒等待在m\_flow\_control\_cond条件变量上的client线程。然后，cachetable会先尝试回收stale列表里面cachefile上的数据节点。若stale列表里面没有可回收的数据节点，就会从m\_clock\_head开始尝试回收内存。对于近期没有被访问过的数据节点，会调用`try_evict_pair`尝试回收；否则会使之逐渐退化并尝试partial evict。如果把整个m\_clock\_head队列扫描一遍都没发现可回收的数据节点，那么这次evictor线程的工作就完成了，等下次被唤醒时再次尝试回收内存。

## Cleaner线程

Cleaner是另一个定期运行(缺省1秒)的工作线程，从m\_cleaner\_head开始最多扫8个数据节点，从中找到cache pressure最大的节点（这个过程会skip掉正在被其他线程访问的节点）。由于叶节点和rollback段的cache pressure为0，找到的节点一定是中间节点。如果这个节点设置了checkpoint\_pending标记，那么需要先调用`write_locked_pair_for_checkpoint`把数据写回再调用`cleaner_callback`把中间节点的message buffer刷到叶节点上去。数据写回的过程，如果节点设置了`clone_callback`，写回是由checkpoint线程池来完成的；没有设置`clone_callback`的情况，写回是由cleaner线程完成的。中间节点flush message buffer是一个很复杂的过程，涉及到message apply和merge等操作，打算另写一篇文章介绍。  
伪码如下：

```plain
run_cleaner(){
    uint32_t num_iterations = get_iterations(); // by default, iteration == 1
    for (uint32_t i = 0; i < num_iterations; ++i) {
        get pl->read_list_lock;
        PAIR best_pair = NULL;
        int n_seen = 0;
        long best_score = 0;
        const PAIR first_pair = m_cleaner_head;
        if (first_pair == NULL) {
            /* nothing to clean */
            break;
        }
        /* pick up best_pair */
        do {
            get m_cleaner_head pair lock;
            skip m_cleaner_head if which was being referenced by others
            n_seen++;
            long score = 0;
            bool need_unlock = false;
            score = m_cleaner_head cache pressure
            if (best_score < score) {
                best_score = score;
                if (best_pair) {
                    need_unlock = true;
                }
                best_pair = m_cleaner_head;
            } else {
                need_unlock = true;
            }
            if (need_unlock) {
                release m_cleaner_head pair lock;
            }
            m_cleaner_head = m_cleaner_head->clock_next;
        } while (m_cleaner_head != first_pair && n_seen < 8);
        release m_pl->read_list_lock;
        if (best_pair) {
            get best_pair->value_rwlock;
            if (best_pair->checkpoint_pending) {
                write_locked_pair_for_checkpoint(ct, best_pair, true);
            }
            bool cleaner_callback_called = false;
            if (best_pair cache pressure > 0) {
                r = best_pair->cleaner_callback(best_pair->value_data, best_pair->key, best_pair->fullhash, best_pair->write_extraargs);
                cleaner_callback_called = true;
            }
            if (!cleaner_callback_called) {
                release best_pair->value_rwlock;
            }
        }
    }
}
```

## Checkpointer线程

Cachetable的脏数据是由checkpointer线程定期(缺省60秒)刷到磁盘上。  
Checkpointer线程执行过程分为两个阶段：

begin checkpoint阶段

1.  为每个active的cache file打for\_checkpoint标记；
2.  写日志；
3.  为每个数据节点打checkpoint\_pending标记，并加到m\_pending\_head队列；
4.  clone checkpoint\_header: FT的metadata在内存中的数据结构是FT\_HEADER，这个header有两个版本:
    *   h表示当前版本
    *   checkpoint\_header表示当前正在进行checkpoint的版本，是h在checkpoint开始时刻的副本
5.  clone BTT（block translation table）: TokuDB采用BTT记录逻辑页号（blocknum）到文件offset的映射关系。每次刷新数据节点时申请一个未使用的offset，把脏页刷到新的offset位置上，不覆盖老的数据。  
    BTT表也采用类似的机制被映射到FT文件不同的offset上。BTT的起始地址记录在FT\_HEADER中。checkpoint完成时FT\_HEADER会被更新，使新数据生效。用户可以使用checkpoint机制生成backup加速重建数据库的过程。BTT表有三个版本
    *   当前版本(\_current)
    *   正在checkpoint的版本(\_inprogress)
    *   上次checkpoint的版本(\_checkpointed)

end checkpoint阶段

1.  把m\_pending\_head队列里的数据节点挨个写回到磁盘。写的时候首先检查是否设置`clone_callback`方法，如有调用`clone_callback`生成clone节点，在`clone_callback`里可能会对叶节点做rebalance操作，clone完成后调用`cachetable_only_write_locked_data`把cloned pair写回。没有设置clone\_callback的情况会直接调用`cachetable_write_locked_pair`把节点写回。  
    伪码如下：
    
    ```plain
     void write_pair_for_checkpoint_thread (evictor* ev, PAIR p) {
         get p->value_rwlock.write_lock;
         if (p->dirty && p->checkpoint_pending) {
             if (p->clone_callback) {
                 get p->disk_nb_mutex;
                 clone_pair(ev, p);
             } else {
                 cachetable_write_locked_pair(ev, p, true /* for_checkpoint */);
             }
         }
         p->checkpoint_pending = false;
         put p->value_rwlock.write_lock;
         if (p->clone_callback) {
             cachetable_only_write_locked_data(ev, p, true /* for_checkpoint */,
     &attr, true /* is_clone */);
         }
     }
    ```
    
2.  调用`checkpoint_userdata`：
    
    *   写回BTT的\_inprogress版本
    *   写回FT\_HEADER的checkpoint\_header版本，后面会把checkpoint\_header释放掉
3.  调用`end_checkpoint_userdata`：
    
    *   释放BTT \_checkpointed版本占用的地址空间
    *   把\_inprogress版本切换成\_checkpointed

上一篇：[MySQL · 答疑解惑 · 物理备份死锁分析](https://www.kancloud.cn/taobaomysql/monthly/117960)下一篇：[MySQL · 特性分析 · drop table的优化](https://www.kancloud.cn/taobaomysql/monthly/117962)

---------------------------------------------------


原网址: [访问](https://www.kancloud.cn/taobaomysql/monthly/117961)

创建于: 2020-05-27 15:26:47

目录: default

标签: `www.kancloud.cn`

