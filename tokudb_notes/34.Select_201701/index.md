---
title: TokuDB · 源码分析 · 一条query语句的执行过程
category: default
tags: 
  - mysql.taobao.org
created_at: 2020-05-28 14:05:40
original_url: http://mysql.taobao.org/monthly/2017/01/10/
---

[

# 数据库内核月报 － 2017 / 01

](http://mysql.taobao.org/monthly/2017/01)

[‹](http://mysql.taobao.org/monthly/2017/01/09/)

*   [当期文章](#)

## TokuDB · 源码分析 · 一条query语句的执行过程

Mysql是基于代价cost来选择索引，如果一个表有好几个索引，optimizer会分别计算每个索引访问的代价，选择代价最小的索引进行访问，这个索引也被称为access path。

## Pickup index

Mysql在执行query语句的时候会在server层计算每个可选索引的代价，并选择代价最小的索引作为访问路径（access path）去引擎读取数据。 server层的handler类为引擎层提供一个框架来计算索引的代价。

1.  scan\_time：计算全表扫描需要执行时间
2.  records\_in\_range：计算索引在search condition范围内包含多少行数据
3.  read\_time：计算索引range query执行时间

Tokudb的records\_in\_range：根据search condition区间的大小做不同的处理。

*   如果search condition为NULL，并且start\_key和end\_key均为NULL，这个函数调用estimate\_num\_rows去读ft的内存统计信息in\_memory\_stats.numrows，得到索引有多少个<key,value>pair，也即unique key的个数。因为mysql的二级索引的key都会拼上pk，到了索引层所有的key都是unique的。
*   把search condition的start\_key和end\_key封装成ft的key，调用ft的keys\_range64函数计算落在<start\_key,end\_key>区间的key个数。less表示小于start\_key的个数，equal1表示等于start\_key的个数，middle表示大于等于start key且小于end\_key的个数，equal2表示等于end\_key的个数，greater表示大于等于end\_key的个数。这个函数递归计算，代码比较多，但是不复杂。
*   如果start\_key和end\_key落在同一个basement节点，就读取那个basement节点并把满足条件的记录条数返回给server层。
*   如果search conditionn跨越多个basement节点，就需要把索引中存储的键值key个数（in\_memory\_stats.numrows）分摊到从root到leaf路径上的每一层节点上，这样得到每一层节点的权重。然后把start\_key到end\_key区间在每一层节点上的权重累加起来得到区间的记录条数。

当start\_key和end\_key不在同一个basement节点时，keys\_range64是通过估算得到记录条数的。 估算的值受FT tree layout影响，FT tree可以是瘦高的，也可以是扁平的，不同的layout计算的结果可能会差别比较大。 而且，tokudb的键值key个数（in\_memory\_stats.numrows）也是个统计值，是每次在leaf节点做msn apply更新的，这个值也可能不准确。

```php
ha_rows ha_tokudb::records_in_range(uint keynr, key_range* start_key, key_range* end_key) {
    DBT *pleft_key, *pright_key;
    DBT left_key, right_key;
    ha_rows ret_val = HA_TOKUDB_RANGE_COUNT;
    DB *kfile = share->key_file[keynr];
    uint64_t rows = 0;
    int error;

    // get start_rows and end_rows values so that we can estimate range
    // when calling key_range64, the only value we can trust is the value for less
    // The reason is that the key being passed in may be a prefix of keys in the DB
    // As a result, equal may be 0 and greater may actually be equal+greater
    // So, we call key_range64 on the key, and the key that is after it.
    if (!start_key && !end_key) {
        error = estimate_num_rows(kfile, &rows, transaction);
        if (error) {
            ret_val = HA_TOKUDB_RANGE_COUNT;
            goto cleanup;
        }
        ret_val = (rows <= 1) ? 1 : rows;
        goto cleanup;
    }
    if (start_key) {
        uchar inf_byte = (start_key->flag == HA_READ_KEY_EXACT) ? COL_NEG_INF : COL_POS_INF;
        pack_key(&left_key, keynr, key_buff, start_key->key, start_key->length, inf_byte);
        pleft_key = &left_key;
    } else {
        pleft_key = NULL;
    }
    if (end_key) {
        uchar inf_byte = (end_key->flag == HA_READ_BEFORE_KEY) ? COL_NEG_INF : COL_POS_INF;
        pack_key(&right_key, keynr, key_buff2, end_key->key, end_key->length, inf_byte);
        pright_key = &right_key;
    } else {
        pright_key = NULL;
    }
    // keys_range64 can not handle a degenerate range (left_key > right_key), so we filter here
    if (pleft_key && pright_key && tokudb_cmp_dbt_key(kfile, pleft_key, pright_key) > 0) {
        rows = 0;
    } else {
        uint64_t less, equal1, middle, equal2, greater;
        bool is_exact;
        error = kfile->keys_range64(kfile, transaction, pleft_key, pright_key,
                                    &less, &equal1, &middle, &equal2, &greater, &is_exact);
        if (error) {
            ret_val = HA_TOKUDB_RANGE_COUNT;
            goto cleanup;
        }
        rows = middle;
    }

    // MySQL thinks a return value of 0 means there are exactly 0 rows
    // Therefore, always return non-zero so this assumption is not made
    ret_val = (ha_rows) (rows <= 1 ? 1 : rows);

cleanup:
    if (tokudb_debug & TOKUDB_DEBUG_RETURN) {
        TOKUDB_HANDLER_TRACE("return %" PRIu64 " %" PRIu64, (uint64_t) ret_val, rows);
    }
    DBUG_RETURN(ret_val);
}
```

用户创建的表可能只有数据没有索引，也可能有好几个索引。optimizer选择索引的过程：用search condition找出可用索引的集合，然后尝试用每个可选索引计算代价，也就是计算read\_time。这个值是根据records\_in\_range返回的记录条数计算出来的。 Optimizer会选择代价最小的索引（在server层被称为access path）去引擎里面取数据，访问access path的方式可能是index point query/index range query也可能是full index scan，也可能是full table scan。

## Read data

选定了索引，server层会把索引信息（keynr）传给引擎，在引擎层创建索引的cursor去读取数据。一般来说，对于full table scan引擎层会隐式转成pk index scan。

### Full table scan

我们先来看一下full table scan的函数：

*   rnd\_init: 调用index\_init隐式转为pk index scan，并锁表
*   rnd\_next：调用get\_next读取下一行数据，这个函数后面会详细讨论
*   rnd\_end：结束scan

```plain
int ha_tokudb::rnd_init(bool scan) {
    int error = 0;
    range_lock_grabbed = false;
    error = index_init(MAX_KEY, 0);
    if (error) { goto cleanup;}

    if (scan) {
        error = prelock_range(NULL, NULL);
        if (error) { goto cleanup; }
        range_lock_grabbed = true;
    }

    error = 0;
cleanup:
    if (error) {
        index_end();
        last_cursor_error = error;
    }
    TOKUDB_HANDLER_DBUG_RETURN(error);
}

int ha_tokudb::rnd_next(uchar * buf) {
    ha_statistic_increment(&SSV::ha_read_rnd_next_count);
    int error = get_next(buf, 1, NULL, false);
    TOKUDB_HANDLER_DBUG_RETURN(error);
}

int ha_tokudb::rnd_end() {
    range_lock_grabbed = false;
    TOKUDB_HANDLER_DBUG_RETURN(index_end());
}
```

### Index scan

#### ICP

当search condition非空时，server层可能会选择使用ICP (index condition pushdown)，把search condition下推到引擎层来做过滤。

*   keyno\_arg：server层选的索引index：keyno\_arg
*   idx\_cond\_arg：server层的search condition，可能包含过滤条件

```plain
Item* ha_tokudb::idx_cond_push(uint keyno_arg, Item* idx_cond_arg) {
    toku_pushed_idx_cond_keyno = keyno_arg;
    toku_pushed_idx_cond = idx_cond_arg;
    return idx_cond_arg;
}
```

#### Index init

前面提到full table scan会隐式转为pk index scan，在rnd\_init中调用index\_init把tokudb\_active\_index设置为primary\_key。 Handler类成员active\_index表示当前索引index，这个值等于MAX\_KEY（64）表示full table scan。 Tokudb类成员tokudb\_active\_index表示tokudb当前的索引index，一般来说这个值跟active\_index是一样的。 Full table scan是个例外，active\_index等于MAX\_KEY，tokudb\_active\_index等于primary\_key。 Index\_init中最重要的工作就是创建cursor，并且重置bulk fetch信息。bulk fetch将在get\_next函数中详细讨论。

```php
int ha_tokudb::index_init(uint keynr, bool sorted) {
    int error;
    THD* thd = ha_thd();

    /*
       Under some very rare conditions (like full joins) we may already have
       an active cursor at this point
     */
    if (cursor) {
        int r = cursor->c_close(cursor);
        assert(r==0);
        remove_from_trx_handler_list();
    }
    active_index = keynr;

    if (active_index < MAX_KEY) {
        DBUG_ASSERT(keynr <= table->s->keys);
    } else {
        DBUG_ASSERT(active_index == MAX_KEY);
        keynr = primary_key;
    }
    tokudb_active_index = keynr;

#if TOKU_CLUSTERING_IS_COVERING
    if (keynr < table->s->keys && table->key_info[keynr].option_struct->clustering)
        key_read = false;
#endif

    last_cursor_error = 0;
    range_lock_grabbed = false;
    range_lock_grabbed_null = false;
    DBUG_ASSERT(share->key_file[keynr]);
    cursor_flags = get_cursor_isolation_flags(lock.type, thd);
    if (use_write_locks) {
        cursor_flags |= DB_RMW;
    }
    if (get_disable_prefetching(thd)) {
        cursor_flags |= DBC_DISABLE_PREFETCHING;
    }
    if ((error = share->key_file[keynr]->cursor(share->key_file[keynr], transaction, &cursor, cursor_flags))) {
        last_cursor_error = error;
        cursor = NULL;             // Safety
        goto exit;
    }
    cursor->c_set_check_interrupt_callback(cursor, tokudb_killed_thd_callback, thd);
    memset((void *) &last_key, 0, sizeof(last_key));

    add_to_trx_handler_list();

    if (thd_sql_command(thd) == SQLCOM_SELECT) {
        set_query_columns(keynr);
        unpack_entire_row = false;
    }
    else {
        unpack_entire_row = true;
    }
    invalidate_bulk_fetch();
    doing_bulk_fetch = false;
    maybe_index_scan = false;
    error = 0;
exit:
    TOKUDB_HANDLER_DBUG_RETURN(error);
}
```

#### Prepare index

初始化cursor之后，server层会调下面四个函数之一去拿<start\_key,end\_key>区间的range锁。

*   prepare\_index\_scan
*   prepare\_index\_key\_scan
*   prepare\_range\_scan
*   read\_range\_first

这四个函数都是调用prelock\_range去拿rangelock。

*   prepare\_index\_scan拿的是<负无穷,正无穷>区间的rangelock，其实就是锁表。
*   prepare\_index\_key\_scan只拿对应key的rangelock，
*   prepare\_range\_scan和read\_range\_first都是拿<start\_key,end\_key>区间的rangelock。前者是处理reverse index range scan的，后者是处理index range scan的。

Start\_key和end\_key就是server层传下来的range区间的起点和终点，是server层的数据结构，prelock\_range会生成相应的索引key并获取索引key的rangelock。

Full table scan的时候，rnd\_init直接调用prelock\_range拿<负无穷,正无穷>区间的rangelock，也就是锁表。 由于full table scan转pk index scan是在引擎内部做隐式转换，sever层并不知道，不走prepare\_index\_scan。

Rangelock的机制在[之前的月报](http://mysql.taobao.org/monthly/2015/04/03/)有提到。 如果既不是SERIALIZABLE隔离级别，也不是为写操作读取数据，调用prelock\_range是不会真的去拿rangelock锁的。此种情况的rangelock锁是在query成功返回前拿的，防止并发事务更新相应的数据。

```php
static int
c_set_bounds(DBC *dbc, const DBT *left_key, const DBT *right_key, bool pre_acquire, int out_of_range_error) {
    if (out_of_range_error != DB_NOTFOUND &&
        out_of_range_error != TOKUDB_OUT_OF_RANGE &&
        out_of_range_error != 0) {
        return toku_ydb_do_error(
            dbc->dbp->dbenv,
            EINVAL,
            "Invalid out_of_range_error [%d] for %s\n",
            out_of_range_error,
            __FUNCTION__
            );
    }
    if (left_key == toku_dbt_negative_infinity() && right_key == toku_dbt_positive_infinity()) {
        out_of_range_error = 0;
    }
    DB *db = dbc->dbp;
    DB_TXN *txn = dbc_struct_i(dbc)->txn;
    HANDLE_PANICKED_DB(db);
    toku_ft_cursor_set_range_lock(dbc_ftcursor(dbc), left_key, right_key,
                                   (left_key == toku_dbt_negative_infinity()),
                                   (right_key == toku_dbt_positive_infinity()),
                                   out_of_range_error);
    if (!db->i->lt || !txn || !pre_acquire)
        return 0;
    //READ_UNCOMMITTED and READ_COMMITTED transactions do not need read locks.
    if (!dbc_struct_i(dbc)->rmw && dbc_struct_i(dbc)->iso != TOKU_ISO_SERIALIZABLE)
        return 0;

    toku::lock_request::type lock_type = dbc_struct_i(dbc)->rmw ?
        toku::lock_request::type::WRITE : toku::lock_request::type::READ;
    int r = toku_db_get_range_lock(db, txn, left_key, right_key, lock_type);
    return r;
}
```

#### Read index

如果server层指定了range的start\_key和end\_key，handler的执行框架会根据execution plan指定的方式访问索引数据。

第一行数据的访问方式：

*   Index point query：server层直接调用index\_read
*   Index range scan且start\_key非空：server层通常是调用read\_range\_first函数读取第一行数据。read\_range\_first最终也是调用index\_read
*   Index range scan且start\_key为空：server层直接调用index\_first
*   Index reverse range scan且end\_key非空：server层通常是调用index\_read读数据
*   Index reverse range scan且end\_key为空：server层直接调用index\_last
*   Full index scan：server层直接调用index\_first
*   Reverse full index scan：server层直接调用index\_last

Index\_read函数比较长，举几个常见的场景来说明 1) index point query:

*   HA\_READ\_KEY\_EXACT

2) index range query：

*   HA\_READ\_AFTER\_KEY：处理大于start\_key的情况
*   HA\_READ\_KEY\_OR\_NEXT：处理大于等于start\_key的情况

3) reverse index range query：

*   HA\_READ\_BEFORE\_KEY：处理小于end\_key的情况
*   HA\_READ\_PREFIX\_LAST\_OR\_PREV：处理小于等于end\_key的情况

```plain
int ha_tokudb::index_read(
    uchar* buf,
    const uchar* key,
    uint key_len,
    enum ha_rkey_function find_flag) {

    invalidate_bulk_fetch();

    DBT row;
    DBT lookup_key;
    int error = 0;
    uint32_t flags = 0;
    THD* thd = ha_thd();
    tokudb_trx_data* trx = (tokudb_trx_data*)thd_get_ha_data(thd, tokudb_hton);
    struct smart_dbt_info info;
    struct index_read_info ir_info;

    HANDLE_INVALID_CURSOR();

    // if we locked a non-null key range and we now have a null key, then
    // remove the bounds from the cursor
    if (range_lock_grabbed &&
        !range_lock_grabbed_null &&
        index_key_is_null(table, tokudb_active_index, key, key_len)) {
        range_lock_grabbed = range_lock_grabbed_null = false;
        cursor->c_remove_restriction(cursor);
    }

    ha_statistic_increment(&SSV::ha_read_key_count);
    memset((void *) &row, 0, sizeof(row));

    info.ha = this;
    info.buf = buf;
    info.keynr = tokudb_active_index;

    ir_info.smart_dbt_info = info;
    ir_info.cmp = 0;

    flags = SET_PRELOCK_FLAG(0);
    switch (find_flag) {
    case HA_READ_KEY_EXACT: /* Find first record else error */ {
        pack_key(&lookup_key, tokudb_active_index, key_buff3, key, key_len, COL_NEG_INF);
        DBT lookup_bound;
        pack_key(&lookup_bound, tokudb_active_index, key_buff4, key, key_len, COL_POS_INF);
        ir_info.orig_key = &lookup_key;
        error = cursor->c_getf_set_range_with_bound(cursor, flags, &lookup_key, &lookup_bound, SMART_DBT_IR_CALLBACK(key_read), &ir_info);
        if (ir_info.cmp) {
            error = DB_NOTFOUND;
        }
        break;
    }
    case HA_READ_AFTER_KEY: /* Find next rec. after key-record */
        pack_key(&lookup_key, tokudb_active_index, key_buff3, key, key_len, COL_POS_INF);
        error = cursor->c_getf_set_range(cursor, flags, &lookup_key, SMART_DBT_CALLBACK(key_read), &info);
        break;
    case HA_READ_BEFORE_KEY: /* Find next rec. before key-record */
        pack_key(&lookup_key, tokudb_active_index, key_buff3, key, key_len, COL_NEG_INF);
        error = cursor->c_getf_set_range_reverse(cursor, flags, &lookup_key, SMART_DBT_CALLBACK(key_read), &info);
        break;
    case HA_READ_KEY_OR_NEXT: /* Record or next record */
        pack_key(&lookup_key, tokudb_active_index, key_buff3, key, key_len, COL_NEG_INF);
        error = cursor->c_getf_set_range(cursor, flags, &lookup_key, SMART_DBT_CALLBACK(key_read), &info);
        break;
    //
    // This case does not seem to ever be used, it is ok for it to be slow
    //
    case HA_READ_KEY_OR_PREV: /* Record or previous */
        pack_key(&lookup_key, tokudb_active_index, key_buff3, key, key_len, COL_NEG_INF);
        ir_info.orig_key = &lookup_key;
        error = cursor->c_getf_set_range(cursor, flags, &lookup_key, SMART_DBT_IR_CALLBACK(key_read), &ir_info);
        if (error == DB_NOTFOUND) {
            error = cursor->c_getf_last(cursor, flags, SMART_DBT_CALLBACK(key_read), &info);
        }
        else if (ir_info.cmp) {
            error = cursor->c_getf_prev(cursor, flags, SMART_DBT_CALLBACK(key_read), &info);
        }
        break;
    case HA_READ_PREFIX_LAST_OR_PREV: /* Last or prev key with the same prefix */
        pack_key(&lookup_key, tokudb_active_index, key_buff3, key, key_len, COL_POS_INF);
        error = cursor->c_getf_set_range_reverse(cursor, flags, &lookup_key, SMART_DBT_CALLBACK(key_read), &info);
        break;
    case HA_READ_PREFIX_LAST:
        pack_key(&lookup_key, tokudb_active_index, key_buff3, key, key_len, COL_POS_INF);
        ir_info.orig_key = &lookup_key;
        error = cursor->c_getf_set_range_reverse(cursor, flags, &lookup_key, SMART_DBT_IR_CALLBACK(key_read), &ir_info);
        if (ir_info.cmp) {
            error = DB_NOTFOUND;
        }
        break;
    default:
        TOKUDB_HANDLER_TRACE("unsupported:%d", find_flag);
        error = HA_ERR_UNSUPPORTED;
        break;
    }
    error = handle_cursor_error(error,HA_ERR_KEY_NOT_FOUND,tokudb_active_index);
    if (!error && !key_read && tokudb_active_index != primary_key && !key_is_clustering(&table->key_info[tokudb_active_index])) {
        error = read_full_row(buf);
    }
    trx->stmt_progress.queried++;
    track_progress(thd);

cleanup:
    TOKUDB_HANDLER_DBUG_RETURN(error);
}
```

Index\_read在调用ydb\_cursor.cc中的回调函数时，flags参数初始化为0。ydb\_cursor.cc中注册的回调函数会检查tokudb\_cursor->rmw标记，如果tokudb\_cursor->rmw是0，并且不是SERIALIZABLE隔离级别，函数 query\_context\_with\_input\_init会设置context的do\_locking字段，告诉toku\_ft\_cursor\_set\_range在成功返回前去拿rangelock。

```plain
static int
c_getf_set_range_with_bound(DBC *c, uint32_t flag, DBT *key, DBT *key_bound, YDB_CALLBACK_FUNCTION f, void *extra) {
    HANDLE_PANICKED_DB(c->dbp);
    HANDLE_CURSOR_ILLEGAL_WORKING_PARENT_TXN(c);

    int r = 0;
    QUERY_CONTEXT_WITH_INPUT_S context; //Describes the context of this query.
    query_context_with_input_init(&context, c, flag, key, NULL, f, extra);
    while (r == 0) {
        //toku_ft_cursor_set_range will call c_getf_set_range_callback(..., context) (if query is successful)
        r = toku_ft_cursor_set_range(dbc_ftcursor(c), key, key_bound, c_getf_set_range_callback, &context);
        if (r == DB_LOCK_NOTGRANTED) {
            r = toku_db_wait_range_lock(context.base.db, context.base.txn, &context.base.request);
        } else {
            break;
        }
    }
    query_context_base_destroy(&context.base);
    return r;
}

static int
c_getf_set_range_callback(uint32_t keylen, const void *key, uint32_t vallen, const void *val, void *extra, bool lock_only) {
    QUERY_CONTEXT_WITH_INPUT super_context = (QUERY_CONTEXT_WITH_INPUT) extra;
    QUERY_CONTEXT_BASE       context       = &super_context->base;

    int r;
    DBT found_key = { .data = (void *) key, .size = keylen };

    //Lock:
    //  left(key,val)  = (input_key, -infinity)
    //  right(key) = found ? found_key : infinity
    //  right(val) = found ? found_val : infinity
    if (context->do_locking) {
        const DBT *left_key = super_context->input_key;
        const DBT *right_key = key != NULL ? &found_key : toku_dbt_positive_infinity();
        r = toku_db_start_range_lock(context->db, context->txn, left_key, right_key, query_context_determine_lock_type(context), &context->request);
    } else {
        r = 0;
    }

    //Call application-layer callback if found and locks were successfully obtained.
    if (r==0 && key!=NULL && !lock_only) {
        DBT found_val = { .data = (void *) val, .size = vallen };
        context->r_user_callback = context->f(&found_key, &found_val, context->f_extra);
        r = context->r_user_callback;
    }

    //Give ft-layer an error (if any) to return from toku_ft_cursor_set_range
    return r;
}
```

#### Get next

取到第一行数据后，server层会根据execution plan来调用 index\_next（index\_next\_same）或者index\_prev来取后面的记录。

*   index\_next\_same：读取相同index的下一个记录。
*   index\_next：读取range区间内的下一个记录，如果设置ICP，还会对找到的记录进行过滤条件匹配。
*   index\_prev：读取range区间内的上一个记录，如果设置ICP，还会对找到的记录进行过滤条件匹配。

还有两个读取数据的方法：

*   index\_first：读取index的第一条记录
*   index\_last：读取index最后一条记录

这几个函数比较简单，这里只分析index\_next函数，感兴趣的朋友可以自行分析其余的函数。 index\_next直接调用get\_next函数读取下一条记录。

```plain
int ha_tokudb::index_next(uchar * buf) {
    TOKUDB_HANDLER_DBUG_ENTER("");
    ha_statistic_increment(&SSV::ha_read_next_count);
    int error = get_next(buf, 1, NULL, key_read);
    TOKUDB_HANDLER_DBUG_RETURN(error);
}
```

#### Bulk fetch

Tokudb为range query做了一个优化，被称作bulk fetch。对当前basement节点上落在range区间的key进行批量读取，一次msg apply多次读key操作，同时也减轻leaf节点读写锁争抢，避免频繁拿锁放锁。 Tokudb为提供bulk fetch功能，增加了如下几个数据成员：

*   doing\_bulk\_fetch：标记是否正在进行bulk fetch
*   range\_query\_buff：缓存批量读取数据的buffer
*   size\_range\_query\_buff：range\_query\_buff的malloc\_size
*   bytes\_used\_in\_range\_query\_buff：range\_query\_buff的实际size
*   curr\_range\_query\_buff\_offset：range\_query\_buff的当前位置
*   bulk\_fetch\_iteration和rows\_fetched\_using\_bulk\_fetch是统计数据，控制批量大小

```plain
class ha_tokudb : public handler {
private:
    ...
    uchar* range_query_buff; // range query buffer
    uint32_t size_range_query_buff; // size of the allocated range query buffer
    uint32_t bytes_used_in_range_query_buff; // number of bytes used in the range query buffer
    uint32_t curr_range_query_buff_offset; // current offset into the range query buffer for queries to read
    uint64_t bulk_fetch_iteration;
    uint64_t rows_fetched_using_bulk_fetch;
    bool doing_bulk_fetch;
    ...
};
```

Get\_next函数首先判读是否可以从当前的bulk fetch bufffer中读取数据，判读的标准是bytes\_used\_in\_range\_query\_buff - curr\_range\_query\_buff\_offset > 0，表示bulk fetch buffer有数据可以读取。

1.  如果条件成立，调用read\_data\_from\_range\_query\_buff直接从bulk fetch buffer中读数据。
2.  如果bulk fetch buffer没有数据可读了，需要检查icp\_went\_out\_of\_range判断是否已超出range范围，那样的话表示没有更多数据，可以直接返回。
3.  如果前面两个条件都不满足，需要调用cursor读取后面的数据。如果是bulk fetch的情况，需要调用invalidate\_bulk\_fetch重置bulf fetch的数据结构。

如果用户禁掉bulk fetch的功能，该如何处理呢？禁掉bulk fetch，第1和第2两个条件都不满足，直接执行invalidate\_bulk\_fetch，然后检查doing\_bulk\_fetch标记为false，调用cursor读取数据。这部分比较简单，请读者自行分析。

bulk fetch的处理跟非bulk fetch的处理类似，最大的区别在于cursor->c\_getf\_XXX方法的回调函数和回调函数参数。它们分别被设置为smart\_dbt\_bf\_callback和struct smart\_dbt\_bf\_info结构的指针。

smart\_dbt\_bf\_info结构告诉回调函数如何缓存当前数据，并且如何读取下一行数据。

*   ha：tokudb handler指针
*   need\_val：bulk fetch buffer是否要缓存value，对于pk和cluster index情况设置成true，其他情况为false
*   director：读取数据的方向。1表示next，-1表示prev
*   thd：server层的线程指针
*   buf：server层提供的buffer
*   key\_to\_compare：比较key，只有index\_next\_same需要设置这个参数

```plain
typedef struct smart_dbt_bf_info {
    ha_tokudb* ha;
    bool need_val;
    int direction;
    THD* thd;
    uchar* buf;
    DBT* key_to_compare;
} *SMART_DBT_BF_INFO
```

一个批量读取完成后，get\_next会调用read\_data\_from\_range\_query\_buff，从bulk fetch buffer中取数据。

Get\_next成功读取一行数据后，需要判断是否是需要回表读取row。对于pk和cluster index的情况，index中存储了完整数据不需要回表。

```plain
int ha_tokudb::get_next(
    uchar* buf,
    int direction,
    DBT* key_to_compare,
    bool do_key_read) {

    int error = 0;
    HANDLE_INVALID_CURSOR();

    if (maybe_index_scan) {
        maybe_index_scan = false;
        if (!range_lock_grabbed) {
            error = prepare_index_scan();
        }
    }

    if (!error) {
        uint32_t flags = SET_PRELOCK_FLAG(0);

        // we need to read the val of what we retrieve if
        // we do NOT have a covering index AND we are using a clustering secondary
        // key
        bool need_val =
            (do_key_read == 0) &&
            (tokudb_active_index == primary_key ||
             key_is_clustering(&table->key_info[tokudb_active_index]));

        if ((bytes_used_in_range_query_buff -
             curr_range_query_buff_offset) > 0) {
            error = read_data_from_range_query_buff(buf, need_val, do_key_read);
        } else if (icp_went_out_of_range) {
            icp_went_out_of_range = false;
            error = HA_ERR_END_OF_FILE;
        } else {
            invalidate_bulk_fetch();
            if (doing_bulk_fetch) {
                struct smart_dbt_bf_info bf_info;
                bf_info.ha = this;
                // you need the val if you have a clustering index and key_read is not 0;
                bf_info.direction = direction;
                bf_info.thd = ha_thd();
                bf_info.need_val = need_val;
                bf_info.buf = buf;
                bf_info.key_to_compare = key_to_compare;
                //
                // call c_getf_next with purpose of filling in range_query_buff
                //
                rows_fetched_using_bulk_fetch = 0;
                // it is expected that we can do ICP in the smart_dbt_bf_callback
                // as a result, it's possible we don't return any data because
                // none of the rows matched the index condition. Therefore, we need
                // this while loop. icp_out_of_range will be set if we hit a row that
                // the index condition states is out of our range. When that hits,
                // we know all the data in the buffer is the last data we will retrieve
                while (bytes_used_in_range_query_buff == 0 &&
                       !icp_went_out_of_range && error == 0) {
                    if (direction > 0) {
                        error =
                            cursor->c_getf_next(
                                cursor,
                                flags,
                                smart_dbt_bf_callback,
                                &bf_info);
                    } else {
                        error =
                            cursor->c_getf_prev(
                                cursor,
                                flags,
                                smart_dbt_bf_callback,
                                &bf_info);
                    }
                }
                // if there is no data set and we went out of range,
                // then there is nothing to return
                if (bytes_used_in_range_query_buff == 0 &&
                    icp_went_out_of_range) {
                    icp_went_out_of_range = false;
                    error = HA_ERR_END_OF_FILE;
                }
                if (bulk_fetch_iteration < HA_TOKU_BULK_FETCH_ITERATION_MAX) {
                    bulk_fetch_iteration++;
                }

                error =
                    handle_cursor_error(
                        error,
                        HA_ERR_END_OF_FILE,
                        tokudb_active_index);
                if (error) {
                    goto cleanup;
                }

                //
                // now that range_query_buff is filled, read an element
                //
                error =
                    read_data_from_range_query_buff(buf, need_val, do_key_read);
            } else {
                struct smart_dbt_info info;
                info.ha = this;
                info.buf = buf;
                info.keynr = tokudb_active_index;

                if (direction > 0) {
                    error =
                        cursor->c_getf_next(
                            cursor,
                            flags,
                            SMART_DBT_CALLBACK(do_key_read),
                            &info);
                } else {
                    error =
                        cursor->c_getf_prev(
                            cursor,
                            flags,
                            SMART_DBT_CALLBACK(do_key_read),
                            &info);
                }
                error =
                    handle_cursor_error(
                        error,
                        HA_ERR_END_OF_FILE,
                        tokudb_active_index);
            }
        }
    }

    //
    // at this point, one of two things has happened
    // either we have unpacked the data into buf, and we
    // are done, or we have unpacked the primary key
    // into last_key, and we use the code below to
    // read the full row by doing a point query into the
    // main table.
    //
    if (!error &&
        !do_key_read &&
        (tokudb_active_index != primary_key) &&
        !key_is_clustering(&table->key_info[tokudb_active_index])) {
        error = read_full_row(buf);
    }

    if (!error) {
        THD *thd = ha_thd();
        tokudb_trx_data* trx =
            static_cast<tokudb_trx_data*>(thd_get_ha_data(thd, tokudb_hton));
        trx->stmt_progress.queried++;
        track_progress(thd);
        if (thd_killed(thd))
            error = ER_ABORTING_CONNECTION;
    }
cleanup:
    return error;
}
```

下面我们一起看看回调函数smart\_dbt\_bf\_callback的处理。这个函数是fill\_range\_query\_buf简单封装，当成功读取一行索引数据后，把结果缓存到bulk fetch buffer中，并继续读取下一行数据。

```php
static int smart_dbt_bf_callback(
    DBT const* key,
    DBT const* row,
    void* context) {
    SMART_DBT_BF_INFO info = (SMART_DBT_BF_INFO)context;
    return
        info->ha->fill_range_query_buf(
            info->need_val,
            key,
            row,
            info->direction,
            info->thd,
            info->buf,
            info->key_to_compare);
}
```

接下来，让我们把目光聚焦在fill\_range\_query\_buf函数。参数key和value是当前读取到索引key和value；其余的参数是从smart\_dbt\_bf\_info中结构提取出来的server层调用时指定的信息。

如果指定了key\_to\_compare，需要判断当前读取的key是否等于key\_to\_compare，因为二级索引的key后面拼了pk，所以这里做的是前缀比较。如果前缀不匹配，表示已经读到一个新key，设置icp\_went\_out\_of\_range并退出。

如果server层设置了ICP信息，需要判断当前读取的索引key是否在range范围内。 一般来说，判断是否在range范围内的方法是跟prelocked\_right\_range（range scan）或者prelocked\_left\_range（reverse range scan）比较的。 而ICP的情况下，判断是否在range范围内是跟end\_range做比较的。 对于索引key不在range范围内的情况，设置icp\_went\_out\_of\_range并返回。

如果当前读到的索引key是在range范围内，ICP的情况还要做过滤条件检查。如果满足过滤条件，就存储到bulk fetch buffer中；不满足过滤条件，就跳过这条记录取下一条。

把key存储到bulk fetch buffer中时，需要检查need\_val。为true时，先存key后存value；否则，只存key。 Value要存的数据可能是整个row，可能是set\_query\_columns函数记录的那些字段的数据。如果是第二种情况，需要把相应字段的数据提取出来。 Bulk fetch buffer中的数据按照一定格式存储，先存4个字节的size，接着存data。 当前key的存储位置是在bytes\_used\_in\_range\_query\_buff偏移位置。 把key/value数据缓存到bulk fetch buffer中以后，还需要更新bytes\_used\_in\_range\_query\_buff指向下一次写入的位置。

对于非ICP的情况，在fill\_range\_query\_buf函数的最后判断是否超出range范围。这里跟prelocked\_right\_range（range scan）或者prelocked\_left\_range（reverse range scan）做比较。这两个值是在prelock\_range函数设置的，也是rangelock的范围。

如果当前读取的key属于range范围内，需要继续读取下一条数据到bulk fetch buffer中，fill\_range\_query\_buf返回TOKUDB\_CURSOR\_CONTINUE告诉toku\_ft\_search继续读取当前basement节点的下一条数据。 bulk fetch不能跨越basement节点，因为无法保证其他basement节点上是否做过msg apply。

```plain
int ha_tokudb::fill_range_query_buf(
    bool need_val,
    DBT const* key,
    DBT const* row,
    int direction,
    THD* thd,
    uchar* buf,
    DBT* key_to_compare) {

    int error;
    //
    // first put the value into range_query_buf
    //
    uint32_t size_remaining =
        size_range_query_buff - bytes_used_in_range_query_buff;
    uint32_t size_needed;
    uint32_t user_defined_size = tokudb::sysvars::read_buf_size(thd);
    uchar* curr_pos = NULL;

    if (key_to_compare) {
        int cmp = tokudb_prefix_cmp_dbt_key(
            share->key_file[tokudb_active_index],
            key_to_compare,
            key);
        if (cmp) {
            icp_went_out_of_range = true;
            error = 0;
            goto cleanup;
        }
    }

    // if we have an index condition pushed down, we check it
    if (toku_pushed_idx_cond &&
        (tokudb_active_index == toku_pushed_idx_cond_keyno)) {
        unpack_key(buf, key, tokudb_active_index);
        enum icp_result result =
            toku_handler_index_cond_check(toku_pushed_idx_cond);

        // If we have reason to stop, we set icp_went_out_of_range and get out
        // otherwise, if we simply see that the current key is no match,
        // we tell the cursor to continue and don't store
        // the key locally
        if (result == ICP_OUT_OF_RANGE || thd_killed(thd)) {
            icp_went_out_of_range = true;
            error = 0;
            DEBUG_SYNC(ha_thd(), "tokudb_icp_asc_scan_out_of_range");
            goto cleanup;
        } else if (result == ICP_NO_MATCH) {
            // if we are performing a DESC ICP scan and have no end_range
            // to compare to stop using ICP filtering as there isn't much more
            // that we can do without going through contortions with remembering
            // and comparing key parts.
            if (!end_range &&
                direction < 0) {

                cancel_pushed_idx_cond();
                DEBUG_SYNC(ha_thd(), "tokudb_icp_desc_scan_invalidate");
            }

            error = TOKUDB_CURSOR_CONTINUE;
            goto cleanup;
        }
    }

    // at this point, if ICP is on, we have verified that the key is one
    // we are interested in, so we proceed with placing the data
    // into the range query buffer

    if (need_val) {
        if (unpack_entire_row) {
            size_needed = 2*sizeof(uint32_t) + key->size + row->size;
        } else {
            // this is an upper bound
            size_needed =
                // size of key length
                sizeof(uint32_t) +
                // key and row
                key->size + row->size +
                // lengths of varchars stored
                num_var_cols_for_query * (sizeof(uint32_t)) +
                // length of blobs
                sizeof(uint32_t);
        }
    } else {
        size_needed = sizeof(uint32_t) + key->size;
    }
    if (size_remaining < size_needed) {
        range_query_buff =
            static_cast<uchar*>(tokudb::memory::realloc(
                static_cast<void*>(range_query_buff),
                bytes_used_in_range_query_buff + size_needed,
                MYF(MY_WME)));
        if (range_query_buff == NULL) {
            error = ENOMEM;
            invalidate_bulk_fetch();
            goto cleanup;
        }
        size_range_query_buff = bytes_used_in_range_query_buff + size_needed;
    }
    //
    // now we know we have the size, let's fill the buffer, starting with the key
    //
    curr_pos = range_query_buff + bytes_used_in_range_query_buff;

    *reinterpret_cast<uint32_t*>(curr_pos) = key->size;
    curr_pos += sizeof(uint32_t);
    memcpy(curr_pos, key->data, key->size);
    curr_pos += key->size;
    if (need_val) {
        if (unpack_entire_row) {
            *reinterpret_cast<uint32_t*>(curr_pos) = row->size;
            curr_pos += sizeof(uint32_t);
            memcpy(curr_pos, row->data, row->size);
            curr_pos += row->size;
        } else {
            // need to unpack just the data we care about
            const uchar* fixed_field_ptr = static_cast<const uchar*>(row->data);
            fixed_field_ptr += table_share->null_bytes;

            const uchar* var_field_offset_ptr = NULL;
            const uchar* var_field_data_ptr = NULL;

            var_field_offset_ptr =
                fixed_field_ptr +
                share->kc_info.mcp_info[tokudb_active_index].fixed_field_size;
            var_field_data_ptr =
                var_field_offset_ptr +
                share->kc_info.mcp_info[tokudb_active_index].len_of_offsets;

            // first the null bytes
            memcpy(curr_pos, row->data, table_share->null_bytes);
            curr_pos += table_share->null_bytes;
            // now the fixed fields
            //
            // first the fixed fields
            //
            for (uint32_t i = 0; i < num_fixed_cols_for_query; i++) {
                uint field_index = fixed_cols_for_query[i];
                memcpy(
                    curr_pos,
                    fixed_field_ptr + share->kc_info.cp_info[tokudb_active_index][field_index].col_pack_val,
                    share->kc_info.field_lengths[field_index]);
                curr_pos += share->kc_info.field_lengths[field_index];
            }

            //
            // now the var fields
            //
            for (uint32_t i = 0; i < num_var_cols_for_query; i++) {
                uint field_index = var_cols_for_query[i];
                uint32_t var_field_index =
                    share->kc_info.cp_info[tokudb_active_index][field_index].col_pack_val;
                uint32_t data_start_offset;
                uint32_t field_len;

                get_var_field_info(
                    &field_len,
                    &data_start_offset,
                    var_field_index,
                    var_field_offset_ptr,
                    share->kc_info.num_offset_bytes);
                memcpy(curr_pos, &field_len, sizeof(field_len));
                curr_pos += sizeof(field_len);
                memcpy(
                    curr_pos,
                    var_field_data_ptr + data_start_offset,
                    field_len);
                curr_pos += field_len;
            }

            if (read_blobs) {
                uint32_t blob_offset = 0;
                uint32_t data_size = 0;
                //
                // now the blobs
                //
                get_blob_field_info(
                    &blob_offset,
                    share->kc_info.mcp_info[tokudb_active_index].len_of_offsets,
                    var_field_data_ptr,
                    share->kc_info.num_offset_bytes);
                data_size =
                    row->size -
                    blob_offset -
                    static_cast<uint32_t>((var_field_data_ptr -
                        static_cast<const uchar*>(row->data)));
                memcpy(curr_pos, &data_size, sizeof(data_size));
                curr_pos += sizeof(data_size);
                memcpy(curr_pos, var_field_data_ptr + blob_offset, data_size);
                curr_pos += data_size;
            }
        }
    }

    bytes_used_in_range_query_buff = curr_pos - range_query_buff;
    assert_always(bytes_used_in_range_query_buff <= size_range_query_buff);

    //
    // now determine if we should continue with the bulk fetch
    // we want to stop under these conditions:
    //  - we overran the prelocked range
    //  - we are close to the end of the buffer
    //  - we have fetched an exponential amount of rows with
    //  respect to the bulk fetch iteration, which is initialized
    //  to 0 in index_init() and prelock_range().

    rows_fetched_using_bulk_fetch++;
    // if the iteration is less than the number of possible shifts on
    // a 64 bit integer, check that we haven't exceeded this iterations
    // row fetch upper bound.
    if (bulk_fetch_iteration < HA_TOKU_BULK_FETCH_ITERATION_MAX) {
        uint64_t row_fetch_upper_bound = 1LLU << bulk_fetch_iteration;
        assert_always(row_fetch_upper_bound > 0);
        if (rows_fetched_using_bulk_fetch >= row_fetch_upper_bound) {
            error = 0;
            goto cleanup;
        }
    }

    if (bytes_used_in_range_query_buff +
        table_share->rec_buff_length >
        user_defined_size) {
        error = 0;
        goto cleanup;
    }
    if (direction > 0) {
        // compare what we got to the right endpoint of prelocked range
        // because we are searching keys in ascending order
        if (prelocked_right_range_size == 0) {
            error = TOKUDB_CURSOR_CONTINUE;
            goto cleanup;
        }
        DBT right_range;
        memset(&right_range, 0, sizeof(right_range));
        right_range.size = prelocked_right_range_size;
        right_range.data = prelocked_right_range;
        int cmp = tokudb_cmp_dbt_key(
            share->key_file[tokudb_active_index],
            key,
            &right_range);
        error = (cmp > 0) ? 0 : TOKUDB_CURSOR_CONTINUE;
    } else {
        // compare what we got to the left endpoint of prelocked range
        // because we are searching keys in descending order
        if (prelocked_left_range_size == 0) {
            error = TOKUDB_CURSOR_CONTINUE;
            goto cleanup;
        }
        DBT left_range;
        memset(&left_range, 0, sizeof(left_range));
        left_range.size = prelocked_left_range_size;
        left_range.data = prelocked_left_range;
        int cmp = tokudb_cmp_dbt_key(
            share->key_file[tokudb_active_index],
            key,
            &left_range);
        error = (cmp < 0) ? 0 : TOKUDB_CURSOR_CONTINUE;
    }
cleanup:
    return error;
}
```

Bulk fetch buffer数据准备好了，我们就可以从read\_data\_from\_range\_query\_buff读取数据了。 Curr\_range\_query\_buff\_offset表示当前读取的位置。 首先读key信息。如果need\_value为true，还要读取data信息。可能读整行数据，也可能只需要读取函数set\_query\_columns设置的那些字段。 读取完成之后，调整curr\_range\_query\_buff\_offset指向下一次读取的位置。

```plain
int ha_tokudb::read_data_from_range_query_buff(uchar* buf, bool need_val, bool do_key_read) {
    // buffer has the next row, get it from there
    int error;
    uchar* curr_pos = range_query_buff+curr_range_query_buff_offset;
    DBT curr_key;
    memset((void *) &curr_key, 0, sizeof(curr_key));

    // get key info
    uint32_t key_size = *(uint32_t *)curr_pos;
    curr_pos += sizeof(key_size);
    uchar* curr_key_buff = curr_pos;
    curr_pos += key_size;

    curr_key.data = curr_key_buff;
    curr_key.size = key_size;

    // if this is a covering index, this is all we need
    if (do_key_read) {
        assert_always(!need_val);
        extract_hidden_primary_key(tokudb_active_index, &curr_key);
        read_key_only(buf, tokudb_active_index, &curr_key);
        error = 0;
    }
    // we need to get more data
    else {
        DBT curr_val;
        memset((void *) &curr_val, 0, sizeof(curr_val));
        uchar* curr_val_buff = NULL;
        uint32_t val_size = 0;
        // in this case, we don't have a val, we are simply extracting the pk
        if (!need_val) {
            curr_val.data = curr_val_buff;
            curr_val.size = val_size;
            extract_hidden_primary_key(tokudb_active_index, &curr_key);
            error = read_primary_key( buf, tokudb_active_index, &curr_val, &curr_key);
        }
        else {
            extract_hidden_primary_key(tokudb_active_index, &curr_key);
            // need to extract a val and place it into buf
            if (unpack_entire_row) {
                // get val info
                val_size = *(uint32_t *)curr_pos;
                curr_pos += sizeof(val_size);
                curr_val_buff = curr_pos;
                curr_pos += val_size;
                curr_val.data = curr_val_buff;
                curr_val.size = val_size;
                error = unpack_row(buf,&curr_val, &curr_key, tokudb_active_index);
            }
            else {
                if (!(hidden_primary_key && tokudb_active_index == primary_key)) {
                    unpack_key(buf,&curr_key,tokudb_active_index);
                }
                // read rows we care about

                // first the null bytes;
                memcpy(buf, curr_pos, table_share->null_bytes);
                curr_pos += table_share->null_bytes;

                // now the fixed sized rows
                for (uint32_t i = 0; i < num_fixed_cols_for_query; i++) {
                    uint field_index = fixed_cols_for_query[i];
                    Field* field = table->field[field_index];
                    unpack_fixed_field(
                        buf + field_offset(field, table),
                        curr_pos,
                        share->kc_info.field_lengths[field_index]
                        );
                    curr_pos += share->kc_info.field_lengths[field_index];
                }
                // now the variable sized rows
                for (uint32_t i = 0; i < num_var_cols_for_query; i++) {
                    uint field_index = var_cols_for_query[i];
                    Field* field = table->field[field_index];
                    uint32_t field_len = *(uint32_t *)curr_pos;
                    curr_pos += sizeof(field_len);
                    unpack_var_field(
                        buf + field_offset(field, table),
                        curr_pos,
                        field_len,
                        share->kc_info.length_bytes[field_index]
                        );
                    curr_pos += field_len;
                }
                // now the blobs
                if (read_blobs) {
                    uint32_t blob_size = *(uint32_t *)curr_pos;
                    curr_pos += sizeof(blob_size);
                    error = unpack_blobs(
                        buf,
                        curr_pos,
                        blob_size,
                        true
                        );
                    curr_pos += blob_size;
                    if (error) {
                        invalidate_bulk_fetch();
                        goto exit;
                    }
                }
                error = 0;
            }
        }
    }

    curr_range_query_buff_offset = curr_pos - range_query_buff;
exit:
    return error;
}
```

#### Index\_end

所有数据都读完之后，handler框架会调用index\_end关闭cursor，并重置一些状态变量。

```plain
int ha_tokudb::index_end() {
    range_lock_grabbed = false;
    range_lock_grabbed_null = false;
    if (cursor) {
        int r = cursor->c_close(cursor);
        assert_always(r==0);
        cursor = NULL;
        remove_from_trx_handler_list();
        last_cursor_error = 0;
    }
    active_index = tokudb_active_index = MAX_KEY;

    //
    // reset query variables
    //
    unpack_entire_row = true;
    read_blobs = true;
    read_key = true;
    num_fixed_cols_for_query = 0;
    num_var_cols_for_query = 0;

    invalidate_bulk_fetch();
    invalidate_icp();
    doing_bulk_fetch = false;
    close_dsmrr();

    TOKUDB_HANDLER_DBUG_RETURN(0);
}
```

这就是一条query语句在tokudb引擎执行的大致过程。下个月见！

[阿里云RDS-数据库内核组](http://mysql.taobao.org/)  
[欢迎在github上star AliSQL](https://github.com/alibaba/AliSQL)  
阅读： -  
[![知识共享许可协议](assets/1590645940-8232d49bd3e964f917fa8f469ae7c52a.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)  
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

---------------------------------------------------


原网址: [访问](http://mysql.taobao.org/monthly/2017/01/10/)

创建于: 2020-05-28 14:05:40

目录: default

标签: `mysql.taobao.org`

