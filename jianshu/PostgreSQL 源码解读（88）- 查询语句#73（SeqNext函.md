本节介绍了SeqNext函数的主要实现逻辑以及该函数中初始化相关数据结构的实现逻辑。SeqNext函数作为参数传递到函数ExecScan中，执行实际的扫描操作。

### 一、数据结构

**TupleTableSlot**  
Tuple Table Slot,用于存储元组相关信息

    
    
    /* base tuple table slot type */
    typedef struct TupleTableSlot
    {
        NodeTag     type;//Node标记
    #define FIELDNO_TUPLETABLESLOT_FLAGS 1
        uint16      tts_flags;      /* 布尔状态;Boolean states */
    #define FIELDNO_TUPLETABLESLOT_NVALID 2
        AttrNumber  tts_nvalid;     /* 在tts_values中有多少有效的values;# of valid values in tts_values */
        const TupleTableSlotOps *const tts_ops; /* 实现一个slot的成本;implementation of slot */
    #define FIELDNO_TUPLETABLESLOT_TUPLEDESCRIPTOR 4
        TupleDesc   tts_tupleDescriptor;    /* slot的元组描述符;slot's tuple descriptor */
    #define FIELDNO_TUPLETABLESLOT_VALUES 5
        Datum      *tts_values;     /* 当前属性值;current per-attribute values */
    #define FIELDNO_TUPLETABLESLOT_ISNULL 6
        bool       *tts_isnull;     /* 当前属性isnull标记;current per-attribute isnull flags */
        MemoryContext tts_mcxt;     /*内存上下文; slot itself is in this context */
    } TupleTableSlot;
    
    
    typedef struct tupleDesc
    {
        int         natts;          /* tuple中的属性数量;number of attributes in the tuple */
        Oid         tdtypeid;       /* tuple类型的组合类型ID;composite type ID for tuple type */
        int32       tdtypmod;       /* tuple类型的typmode;typmod for tuple type */
        int         tdrefcount;     /* 依赖计数,如为-1,则没有依赖;reference count, or -1 if not counting */
        TupleConstr *constr;        /* 约束,如无则为NULL;constraints, or NULL if none */
        /* attrs[N] is the description of Attribute Number N+1 */
        //attrs[N]是第N+1个属性的描述符
        FormData_pg_attribute attrs[FLEXIBLE_ARRAY_MEMBER];
    }  *TupleDesc;
    
    

