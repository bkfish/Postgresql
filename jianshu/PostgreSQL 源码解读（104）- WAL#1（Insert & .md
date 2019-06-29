本节介绍了插入数据时与WAL相关的处理逻辑，主要的函数是heap_insert。

### 一、数据结构

**宏定义**  
包括Pointer/Page/XLOG_HEAP_INSERT/XLogRecPtr等

    
    
    typedef char* Pointer;//指针
    typedef Pointer Page;//Page
    
    #define XLOG_HEAP_INSERT   0x00
    
    /*
     * Pointer to a location in the XLOG.  These pointers are 64 bits wide,
     * because we don't want them ever to overflow.
     * 指向XLOG中的位置.
     * 这些指针大小为64bit,以确保指针不会溢出.
     */
    typedef uint64 XLogRecPtr;
    
    /*
     * Additional macros for access to page headers. (Beware multiple evaluation
     * of the arguments!)
     */
    #define PageGetLSN(page) \
        PageXLogRecPtrGet(((PageHeader) (page))->pd_lsn)
    #define PageSetLSN(page, lsn) \
        PageXLogRecPtrSet(((PageHeader) (page))->pd_lsn, lsn)
    
    

**xl_heap_insert**  
插入时需要获知的信息结构

    
    
    /*
     * xl_heap_insert/xl_heap_multi_insert flag values, 8 bits are available.
     */
    /* PD_ALL_VISIBLE was cleared */
    #define XLH_INSERT_ALL_VISIBLE_CLEARED          (1<<0)
    #define XLH_INSERT_LAST_IN_MULTI                (1<<1)
    #define XLH_INSERT_IS_SPECULATIVE               (1<<2)
    #define XLH_INSERT_CONTAINS_NEW_TUPLE           (1<<3)
    
    /* This is what we need to know about insert */
    //这是在插入时需要获知的信息
    typedef struct xl_heap_insert
    {
        //元组在page中的偏移
        OffsetNumber offnum;        /* inserted tuple's offset */
        uint8       flags;          //标记位
    
        /* xl_heap_header & TUPLE DATA in backup block 0 */
        //xl_heap_header & TUPLE DATA在备份块0中
    } xl_heap_insert;
    //xl_heap_insert大小
    #define SizeOfHeapInsert    (offsetof(xl_heap_insert, flags) + sizeof(uint8))
    
    

**xl_heap_header**  
PG不会在WAL中存储插入/更新的元组的全部固定部分(HeapTupleHeaderData)  
xl_heap_header是必须存储的结构.

    
    
    /*
     * We don't store the whole fixed part (HeapTupleHeaderData) of an inserted
     * or updated tuple in WAL; we can save a few bytes by reconstructing the
     * fields that are available elsewhere in the WAL record, or perhaps just
     * plain needn't be reconstructed.  These are the fields we must store.
     * NOTE: t_hoff could be recomputed, but we may as well store it because
     * it will come for free due to alignment considerations.
     * PG不会在WAL中存储插入/更新的元组的全部固定部分(HeapTupleHeaderData);
     *   我们可以通过重新构造在WAL记录中可用的一些字段来节省一些空间,或者直接扁平化处理。
     * 这些都是我们必须存储的字段。
     * 注意:t_hoff可以重新计算，但我们也需要存储它，因为出于对齐的考虑,会被析构。
     */
    typedef struct xl_heap_header
    {
        uint16      t_infomask2;//t_infomask2标记
        uint16      t_infomask;//t_infomask标记
        uint8       t_hoff;//t_hoff
    } xl_heap_header;
    //HeapHeader的大小
    #define SizeOfHeapHeader    (offsetof(xl_heap_header, t_hoff) + sizeof(uint8))
    
    

### 二、源码解读