**HeapTuple**  
HeapTupleData是一个指向元组的内存数据结构  
HeapTuple是指向HeapTupleData指针

    
    
    /*
     * HeapTupleData is an in-memory data structure that points to a tuple.
     * HeapTupleData是一个指向元组的内存数据结构。
     *
     * There are several ways in which this data structure is used:
     * 使用这种数据结构有几种方式:
     *
     * * Pointer to a tuple in a disk buffer: t_data points directly into the
     *   buffer (which the code had better be holding a pin on, but this is not
     *   reflected in HeapTupleData itself).
     *   指向磁盘缓冲区中的一个tuple的指针:
     *      t_data点直接指向缓冲区(代码最好将pin放在缓冲区中，但这在HeapTupleData本身中没有反映出来)。
     *  
     * * Pointer to nothing: t_data is NULL.  This is used as a failure indication
     *   in some functions.
     *   没有指针:
     *      t_data是空的。用于在一些函数中作为故障指示。
     *
     * * Part of a palloc'd tuple: the HeapTupleData itself and the tuple
     *   form a single palloc'd chunk.  t_data points to the memory location
     *   immediately following the HeapTupleData struct (at offset HEAPTUPLESIZE).
     *   This is the output format of heap_form_tuple and related routines.
     *   palloc'd tuple的一部分:HeapTupleData本身和tuple形成一个单一的palloc'd chunk。
     *      t_data指向HeapTupleData结构体后面的内存位置(偏移HEAPTUPLESIZE)。
     *      这是heap_form_tuple和相关例程的输出格式。
     *
     * * Separately allocated tuple: t_data points to a palloc'd chunk that
     *   is not adjacent to the HeapTupleData.  (This case is deprecated since
     *   it's difficult to tell apart from case #1.  It should be used only in
     *   limited contexts where the code knows that case #1 will never apply.)
     *   单独分配的tuple: 
     *      t_data指向一个与HeapTupleData不相邻的palloc数据块。
     *      (这个情况已废弃不用，因为很难与第一种情况中进行区分。
     *      它应该只在代码知道第一种情况永远不会适用的有限上下文中使用。
     *
     * * Separately allocated minimal tuple: t_data points MINIMAL_TUPLE_OFFSET
     *   bytes before the start of a MinimalTuple.  As with the previous case,
     *   this can't be told apart from case #1 by inspection; code setting up
     *   or destroying this representation has to know what it's doing.
     *   独立分配的最小元组:
     *      t_data指向MinimalTuple开始前偏移MINIMAL_TUPLE_OFFSET个字节的位置。
     *      与前一种情况一样，不能通过检查与第一种情况相区别;
     *      设置或销毁这种表示的代码必须知道它在做什么。
     *
     * t_len should always be valid, except in the pointer-to-nothing case.
     * t_self and t_tableOid should be valid if the HeapTupleData points to
     * a disk buffer, or if it represents a copy of a tuple on disk.  They
     * should be explicitly set invalid in manufactured tuples.
     * t_len应该总是有效的，除非在指针为NULL。
     * 如果HeapTupleData指向磁盘缓冲区，或者它表示磁盘上元组的副本，那么t_self和t_tableOid应该是有效的。
     * 它们应该显式地在制造的元组中设置为无效。
     */
    typedef struct HeapTupleData
    {
        uint32      t_len;          /* *t_data指针的长度;length of *t_data */
        ItemPointerData t_self;     /* SelfItemPointer */
        Oid         t_tableOid;     /* 该元组所属的table;table the tuple came from */
    #define FIELDNO_HEAPTUPLEDATA_DATA 3
        HeapTupleHeader t_data;     /* 指向元组的header&数据;-> tuple header and data */
    } HeapTupleData;
    
    typedef HeapTupleData *HeapTuple;
    
    #define HEAPTUPLESIZE   MAXALIGN(sizeof(HeapTupleData))
    
    
    

**HeapScanDesc**  
HeapScanDesc是指向HeapScanDescData结构体的指针

    
    
    typedef struct HeapScanDescData
    {
        /* scan parameters */
        Relation    rs_rd;          /* 堆表描述符;heap relation descriptor */
        Snapshot    rs_snapshot;    /* 快照;snapshot to see */
        int         rs_nkeys;       /* 扫描键数;number of scan keys */
        ScanKey     rs_key;         /* 扫描键数组;array of scan key descriptors */
        bool        rs_bitmapscan;  /* bitmap scan=>T;true if this is really a bitmap scan */
        bool        rs_samplescan;  /* sample scan=>T;true if this is really a sample scan */
        bool        rs_pageatatime; /* 是否验证可见性(MVCC机制);verify visibility page-at-a-time? */
        bool        rs_allow_strat; /* 是否允许访问策略的使用;allow or disallow use of access strategy */
        bool        rs_allow_sync;  /* 是否允许syncscan的使用;allow or disallow use of syncscan */
        bool        rs_temp_snap;   /* 是否在扫描结束后取消快照"登记";unregister snapshot at scan end? */
    
        /* state set up at initscan time */
        //在initscan时配置的状态
        BlockNumber rs_nblocks;     /* rel中的blocks总数;total number of blocks in rel */
        BlockNumber rs_startblock;  /* 开始的block编号;block # to start at */
        BlockNumber rs_numblocks;   /* 最大的block编号;max number of blocks to scan */
        /* rs_numblocks is usually InvalidBlockNumber, meaning "scan whole rel" */
        //rs_numblocks通常值为InvalidBlockNumber,意味着扫描整个rel
        
        BufferAccessStrategy rs_strategy;   /* 读取时的访问场景;access strategy for reads */
        bool        rs_syncscan;    /* 在syncscan逻辑处理时是否报告位置;report location to syncscan logic? */
    
        /* scan current state */
        //扫描时的当前状态
        bool        rs_inited;      /* 如为F,则扫描尚未初始化;false = scan not init'd yet */
        HeapTupleData rs_ctup;      /* 当前扫描的tuple;current tuple in scan, if any */
        BlockNumber rs_cblock;      /* 当前扫描的block;current block # in scan, if any */
        Buffer      rs_cbuf;        /* 当前扫描的buffer;current buffer in scan, if any */
        /* NB: if rs_cbuf is not InvalidBuffer, we hold a pin on that buffer */
        //注意:如果rs_cbuf<>InvalidBuffer,在buffer设置pin
    
        ParallelHeapScanDesc rs_parallel;   /* 并行扫描信息;parallel scan information */
    
        /* these fields only used in page-at-a-time mode and for bitmap scans */
        //下面的变量只用于page-at-a-time模式以及位图扫描
        int         rs_cindex;      /* 在vistuples中的当前元组索引;current tuple's index in vistuples */
        int         rs_ntuples;     /* page中的可见元组计数;number of visible tuples on page */
        OffsetNumber rs_vistuples[MaxHeapTuplesPerPage];    /* 元组的偏移;their offsets */
    } HeapScanDescData;
    
    /* struct definitions appear in relscan.h */
    typedef struct HeapScanDescData *HeapScanDesc;
    
    