heap_insert的主要逻辑是插入元组到堆中,其中存在对WAL(XLog)进行处理的部分.

    
    
    /*
     *  heap_insert     - insert tuple into a heap
     *                    插入元组到堆中
     * 
     * The new tuple is stamped with current transaction ID and the specified
     * command ID.
     * 新元组使用当前事务ID和指定的命令ID标记。
     * 
     * If the HEAP_INSERT_SKIP_WAL option is specified, the new tuple is not
     * logged in WAL, even for a non-temp relation.  Safe usage of this behavior
     * requires that we arrange that all new tuples go into new pages not
     * containing any tuples from other transactions, and that the relation gets
     * fsync'd before commit.  (See also heap_sync() comments)
     * 如果指定了HEAP_INSERT_SKIP_WAL选项，那么新的元组就不会记录到WAL中，
     *   即使对于非临时关系也是如此。
     * 如希望安全使用此选项,要求我们组织所有的新元组写入不包含来自其他事务的其他元组的新页面，
     *   并且关系在提交之前得到fsync。(请参阅heap_sync()函数的注释)
     * 
     * The HEAP_INSERT_SKIP_FSM option is passed directly to
     * RelationGetBufferForTuple, which see for more info.
     * HEAP_INSERT_SKIP_FSM选项作为参数直接传给RelationGetBufferForTuple
     *
     * HEAP_INSERT_FROZEN should only be specified for inserts into
     * relfilenodes created during the current subtransaction and when
     * there are no prior snapshots or pre-existing portals open.
     * This causes rows to be frozen, which is an MVCC violation and
     * requires explicit options chosen by user.
     * HEAP_INSERT_FROZEN应该只针对在当前子事务中创建的relfilenodes插入，
     *   以及在没有打开以前的snapshots或已经存在的portals时指定。
     * 这会导致行冻结，这是一种违反MVCC的行为，需要用户选择显式选项。
     * 
     * HEAP_INSERT_SPECULATIVE is used on so-called "speculative insertions",
     * which can be backed out afterwards without aborting the whole transaction.
     * Other sessions can wait for the speculative insertion to be confirmed,
     * turning it into a regular tuple, or aborted, as if it never existed.
     * Speculatively inserted tuples behave as "value locks" of short duration,
     * used to implement INSERT .. ON CONFLICT.
     * HEAP_INSERT_SPECULATIVE用于所谓的“投机性插入 speculative insertions”，
     *   这些插入可以在不中止整个事务的情况下在事后退出。
     * 其他会话可以等待投机性插入得到确认，将其转换为常规元组，
     *   或者中止，就像它不存在一样。
     * Speculatively插入的元组表现为短期的“值锁”，用于实现INSERT .. ON CONFLICT。
     *
     * Note that most of these options will be applied when inserting into the
     * heap's TOAST table, too, if the tuple requires any out-of-line data.  Only
     * HEAP_INSERT_SPECULATIVE is explicitly ignored, as the toast data does not
     * partake in speculative insertion.
     * 注意:在插入到堆的TOAST表时,如需要out-of-line数据,那么也会应用这些选项.
     * 只有HEAP_INSERT_SPECULATIVE选项是显式忽略的,因为toast数据不能speculative insertion.
     *
     * The BulkInsertState object (if any; bistate can be NULL for default
     * behavior) is also just passed through to RelationGetBufferForTuple.
     * BulkInsertState对象(如存在,位状态可以设置为NULL)也传递给RelationGetBufferForTuple函数
     *
     * The return value is the OID assigned to the tuple (either here or by the
     * caller), or InvalidOid if no OID.  The header fields of *tup are updated
     * to match the stored tuple; in particular tup->t_self receives the actual
     * TID where the tuple was stored.  But note that any toasting of fields
     * within the tuple data is NOT reflected into *tup.
     * 返回值是分配给元组的OID(在这里或由调用方指定)，
     *   如果没有OID，则是InvalidOid。
     * 更新*tup的头部字段以匹配存储的元组;特别是tup->t_self接收元组存储的实际TID。
     * 但是请注意，元组数据中的字段的任何toasting都不会反映到*tup中。
     */
    /*
    输入：
        relation-数据表结构体
        tup-Heap Tuple数据（包括头部数据等），亦即数据行
        cid-命令ID（顺序）
        options-选项
        bistate-BulkInsert状态
    输出：
        Oid-数据表Oid
    */
    Oid
    heap_insert(Relation relation, HeapTuple tup, CommandId cid,
                int options, BulkInsertState bistate)
    {
        TransactionId xid = GetCurrentTransactionId();//事务id
        HeapTuple   heaptup;//Heap Tuple数据，亦即数据行
        Buffer      buffer;//数据缓存块
        Buffer      vmbuffer = InvalidBuffer;//vm缓冲块
        bool        all_visible_cleared = false;//标记
    
        /*
         * Fill in tuple header fields, assign an OID, and toast the tuple if
         * necessary.
         * 填充元组的头部字段,分配OID,如需要处理元组的toast信息
         *
         * Note: below this point, heaptup is the data we actually intend to store
         * into the relation; tup is the caller's original untoasted data.
         * 注意:在这一点以下，heaptup是我们实际打算存储到关系中的数据;
         *   tup是调用方的原始untoasted的数据。
         */
        //插入前准备工作，比如设置t_infomask标记等
        heaptup = heap_prepare_insert(relation, tup, xid, cid, options);
    
        /*
         * Find buffer to insert this tuple into.  If the page is all visible,
         * this will also pin the requisite visibility map page.
         * 查找缓冲区并将此元组插入。
         * 如页面都是可见的，这也将固定必需的可见性映射页面。
         */
        //获取相应的buffer，详见上面的子函数解析
        buffer = RelationGetBufferForTuple(relation, heaptup->t_len,
                                           InvalidBuffer, options, bistate,
                                           &vmbuffer, NULL);
    
        /*
         * We're about to do the actual insert -- but check for conflict first, to
         * avoid possibly having to roll back work we've just done.
         * 即将执行实际的插入操作 -- 但首先要检查冲突,以避免可能的回滚.
         *
         * This is safe without a recheck as long as there is no possibility of
         * another process scanning the page between this check and the insert
         * being visible to the scan (i.e., an exclusive buffer content lock is
         * continuously held from this point until the tuple insert is visible).
         * 不重新检查也是安全的,只要在该检查和插入之间不存在其他正在执行扫描页面的进程
         *   (页面对于扫描进程是可见的)
         *
         * For a heap insert, we only need to check for table-level SSI locks. Our
         * new tuple can't possibly conflict with existing tuple locks, and heap
         * page locks are only consolidated versions of tuple locks; they do not
         * lock "gaps" as index page locks do.  So we don't need to specify a
         * buffer when making the call, which makes for a faster check.
         * 对于堆插入,我们只需要检查表级别的SSI锁.
         * 新元组不可能与现有的元组锁冲突，堆页锁只是元组锁的合并版本;
         *   它们不像索引页锁那样锁定“间隙”。
         * 所以我们在调用时不需要指定缓冲区，这样可以更快地进行检查。
         */
        //检查序列化是否冲突
        CheckForSerializableConflictIn(relation, NULL, InvalidBuffer);
    
        /* NO EREPORT(ERROR) from here till changes are logged */
        //开始，变量+1
        START_CRIT_SECTION();
        //插入数据（详见上一节对该函数的解析）
        RelationPutHeapTuple(relation, buffer, heaptup,
                             (options & HEAP_INSERT_SPECULATIVE) != 0);
        //如Page is All Visible
        if (PageIsAllVisible(BufferGetPage(buffer)))
        {
            //复位
            all_visible_cleared = true;
            PageClearAllVisible(BufferGetPage(buffer));
            visibilitymap_clear(relation,
                                ItemPointerGetBlockNumber(&(heaptup->t_self)),
                                vmbuffer, VISIBILITYMAP_VALID_BITS);
        }
    
        /*
         * XXX Should we set PageSetPrunable on this page ?
         * XXX 在页面上设置PageSetPrunable标记?
         *
         * The inserting transaction may eventually abort thus making this tuple
         * DEAD and hence available for pruning. Though we don't want to optimize
         * for aborts, if no other tuple in this page is UPDATEd/DELETEd, the
         * aborted tuple will never be pruned until next vacuum is triggered.
         * 插入事务可能会中止，从而使这个元组"死亡/DEAD"，需要进行裁剪pruning。
         *  虽然我们不想优化事务的中止处理，但是如果本页中没有其他元组被更新/删除，
         *  中止的元组将永远不会被删除，直到下一次触发vacuum。
         *
         * If you do add PageSetPrunable here, add it in heap_xlog_insert too.
         * 如果在这里增加PageSetPrunable,也需要在heap_xlog_insert中添加
         */
        //设置缓冲块为脏块
        MarkBufferDirty(buffer);
    
        /* XLOG stuff */
        //记录日志
        if (!(options & HEAP_INSERT_SKIP_WAL) && RelationNeedsWAL(relation))
        {
            xl_heap_insert xlrec;
            xl_heap_header xlhdr;
            XLogRecPtr  recptr;//uint64
            Page        page = BufferGetPage(buffer);//获取相应的Page
            uint8       info = XLOG_HEAP_INSERT;//XLOG_HEAP_INSERT -> 0x00
            int         bufflags = 0;
    
            /*
             * If this is a catalog, we need to transmit combocids to properly
             * decode, so log that as well.
             * 如果这是一个catalog,需要传输combocids进行解码,因此也需要记录到日志中.
             */
            if (RelationIsAccessibleInLogicalDecoding(relation))
                log_heap_new_cid(relation, heaptup);
    
            /*
             * If this is the single and first tuple on page, we can reinit the
             * page instead of restoring the whole thing.  Set flag, and hide
             * buffer references from XLogInsert.
             * 如果这是页面上独立的第一个元组，我们可以重新初始化页面，而不是恢复整个页面。
             * 设置标志，并隐藏XLogInsert的缓冲区引用。
             */
            if (ItemPointerGetOffsetNumber(&(heaptup->t_self)) == FirstOffsetNumber &&
                PageGetMaxOffsetNumber(page) == FirstOffsetNumber)
            {
                info |= XLOG_HEAP_INIT_PAGE;
                bufflags |= REGBUF_WILL_INIT;
            }
            //Item在页面中的偏移
            xlrec.offnum = ItemPointerGetOffsetNumber(&heaptup->t_self);
            xlrec.flags = 0;//标记
            if (all_visible_cleared)
                xlrec.flags |= XLH_INSERT_ALL_VISIBLE_CLEARED;
            if (options & HEAP_INSERT_SPECULATIVE)
                xlrec.flags |= XLH_INSERT_IS_SPECULATIVE;
            Assert(ItemPointerGetBlockNumber(&heaptup->t_self) == BufferGetBlockNumber(buffer));
    
    
            /*
             * For logical decoding, we need the tuple even if we're doing a full
             * page write, so make sure it's included even if we take a full-page
             * image. (XXX We could alternatively store a pointer into the FPW).
             * 对于逻辑解码,即使正在进行全page write,也需要元组数据,
             *   以确保该元组在全页面镜像时包含在内.
             * (XXX 我们也可以将指针存储到FPW中)
             */
            if (RelationIsLogicallyLogged(relation))
            {
                xlrec.flags |= XLH_INSERT_CONTAINS_NEW_TUPLE;
                bufflags |= REGBUF_KEEP_DATA;
            }
    
            XLogBeginInsert();//开始WAL插入
            XLogRegisterData((char *) &xlrec, SizeOfHeapInsert);//注册数据
            //设置标记
            xlhdr.t_infomask2 = heaptup->t_data->t_infomask2;
            xlhdr.t_infomask = heaptup->t_data->t_infomask;
            xlhdr.t_hoff = heaptup->t_data->t_hoff;
    
            /*
             * note we mark xlhdr as belonging to buffer; if XLogInsert decides to
             * write the whole page to the xlog, we don't need to store
             * xl_heap_header in the xlog.
             * 注意:我们标记xlhdr属于缓冲区;
             *   如果XLogInsert确定把整个page写入到xlog中,那么不需要在xlog中存储xl_heap_header
             */
            XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags);//标记
            XLogRegisterBufData(0, (char *) &xlhdr, SizeOfHeapHeader);//tuple头部
            /* PG73FORMAT: write bitmap [+ padding] [+ oid] + data */
            XLogRegisterBufData(0,
                                (char *) heaptup->t_data + SizeofHeapTupleHeader,
                                heaptup->t_len - SizeofHeapTupleHeader);//tuple实际数据
    
            /* filtering by origin on a row level is much more efficient */
            //根据行级别上的原点进行过滤要有效得多
            XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN);
            //插入数据
            recptr = XLogInsert(RM_HEAP_ID, info);
            //设置LSN
            PageSetLSN(page, recptr);
        }
        //完成！
        END_CRIT_SECTION();
        //解锁Buffer，包括vm buffer
        UnlockReleaseBuffer(buffer);
        if (vmbuffer != InvalidBuffer)
            ReleaseBuffer(vmbuffer);
    
        /*
         * If tuple is cachable, mark it for invalidation from the caches in case
         * we abort.  Note it is OK to do this after releasing the buffer, because
         * the heaptup data structure is all in local memory, not in the shared
         * buffer.
         * 如果tuple已缓存，在终止事务时,则将它标记为无效缓存。
         * 注意，在释放缓冲区之后这样做是可以的，
         *   因为heaptup数据结构都在本地内存中，而不是在共享缓冲区中。
         */
        //缓存操作后变“无效”的Tuple
        CacheInvalidateHeapTuple(relation, heaptup, NULL);
    
        /* Note: speculative insertions are counted too, even if aborted later */
        //注意:speculative insertions也会统计(即使终止此事务)
        //更新统计信息
        pgstat_count_heap_insert(relation, 1);
    
        /*
         * If heaptup is a private copy, release it.  Don't forget to copy t_self
         * back to the caller's image, too.
         * 如果heaptup是一个私有的拷贝,释放之.
         *   不要忘了把t_self拷贝回调用者的镜像中.
         */
        if (heaptup != tup)
        {
            tup->t_self = heaptup->t_self;
            heap_freetuple(heaptup);
        }
    
        return HeapTupleGetOid(tup);
    }
    
    

### 三、跟踪分析

测试脚本如下

    
    
    -- Hash Partition
    drop table if exists t_wal_partition;
    create table t_wal_partition (c1 int not null,c2  varchar(40),c3 varchar(40)) partition by hash(c1);
    create table t_wal_partition_1 partition of t_wal_partition for values with (modulus 6,remainder 0);
    create table t_wal_partition_2 partition of t_wal_partition for values with (modulus 6,remainder 1);
    create table t_wal_partition_3 partition of t_wal_partition for values with (modulus 6,remainder 2);
    create table t_wal_partition_4 partition of t_wal_partition for values with (modulus 6,remainder 3);
    create table t_wal_partition_5 partition of t_wal_partition for values with (modulus 6,remainder 4);
    create table t_wal_partition_6 partition of t_wal_partition for values with (modulus 6,remainder 5);
    
    -- 插入路由
    -- delete from t_wal_partition where c1 = 0;
    insert into t_wal_partition(c1,c2,c3) VALUES(0,'HASH0','HAHS0');
    
    

启动gdb,设置断点,进入heap_insert

    
    
    (gdb) b heap_insert
    Breakpoint 1 at 0x4df4d1: file heapam.c, line 2449.
    (gdb) c
    Continuing.
    
    Breakpoint 1, heap_insert (relation=0x7f9c470a8bd8, tup=0x2908850, cid=0, options=0, bistate=0x0) at heapam.c:2449
    2449        TransactionId xid = GetCurrentTransactionId();
    