**ScanState**  
ScanState扩展了对表示底层关系扫描的节点类型的PlanState。

    
    
    /* ----------------     *   ScanState information
     *
     *      ScanState extends PlanState for node types that represent
     *      scans of an underlying relation.  It can also be used for nodes
     *      that scan the output of an underlying plan node --- in that case,
     *      only ScanTupleSlot is actually useful, and it refers to the tuple
     *      retrieved from the subplan.
     *      ScanState扩展了对表示底层关系扫描的节点类型的PlanState。
     *      它还可以用于扫描底层计划节点的输出的节点——在这种情况下，实际上只有ScanTupleSlot有用，它引用从子计划检索到的元组。
     *
     *      currentRelation    relation being scanned (NULL if none)
     *                          正在扫描的relation,如无则为NULL
     *      currentScanDesc    current scan descriptor for scan (NULL if none)
     *                         当前的扫描描述符,如无则为NULL
     *      ScanTupleSlot      pointer to slot in tuple table holding scan tuple
     *                         指向tuple table中的slot
     * ----------------     */
    typedef struct ScanState
    {
        PlanState   ps;             /* its first field is NodeTag */
        Relation    ss_currentRelation;
        HeapScanDesc ss_currentScanDesc;
        TupleTableSlot *ss_ScanTupleSlot;
    } ScanState;
    
    /* ----------------     *   SeqScanState information
     * ----------------     */
    typedef struct SeqScanState
    {
        ScanState   ss;             /* its first field is NodeTag */
        Size        pscan_len;      /* size of parallel heap scan descriptor */
    } SeqScanState;
    
    

### 二、源码解读