构造Heap Tuple数据/获取相应的Buffer(102号)

    
    
    2449        TransactionId xid = GetCurrentTransactionId();
    (gdb) n
    2452        Buffer      vmbuffer = InvalidBuffer;
    (gdb) 
    2453        bool        all_visible_cleared = false;
    (gdb) 
    2462        heaptup = heap_prepare_insert(relation, tup, xid, cid, options);
    (gdb) 
    2468        buffer = RelationGetBufferForTuple(relation, heaptup->t_len,
    (gdb) 
    2487        CheckForSerializableConflictIn(relation, NULL, InvalidBuffer);
    (gdb) p *heaptup
    $1 = {t_len = 172, t_self = {ip_blkid = {bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_tableOid = 1247, 
      t_data = 0x2908868}
    (gdb) p buffer
    $2 = 102
    

插入到Buffer中,标记Buffer为Dirty

    
    
    (gdb) n
    2490        START_CRIT_SECTION();
    (gdb) 
    2493                             (options & HEAP_INSERT_SPECULATIVE) != 0);
    (gdb) 
    2492        RelationPutHeapTuple(relation, buffer, heaptup,
    (gdb) 
    2495        if (PageIsAllVisible(BufferGetPage(buffer)))
    (gdb) 
    2515        MarkBufferDirty(buffer);
    (gdb) 
    2518        if (!(options & HEAP_INSERT_SKIP_WAL) && RelationNeedsWAL(relation))
    (gdb) 
    

进入WAL处理部分,获取page(位于1号Page)等信息

    
    
    (gdb) n
    2523            Page        page = BufferGetPage(buffer);
    (gdb) 
    2524            uint8       info = XLOG_HEAP_INSERT;
    (gdb) 
    2525            int         bufflags = 0;
    (gdb) 
    2531            if (RelationIsAccessibleInLogicalDecoding(relation))
    (gdb) p page
    $3 = (Page) 0x7f9c1a505380 "\001"
    (gdb) p *page
    $4 = 1 '\001'
    (gdb) p info
    $5 = 0 '\000'
    

设置xl_heap_insert结构体  
其中偏移=34,标记位为0x0

    
    
    (gdb) n
    2539            if (ItemPointerGetOffsetNumber(&(heaptup->t_self)) == FirstOffsetNumber &&
    (gdb) 
    2546            xlrec.offnum = ItemPointerGetOffsetNumber(&heaptup->t_self);
    (gdb) 
    2547            xlrec.flags = 0;
    (gdb) 
    2548            if (all_visible_cleared)
    (gdb) 
    2550            if (options & HEAP_INSERT_SPECULATIVE)
    (gdb) 
    2552            Assert(ItemPointerGetBlockNumber(&heaptup->t_self) == BufferGetBlockNumber(buffer));
    (gdb) 
    2559            if (RelationIsLogicallyLogged(relation) &&
    (gdb) 
    (gdb) p xlrec
    $6 = {offnum = 34, flags = 0 '\000'}
    

开始插入WAL,并注册相关数据

    
    
    (gdb) 
    2566            XLogBeginInsert();
    (gdb) n
    2567            XLogRegisterData((char *) &xlrec, SizeOfHeapInsert);
    (gdb) 
    2569            xlhdr.t_infomask2 = heaptup->t_data->t_infomask2;
    (gdb) n
    2570            xlhdr.t_infomask = heaptup->t_data->t_infomask;
    (gdb) n
    2571            xlhdr.t_hoff = heaptup->t_data->t_hoff;
    (gdb) 
    2578            XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags);
    (gdb) 
    2579            XLogRegisterBufData(0, (char *) &xlhdr, SizeOfHeapHeader);
    (gdb) 
    2583                                heaptup->t_len - SizeofHeapTupleHeader);
    (gdb) 
    2581            XLogRegisterBufData(0,
    (gdb) 
    2582                                (char *) heaptup->t_data + SizeofHeapTupleHeader,
    (gdb) 
    2581            XLogRegisterBufData(0,
    (gdb) 
    2586            XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN);
    (gdb) 
    2588            recptr = XLogInsert(RM_HEAP_ID, info);
    (gdb) 
    (gdb) p xlhdr
    $8 = {t_infomask2 = 30, t_infomask = 2057, t_hoff = 32 ' '}
    
    

调用XLogInsert,插入WAL,并设置LSN

    
    
    (gdb) n
    2590            PageSetLSN(page, recptr);
    (gdb) p recptr
    $9 = 5411089336
    (gdb) p info
    $10 = 0 '\000'
    (gdb) 
    

执行其他后续操作,完成函数调用

    
    
    (gdb) n
    2593        END_CRIT_SECTION();
    (gdb) 
    2595        UnlockReleaseBuffer(buffer);
    (gdb) 
    2596        if (vmbuffer != InvalidBuffer)
    (gdb) 
    2605        CacheInvalidateHeapTuple(relation, heaptup, NULL);
    (gdb) 
    2608        pgstat_count_heap_insert(relation, 1);
    (gdb) 
    2614        if (heaptup != tup)
    (gdb) 
    2620        return HeapTupleGetOid(tup);
    (gdb) 
    2621    }
    (gdb) 
    

函数
**XLogBeginInsert/XLogRegisterData/XLogRegisterBuffer/XLogRegisterBufData/XLogInsert**
下一节再行介绍.

### 四、参考资料

[Write Ahead Logging — WAL](http://www.interdb.jp/pg/pgsql09.html)  
[PostgreSQL 源码解读（4）-插入数据#3（heap_insert）](https://www.jianshu.com/p/0e16fce4ca73)  
[PgSQL · 特性分析 · 数据库崩溃恢复（上）](http://mysql.taobao.org/monthly/2017/05/03/)  
[PgSQL · 特性分析 · 数据库崩溃恢复（下）](http://mysql.taobao.org/monthly/2017/06/04/)  
[PgSQL · 特性分析 · Write-Ahead
Logging机制浅析](http://mysql.taobao.org/monthly/2017/03/02/)  
[PostgreSQL WAL Buffers, Clog Buffers Deep
Dive](https://www.slideshare.net/pgday_seoul/pgdayseoul-2017-3-postgresql-wal-buffers-clog-buffers-deep-dive?from_action=save)