SeqNext函数是ExecSeqScan的元组的实际访问方法(ExecScanAccessMtd).这里简单介绍了初始化过程,实际的元组获取过程下节再行介绍.

    
    
    /* ----------------------------------------------------------------     *      SeqNext
     *
     *      This is a workhorse for ExecSeqScan
     *      这是ExecSeqScan的实际访问方法(ExecScanAccessMtd)
     * ----------------------------------------------------------------     */
    static TupleTableSlot *
    SeqNext(SeqScanState *node)
    {
        HeapTuple   tuple;
        HeapScanDesc scandesc;
        EState     *estate;
        ScanDirection direction;
        TupleTableSlot *slot;
    
        /*
         * get information from the estate and scan state
         * 从EState和ScanSate中获取相关信息
         */
        scandesc = node->ss.ss_currentScanDesc;
        estate = node->ss.ps.state;
        direction = estate->es_direction;
        slot = node->ss.ss_ScanTupleSlot;
    
        if (scandesc == NULL)//如scandesc为NULL,则初始化
        {
            /*
             * We reach here if the scan is not parallel, or if we're serially
             * executing a scan that was planned to be parallel.
             * 如果扫描不是并行的，或者正在序列化执行计划为并行的扫描，实现逻辑就会到这里。
             */
            scandesc = heap_beginscan(node->ss.ss_currentRelation,
                                      estate->es_snapshot,
                                      0, NULL);//扫描前准备,返回HeapScanDesc
            node->ss.ss_currentScanDesc = scandesc;//赋值
        }
    
        /*
         * get the next tuple from the table
         * 从数据表中获取下一个tuple
         */
        tuple = heap_getnext(scandesc, direction);
    
        /*
         * save the tuple and the buffer returned to us by the access methods in
         * our scan tuple slot and return the slot.  Note: we pass 'false' because
         * tuples returned by heap_getnext() are pointers onto disk pages and were
         * not created with palloc() and so should not be pfree()'d.  Note also
         * that ExecStoreHeapTuple will increment the refcount of the buffer; the
         * refcount will not be dropped until the tuple table slot is cleared.
         * 保存的元组和缓冲区，这些信息通过调用访问方法时返回,同时该方法返回slot。
         * 注意:我们传递‘false’，因为heap_getnext()返回的元组是指向磁盘页面的指针，
         * 不是用palloc()创建的，所以不应该使用pfree()函数释放。
         * 还要注意，ExecStoreHeapTuple将增加缓冲区的refcount;在清除tuple table slot之前不会删除refcount。
         */
        if (tuple)//获取了tuple
            ExecStoreBufferHeapTuple(tuple, /* 需要存储的tuple;tuple to store */
                                     slot,  /* 即将用于存储tuple的slot;slot to store in */
                                     scandesc->rs_cbuf);    /* 与该tuple相关联的缓冲区;
                                                               buffer associated
                                                             * with this tuple */
        else
            ExecClearTuple(slot);//tuple为NULL,则释放slot
    
        return slot;//返回slot
    }
    
    /*
     * SeqRecheck -- access method routine to recheck a tuple in EvalPlanQual
     * 访问方法在EvalPlanQual中对元组重新检查
     */
    static bool
    SeqRecheck(SeqScanState *node, TupleTableSlot *slot)
    {
        /*
         * Note that unlike IndexScan, SeqScan never use keys in heap_beginscan
         * (and this is very bad) - so, here we do not check are keys ok or not.
         * 注意，与IndexScan不同，SeqScan从不使用heap_beginscan中的键(这很糟糕)——因此，这里我们不检查键是否正确。
         */
        //直接返回T
        return true;
    }
    
    
    /* ----------------     *      heap_beginscan  - begin relation scan
     *      heap_beginscan - 开始堆表扫描
     *
     * heap_beginscan is the "standard" case.
     * heap_beginscan是标准情况
     *
     * heap_beginscan_catalog differs in setting up its own temporary snapshot.
     * heap_beginscan_catalog与heap_beginscan不同的是,该方法配置自己的临时快照
     *
     * heap_beginscan_strat offers an extended API that lets the caller control
     * whether a nondefault buffer access strategy can be used, and whether
     * syncscan can be chosen (possibly resulting in the scan not starting from
     * block zero).  Both of these default to true with plain heap_beginscan.
     * heap_beginscan_strat提供了一个扩展API，可以让调用者控制是否可以使用非默认的缓冲区访问策略，
     * 以及是否可以选择syncscan(可能导致扫描从非0块开始)。
     * 对于普通的heap_beginscan，这两个默认值都为T。
     *
     * heap_beginscan_bm is an alternative entry point for setting up a
     * HeapScanDesc for a bitmap heap scan.  Although that scan technology is
     * really quite unlike a standard seqscan, there is just enough commonality
     * to make it worth using the same data structure.
     * heap_beginscan_bm是为位图堆扫描设置HeapScanDesc的备选入口点。
     * 尽管这种扫描技术与标准的seqscan非常不同，但它有足够的共性，因此值得使用相同的数据结构。
     * 
     * heap_beginscan_sampling is an alternative entry point for setting up a
     * HeapScanDesc for a TABLESAMPLE scan.  As with bitmap scans, it's worth
     * using the same data structure although the behavior is rather different.
     * In addition to the options offered by heap_beginscan_strat, this call
     * also allows control of whether page-mode visibility checking is used.
     * heap_beginscan_sampling是为TABLESAMPLE扫描设置HeapScanDesc的备选入口点。
     * 与位图扫描一样，使用相同的数据结构是值得的，尽管其行为相当不同。
     * 除了heap_beginscan_strat提供的选项之外，这个调用还允许控制是否使用页面模式可见性检查。
     * ----------------     */
    HeapScanDesc
    heap_beginscan(Relation relation, Snapshot snapshot,
                   int nkeys, ScanKey key)
    {
        return heap_beginscan_internal(relation, snapshot, nkeys, key, NULL,
                                       true, true, true, false, false, false);//标准情况,调用heap_beginscan_internal
    }
    
    
    static HeapScanDesc
    heap_beginscan_internal(Relation relation, Snapshot snapshot,//Relation & snapshot
                            int nkeys, ScanKey key,//键个数&扫描键
                            ParallelHeapScanDesc parallel_scan,//并行扫描描述符
                            bool allow_strat,//允许开始?
                            bool allow_sync,//允许sync扫描?
                            bool allow_pagemode,//允许页模式?
                            bool is_bitmapscan,//是否位图扫描
                            bool is_samplescan,//是否采样扫描
                            bool temp_snap)//是否使用临时快照
    {
        HeapScanDesc scan;//堆表扫描描述符
    
        /*
         * increment relation ref count while scanning relation
         * 在扫描时增加relation依赖计数
         *
         * This is just to make really sure the relcache entry won't go away while
         * the scan has a pointer to it.  Caller should be holding the rel open
         * anyway, so this is redundant in all normal scenarios...
         * 这只是为了确保relcache条目不会在扫描存在指向它的指针时消失。
         * 无论如何，调用者都应该保持rel是打开的，所以这在所有正常情况下都是多余的……
         */
        RelationIncrementReferenceCount(relation);
    
        /*
         * allocate and initialize scan descriptor
         * 分配并初始化扫描描述符
         */
        scan = (HeapScanDesc) palloc(sizeof(HeapScanDescData));
    
        scan->rs_rd = relation;
        scan->rs_snapshot = snapshot;
        scan->rs_nkeys = nkeys;
        scan->rs_bitmapscan = is_bitmapscan;
        scan->rs_samplescan = is_samplescan;
        scan->rs_strategy = NULL;   /* set in initscan */
        scan->rs_allow_strat = allow_strat;
        scan->rs_allow_sync = allow_sync;
        scan->rs_temp_snap = temp_snap;
        scan->rs_parallel = parallel_scan;
    
        /*
         * we can use page-at-a-time mode if it's an MVCC-safe snapshot
         * 如果快照是MVCC-safte,那么要使用page-at-a-time模式
         */
        scan->rs_pageatatime = allow_pagemode && IsMVCCSnapshot(snapshot);
    
        /*
         * For a seqscan in a serializable transaction, acquire a predicate lock
         * on the entire relation. This is required not only to lock all the
         * matching tuples, but also to conflict with new insertions into the
         * table. In an indexscan, we take page locks on the index pages covering
         * the range specified in the scan qual, but in a heap scan there is
         * nothing more fine-grained to lock. A bitmap scan is a different story,
         * there we have already scanned the index and locked the index pages
         * covering the predicate. But in that case we still have to lock any
         * matching heap tuples.
         * 对于serializable事务中的seqscan，获取整个关系上的谓词锁。
         * 这不仅需要锁定所有匹配的元组，还需要与表中发生的新插入存在冲突。
         * 在indexscan中，在覆盖了scan qual中指定的范围的索引页上获取分页锁，但是在堆扫描中没有更细粒度的锁。
         * 位图扫描则不同，已经扫描了索引并锁定了覆盖谓词的索引页。但在这种情况下，仍然需要锁定所有匹配的堆元组。
         */
        if (!is_bitmapscan)
            PredicateLockRelation(relation, snapshot);
    
        /* we only need to set this up once */
        //设置relid
        scan->rs_ctup.t_tableOid = RelationGetRelid(relation);
    
        /*
         * we do this here instead of in initscan() because heap_rescan also calls
         * initscan() and we don't want to allocate memory again
         * 在这里完成而不是在initscan()中处理是因为heap_rescan也调用initscan()，因此不希望再分配内存
         */
        if (nkeys > 0)
            scan->rs_key = (ScanKey) palloc(sizeof(ScanKeyData) * nkeys);
        else
            scan->rs_key = NULL;
        //初始化scan
        initscan(scan, key, false);
    
        return scan;
    }
    
    /* Get the LockTupleMode for a given MultiXactStatus */
    #define TUPLOCK_from_mxstatus(status) \
                (MultiXactStatusLock[(status)])
    
    /* ----------------------------------------------------------------     *                       heap support routines
     * ----------------------------------------------------------------     */
    
    /* ----------------     *      initscan - scan code common to heap_beginscan and heap_rescan
     *      initscan - heap_beginscan & heap_rescan的扫描代码
     * ----------------     */
    static void
    initscan(HeapScanDesc scan, ScanKey key, bool keep_startblock)
    {
        bool        allow_strat;
        bool        allow_sync;
    
        /*
         * Determine the number of blocks we have to scan.
         * 确定必须扫描的block数
         *
         * It is sufficient to do this once at scan start, since any tuples added
         * while the scan is in progress will be invisible to my snapshot anyway.
         * (That is not true when using a non-MVCC snapshot.  However, we couldn't
         * guarantee to return tuples added after scan start anyway, since they
         * might go into pages we already scanned.  To guarantee consistent
         * results for a non-MVCC snapshot, the caller must hold some higher-level
         * lock that ensures the interesting tuple(s) won't change.)
         * 只要在扫描开始时做一次就足够了，因为在扫描进行过程中添加的任何元组对快照都是不可见的。
         * (在使用非MVCC快照时不是这样,不能保证返回扫描开始后添加的元组，因为它们可能会存储在已扫描的页面。
         *  为了保证非MVCC快照的一致结果，调用者必须持有一些高级锁，以确保有受影响的元组不会改变。)
         */
        if (scan->rs_parallel != NULL)
            scan->rs_nblocks = scan->rs_parallel->phs_nblocks;
        else
            scan->rs_nblocks = RelationGetNumberOfBlocks(scan->rs_rd);
    
        /*
         * If the table is large relative to NBuffers, use a bulk-read access
         * strategy and enable synchronized scanning (see syncscan.c).  Although
         * the thresholds for these features could be different, we make them the
         * same so that there are only two behaviors to tune rather than four.
         * (However, some callers need to be able to disable one or both of these
         * behaviors, independently of the size of the table; also there is a GUC
         * variable that can disable synchronized scanning.)
         * 如果表相对于nbuffer较大，则使用批量读取访问策略并启用同步扫描(参见syncscan.c)。
         * 尽管这些特性的阈值可能不同，但我们使它们相同，以便只有两种行为可以进行调优，而不是四种。
         * (然而，一些调用者需要能够禁用其中一种或两种行为，这与表的大小无关;还有一个GUC变量可以禁用同步扫描。)
         *
         * Note that heap_parallelscan_initialize has a very similar test; if you
         * change this, consider changing that one, too.
         * 注意，heap_parallelscan_initialize中有一个非常类似的测试;
         * 如果你改变了这个，也应该考虑改变那个。
         */
        if (!RelationUsesLocalBuffers(scan->rs_rd) &&
            scan->rs_nblocks > NBuffers / 4)
        {
            allow_strat = scan->rs_allow_strat;
            allow_sync = scan->rs_allow_sync;
        }
        else
            allow_strat = allow_sync = false;//设置为F
    
        if (allow_strat)//允许使用访问策略
        {
            /* During a rescan, keep the previous strategy object. */
            //在重新扫描期间,存储先前的策略(strategy)对象
            if (scan->rs_strategy == NULL)
                scan->rs_strategy = GetAccessStrategy(BAS_BULKREAD);
        }
        else
        {
            if (scan->rs_strategy != NULL)
                FreeAccessStrategy(scan->rs_strategy);
            scan->rs_strategy = NULL;//不允许,则设置为NULL
        }
    
        if (scan->rs_parallel != NULL)//使用并行
        {
            /* For parallel scan, believe whatever ParallelHeapScanDesc says. */
            //对于并行扫描,使用ParallelHeapScanDesc中的变量
            scan->rs_syncscan = scan->rs_parallel->phs_syncscan;
        }
        else if (keep_startblock)
        {
            /*
             * When rescanning, we want to keep the previous startblock setting,
             * so that rewinding a cursor doesn't generate surprising results.
             * Reset the active syncscan setting, though.
             * 当重新扫描时，希望保持先前的startblock设置，以便重新回退游标,这样不会产生令人惊讶的结果。
             * 不过，注意重置活动syncscan的设置。
             */
            scan->rs_syncscan = (allow_sync && synchronize_seqscans);
        }
        else if (allow_sync && synchronize_seqscans)
        {
            scan->rs_syncscan = true;
            scan->rs_startblock = ss_get_location(scan->rs_rd, scan->rs_nblocks);
        }
        else
        {
            scan->rs_syncscan = false;
            scan->rs_startblock = 0;
        }
    
        scan->rs_numblocks = InvalidBlockNumber;
        scan->rs_inited = false;
        scan->rs_ctup.t_data = NULL;
        ItemPointerSetInvalid(&scan->rs_ctup.t_self);
        scan->rs_cbuf = InvalidBuffer;
        scan->rs_cblock = InvalidBlockNumber;
    
        /* page-at-a-time fields are always invalid when not rs_inited */
        //page-at-a-time相关的域通常设置为无效值
    
        /*
         * copy the scan key, if appropriate
         * 如需要,拷贝扫描键
         */
        if (key != NULL)
            memcpy(scan->rs_key, key, scan->rs_nkeys * sizeof(ScanKeyData));
    
        /*
         * Currently, we don't have a stats counter for bitmap heap scans (but the
         * underlying bitmap index scans will be counted) or sample scans (we only
         * update stats for tuple fetches there)
         * 目前，没有一个用于位图堆扫描的统计计数器(但是将计算底层的位图索引扫描)
         * 或样本扫描(只对那里的元组读取更新统计数据)
         */
        if (!scan->rs_bitmapscan && !scan->rs_samplescan)
            pgstat_count_heap_scan(scan->rs_rd);
    }
    
    

### 三、跟踪分析

测试脚本如下

    
    
    testdb=# explain select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je 
    testdb-# from t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je 
    testdb(#                         from t_grxx gr inner join t_jfxx jf 
    testdb(#                                        on gr.dwbh = dw.dwbh 
    testdb(#                                           and gr.grbh = jf.grbh) grjf
    testdb-# order by dw.dwbh;
                                            QUERY PLAN                                        
    ------------------------------------------------------------------------------------------     Sort  (cost=20070.93..20320.93 rows=100000 width=47)
       Sort Key: dw.dwbh
       ->  Hash Join  (cost=3754.00..8689.61 rows=100000 width=47)
             Hash Cond: ((gr.dwbh)::text = (dw.dwbh)::text)
             ->  Hash Join  (cost=3465.00..8138.00 rows=100000 width=31)
                   Hash Cond: ((jf.grbh)::text = (gr.grbh)::text)
                   ->  Seq Scan on t_jfxx jf  (cost=0.00..1637.00 rows=100000 width=20)
                   ->  Hash  (cost=1726.00..1726.00 rows=100000 width=16)
                         ->  Seq Scan on t_grxx gr  (cost=0.00..1726.00 rows=100000 width=16)
             ->  Hash  (cost=164.00..164.00 rows=10000 width=20)
                   ->  Seq Scan on t_dwxx dw  (cost=0.00..164.00 rows=10000 width=20)
    (11 rows)
    
    

启动gdb,设置断点,进入SeqNext

    
    
    (gdb) b SeqNext
    Breakpoint 1 at 0x7156b2: file nodeSeqscan.c, line 60.
    (gdb) c
    Continuing.
    
    Breakpoint 1, SeqNext (node=0x2ed1588) at nodeSeqscan.c:60
    60      scandesc = node->ss.ss_currentScanDesc;
    

变量赋值

    
    
    60      scandesc = node->ss.ss_currentScanDesc;
    (gdb) n
    61      estate = node->ss.ps.state;
    (gdb) 
    62      direction = estate->es_direction;
    (gdb) 
    63      slot = node->ss.ss_ScanTupleSlot;
    (gdb) 
    65      if (scandesc == NULL)
    

scandesc为NULL,进入初始化,调用heap_beginscan

    
    
    (gdb) p scandesc
    $1 = (HeapScanDesc) 0x0
    

进入heap_beginscan/heap_beginscan_internal函数

    
    
    (gdb) n
    71          scandesc = heap_beginscan(node->ss.ss_currentRelation,
    (gdb) step
    heap_beginscan (relation=0x7fb27c488a90, snapshot=0x2e0b8f0, nkeys=0, key=0x0) at heapam.c:1407
    1407        return heap_beginscan_internal(relation, snapshot, nkeys, key, NULL,
    (gdb) step
    heap_beginscan_internal (relation=0x7fb27c488a90, snapshot=0x2e0b8f0, nkeys=0, key=0x0, parallel_scan=0x0, 
        allow_strat=true, allow_sync=true, allow_pagemode=true, is_bitmapscan=false, is_samplescan=false, temp_snap=false)
        at heapam.c:1469
    1469        RelationIncrementReferenceCount(relation);    
    

heap_beginscan_internal->增加relation参考计数

    
    
    1469        RelationIncrementReferenceCount(relation);
    (gdb) n
    

heap_beginscan_internal->初始化HeapScanDesc结构体

    
    
    1474        scan = (HeapScanDesc) palloc(sizeof(HeapScanDescData));
    (gdb) 
    1476        scan->rs_rd = relation;
    (gdb) 
    1477        scan->rs_snapshot = snapshot;
    (gdb) 
    1478        scan->rs_nkeys = nkeys;
    (gdb) 
    1479        scan->rs_bitmapscan = is_bitmapscan;
    (gdb) 
    1480        scan->rs_samplescan = is_samplescan;
    (gdb) 
    1481        scan->rs_strategy = NULL;   /* set in initscan */
    (gdb) 
    1482        scan->rs_allow_strat = allow_strat;
    (gdb) 
    1483        scan->rs_allow_sync = allow_sync;
    (gdb) 
    1484        scan->rs_temp_snap = temp_snap;
    (gdb) 
    1485        scan->rs_parallel = parallel_scan;
    (gdb) 
    1490        scan->rs_pageatatime = allow_pagemode && IsMVCCSnapshot(snapshot);
    (gdb) 
    1503        if (!is_bitmapscan)
    

heap_beginscan_internal->非位图扫描,谓词锁定

    
    
    1503        if (!is_bitmapscan)
    (gdb) p is_bitmapscan
    $2 = false
    (gdb) n
    1504            PredicateLockRelation(relation, snapshot);
    (gdb) 
    1507        scan->rs_ctup.t_tableOid = RelationGetRelid(relation);
    

heap_beginscan_internal->进入initscan函数

    
    
    (gdb) n
    1513        if (nkeys > 0)
    (gdb) 
    1516            scan->rs_key = NULL;
    (gdb) 
    1518        initscan(scan, key, false);
    (gdb) step
    initscan (scan=0x2ee4568, key=0x0, keep_startblock=false) at heapam.c:236
    236     if (scan->rs_parallel != NULL)
    

heap_beginscan_internal->relation的大小相对于buffer并不大(<25%),不使用访问策略(批量读取)&同步扫描

    
    
    (gdb) n
    239         scan->rs_nblocks = RelationGetNumberOfBlocks(scan->rs_rd);
    (gdb) 
    253     if (!RelationUsesLocalBuffers(scan->rs_rd) &&
    (gdb) 
    254         scan->rs_nblocks > NBuffers / 4)
    (gdb) 
    253     if (!RelationUsesLocalBuffers(scan->rs_rd) &&
    (gdb) 
    260         allow_strat = allow_sync = false;
    

heap_beginscan_internal->设置其他变量

    
    
    312     if (key != NULL)
    (gdb) 
    320     if (!scan->rs_bitmapscan && !scan->rs_samplescan)
    (gdb) 
    321         pgstat_count_heap_scan(scan->rs_rd);
    (gdb) 
    322 }
    (gdb) 
    

heap_beginscan_internal->回到heap_beginscan_internal,完成初始化

    
    
    (gdb) n
    heap_beginscan_internal (relation=0x7fb27c488a90, snapshot=0x2e0b8f0, nkeys=0, key=0x0, parallel_scan=0x0, 
        allow_strat=true, allow_sync=true, allow_pagemode=true, is_bitmapscan=false, is_samplescan=false, temp_snap=false)
        at heapam.c:1520
    1520        return scan;
    (gdb) p *scan
    $4 = {rs_rd = 0x7fb27c488a90, rs_snapshot = 0x2e0b8f0, rs_nkeys = 0, rs_key = 0x0, rs_bitmapscan = false, 
      rs_samplescan = false, rs_pageatatime = true, rs_allow_strat = true, rs_allow_sync = true, rs_temp_snap = false, 
      rs_nblocks = 726, rs_startblock = 0, rs_numblocks = 4294967295, rs_strategy = 0x0, rs_syncscan = false, 
      rs_inited = false, rs_ctup = {t_len = 2139062143, t_self = {ip_blkid = {bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, 
        t_tableOid = 16742, t_data = 0x0}, rs_cblock = 4294967295, rs_cbuf = 0, rs_parallel = 0x0, rs_cindex = 2139062143, 
      rs_ntuples = 2139062143, rs_vistuples = {32639 <repeats 291 times>}}
    (gdb) 
    

DONE!

### 四、参考资料

[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

