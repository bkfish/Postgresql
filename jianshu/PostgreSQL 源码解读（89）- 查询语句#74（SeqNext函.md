本节是SeqNext函数介绍的第二部分，主要介绍了SeqNext->heap_getnext函数的实现逻辑。

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

heap_getnext函数从数据表中获取下一个tuple.根据ScanDesc->rs_pageatatime的设定,如为T,则调用heapgettup_pagemode函数,使用page-at-a-time模式提取元组,否则调用函数heapgettup使用常规模式提取.

    
    
    HeapTuple
    heap_getnext(HeapScanDesc scan, ScanDirection direction)
    {
        /* Note: no locking manipulations needed */
        //注意:无需锁定处理
        HEAPDEBUG_1;                /* heap_getnext( info ) */
    
        if (scan->rs_pageatatime)
            heapgettup_pagemode(scan, direction,
                                scan->rs_nkeys, scan->rs_key);//page-at-a-time模式
        else
            heapgettup(scan, direction, scan->rs_nkeys, scan->rs_key);//常规模式
    
        if (scan->rs_ctup.t_data == NULL)//已完成
        {
            HEAPDEBUG_2;            /* heap_getnext returning EOS */
            return NULL;
        }
    
        /*
         * if we get here it means we have a new current scan tuple, so point to
         * the proper return buffer and return the tuple.
         * 如果实现逻辑到这里，意味着有一个新的当前扫描元组，指向正确的返回缓冲区并返回元组。
         */
        HEAPDEBUG_3;                /* heap_getnext returning tuple */
    
        pgstat_count_heap_getnext(scan->rs_rd);
    
        return &(scan->rs_ctup);
    }
    
    
    /* ----------------     *      heapgettup_pagemode - fetch next heap tuple in page-at-a-time mode
     *      heapgettup_pagemode - 以page-at-a-time模式提取下一个元组
     *
     *      Same API as heapgettup, but used in page-at-a-time mode
     *      与heapgettup是相同的API,只不过仅用于page-at-a-time模式
     *
     * The internal logic is much the same as heapgettup's too, but there are some
     * differences: we do not take the buffer content lock (that only needs to
     * happen inside heapgetpage), and we iterate through just the tuples listed
     * in rs_vistuples[] rather than all tuples on the page.  Notice that
     * lineindex is 0-based, where the corresponding loop variable lineoff in
     * heapgettup is 1-based.
     * 内部逻辑与heapgettup的逻辑大体相同，但也有一些区别:
     *      不使用缓冲区锁(这只需要在heapgetpage内发生)，只迭代rs_vistuples[]中列出的元组，而不是页面上的所有元组。
     * 注意，lineindex是从0开始的，而heapgettup中对应的循环变量lineoff是从1开始的。
     * ----------------     */
    static void
    heapgettup_pagemode(HeapScanDesc scan,//ScanDesc
                        ScanDirection dir,//扫描方向
                        int nkeys,//键个数
                        ScanKey key)//扫描键
    {
        HeapTuple   tuple = &(scan->rs_ctup);//当前扫描的Tuple(scan->rs_ctup类型为HeapTupleData)
        bool        backward = ScanDirectionIsBackward(dir);//是否后向扫描
        BlockNumber page;//page编号
        bool        finished;//是否已完成
        Page        dp;//page
        int         lines;//
        int         lineindex;
        OffsetNumber lineoff;//偏移
        int         linesleft;
        ItemId      lpp;//项ID
    
        /*
         * calculate next starting lineindex, given scan direction
         * 给定扫描方向,计算下一个开始的lineindex
         */
        if (ScanDirectionIsForward(dir))
        {
            //前向扫描
            if (!scan->rs_inited)
            {
                //尚未初始化
                /*
                 * return null immediately if relation is empty
                 * 如relation为空,则马上返回null
                 */
                if (scan->rs_nblocks == 0 || scan->rs_numblocks == 0)
                {
                    Assert(!BufferIsValid(scan->rs_cbuf));
                    tuple->t_data = NULL;
                    return;
                }
                //判断是否并行扫描
                if (scan->rs_parallel != NULL)
                {
                    //并行扫描初始化
                    heap_parallelscan_startblock_init(scan);
    
                    page = heap_parallelscan_nextpage(scan);
    
                    /* Other processes might have already finished the scan. */
                    //其他进程可能已经完成了扫描
                    if (page == InvalidBlockNumber)
                    {
                        Assert(!BufferIsValid(scan->rs_cbuf));
                        tuple->t_data = NULL;
                        return;
                    }
                }
                else
                    page = scan->rs_startblock; /* 非并行扫描,返回开始页;first page */
                //获取page
                heapgetpage(scan, page);
                //初始化lineindex为0
                lineindex = 0;
                //设置初始化标记为T
                scan->rs_inited = true;
            }
            else
            {
                //已完成初始化
                /* continue from previously returned page/tuple */
                //从上一次返回的page/tuple处开始
                page = scan->rs_cblock; /* 当前页;current page */
                lineindex = scan->rs_cindex + 1;//加+1
            }
            //根据buffer获取相应的page
            dp = BufferGetPage(scan->rs_cbuf);
            //验证快照是否过旧
            TestForOldSnapshot(scan->rs_snapshot, scan->rs_rd, dp);
            lines = scan->rs_ntuples;
            /* page and lineindex now reference the next visible tid */
            //page和lineindex现在依赖于下一个可见的tid
            linesleft = lines - lineindex;
        }
        else if (backward)
        {
            //反向扫描
            /* backward parallel scan not supported */
            //并行后向扫描目前不支持
            Assert(scan->rs_parallel == NULL);
    
            if (!scan->rs_inited)
            {
                /*
                 * return null immediately if relation is empty
                 * 同正向扫描
                 */
                if (scan->rs_nblocks == 0 || scan->rs_numblocks == 0)
                {
                    Assert(!BufferIsValid(scan->rs_cbuf));
                    tuple->t_data = NULL;
                    return;
                }
    
                /*
                 * Disable reporting to syncscan logic in a backwards scan; it's
                 * not very likely anyone else is doing the same thing at the same
                 * time, and much more likely that we'll just bollix things for
                 * forward scanners.
                 * 在反向扫描中禁用对syncscan逻辑的报告;
                 * 不太可能有其他人在同一时间做同样的事情，更可能的是我们会把向前扫描的事情搞砸:( 
                 */
                scan->rs_syncscan = false;//禁用sync扫描
                /* start from last page of the scan */
                //从最后一个page开始
                if (scan->rs_startblock > 0)
                    page = scan->rs_startblock - 1;//已开始扫描,减一
                else
                    page = scan->rs_nblocks - 1;//未开始扫描,页数减一
                //获取page
                heapgetpage(scan, page);
            }
            else
            {
                /* continue from previously returned page/tuple */
                //获取当前page
                page = scan->rs_cblock; /* current page */
            }
            //根据buffer获取page
            dp = BufferGetPage(scan->rs_cbuf);
            //快照是否过旧判断
            TestForOldSnapshot(scan->rs_snapshot, scan->rs_rd, dp);
            //行数
            lines = scan->rs_ntuples;
    
            if (!scan->rs_inited)
            {
                //未初始化,初始化相关信息
                lineindex = lines - 1;
                scan->rs_inited = true;
            }
            else
            {
                //已完成初始化,index-1
                lineindex = scan->rs_cindex - 1;
            }
            /* page and lineindex now reference the previous visible tid */
    
            linesleft = lineindex + 1;
        }
        else
        {
            //既不是正向也不是反向,扫描不能移动
            /*
             * ``no movement'' scan direction: refetch prior tuple
             * ``no movement'' scan direction: 取回之前的元组
             */
            if (!scan->rs_inited)
            {
                Assert(!BufferIsValid(scan->rs_cbuf));
                tuple->t_data = NULL;
                return;
            }
    
            page = ItemPointerGetBlockNumber(&(tuple->t_self));
            if (page != scan->rs_cblock)
                heapgetpage(scan, page);
    
            /* Since the tuple was previously fetched, needn't lock page here */
            dp = BufferGetPage(scan->rs_cbuf);
            TestForOldSnapshot(scan->rs_snapshot, scan->rs_rd, dp);
            lineoff = ItemPointerGetOffsetNumber(&(tuple->t_self));
            lpp = PageGetItemId(dp, lineoff);
            Assert(ItemIdIsNormal(lpp));
    
            tuple->t_data = (HeapTupleHeader) PageGetItem((Page) dp, lpp);
            tuple->t_len = ItemIdGetLength(lpp);
    
            /* check that rs_cindex is in sync */
            Assert(scan->rs_cindex < scan->rs_ntuples);
            Assert(lineoff == scan->rs_vistuples[scan->rs_cindex]);
    
            return;
        }
    
        /*
         * advance the scan until we find a qualifying tuple or run out of stuff
         * to scan
         * 推进扫描，直到找到一个合格的元组或耗尽了要扫描的东西
         */
        for (;;)
        {
            while (linesleft > 0)//该page中剩余的行数>0(linesleft > 0),亦即扫描该page
            {
                //获得偏移
                lineoff = scan->rs_vistuples[lineindex];
                //获取ItemID
                lpp = PageGetItemId(dp, lineoff);
                Assert(ItemIdIsNormal(lpp));
                //获取元组头部数据
                tuple->t_data = (HeapTupleHeader) PageGetItem((Page) dp, lpp);
                //大小
                tuple->t_len = ItemIdGetLength(lpp);
                //设置指针(ItemPointer是ItemPointerData结构体指针)
                ItemPointerSet(&(tuple->t_self), page, lineoff);
    
                /*
                 * if current tuple qualifies, return it.
                 * 当前元组满足条件,返回
                 */
                if (key != NULL)
                {
                    //扫描键不为NULL
                    bool        valid;
                    //验证是否符合要求
                    HeapKeyTest(tuple, RelationGetDescr(scan->rs_rd),
                                nkeys, key, valid);
                    if (valid)
                    {
                        //满足,则返回
                        scan->rs_cindex = lineindex;
                        return;
                    }
                }
                else
                {
                    //不存在扫描键,直接返回
                    scan->rs_cindex = lineindex;
                    return;
                }
    
                /*
                 * otherwise move to the next item on the page
                 * 不满足扫描键要求,继续page中的下一个Item
                 */
                --linesleft;//减少剩余计数
                if (backward)
                    --lineindex;//反向,减一
                else
                    ++lineindex;//正向,加一
            }
    
            /*
             * if we get here, it means we've exhausted the items on this page and
             * it's time to move to the next.
             * 如果执行到这里，意味着已经把这一页的内容已完成，是时候转移到下一页了。
             */
            if (backward)//反向
            {
                finished = (page == scan->rs_startblock) ||
                    (scan->rs_numblocks != InvalidBlockNumber ? --scan->rs_numblocks == 0 : false);//判断是否已完成
                if (page == 0)//如page为0
                    page = scan->rs_nblocks;//重置为block数
                page--;//page减一
            }
            else if (scan->rs_parallel != NULL)
            {
                //并行扫描
                page = heap_parallelscan_nextpage(scan);
                finished = (page == InvalidBlockNumber);
            }
            else
            {
                //正向扫描
                page++;//page加一
                if (page >= scan->rs_nblocks)
                    page = 0;//page超出总数,重置为0
                finished = (page == scan->rs_startblock) ||
                    (scan->rs_numblocks != InvalidBlockNumber ? --scan->rs_numblocks == 0 : false);//判断是否已完成
    
                /*
                 * Report our new scan position for synchronization purposes. We
                 * don't do that when moving backwards, however. That would just
                 * mess up any other forward-moving scanners.
                 * 报告新扫描位置用于同步。然而，在反向扫描时不会这样做。那只会把其他前进的扫描器弄乱。
                 *
                 * Note: we do this before checking for end of scan so that the
                 * final state of the position hint is back at the start of the
                 * rel.  That's not strictly necessary, but otherwise when you run
                 * the same query multiple times the starting position would shift
                 * a little bit backwards on every invocation, which is confusing.
                 * We don't guarantee any specific ordering in general, though.
                 * 注意:在扫描结束前做这个前置检查以便位置的最终状态是在rel开始之后。
                 * 这不是严格必要的,否则当多次运行相同的查询时,起始位置将稍微有一点靠后,这相当令人困惑。
                 * 不过，我们一般来说不保证任何特定的顺序。
                 */
                if (scan->rs_syncscan)
                    //同步扫描,报告位置
                    ss_report_location(scan->rs_rd, page);
            }
    
            /*
             * return NULL if we've exhausted all the pages
             * 已耗尽所有page,返回NULL
             */
            if (finished)
            {
                if (BufferIsValid(scan->rs_cbuf))
                    ReleaseBuffer(scan->rs_cbuf);
                scan->rs_cbuf = InvalidBuffer;
                scan->rs_cblock = InvalidBlockNumber;
                tuple->t_data = NULL;
                scan->rs_inited = false;
                return;
            }
            //获取下一个page,继续循环
            heapgetpage(scan, page);
            //执行类似的逻辑
            dp = BufferGetPage(scan->rs_cbuf);
            TestForOldSnapshot(scan->rs_snapshot, scan->rs_rd, dp);
            lines = scan->rs_ntuples;
            linesleft = lines;
            if (backward)
                lineindex = lines - 1;
            else
                lineindex = 0;//ItemID从0开始
        }
    }
    
    
    
    /* ----------------     *      heapgettup - fetch next heap tuple
     *      提取下一个元组
     *
     *      Initialize the scan if not already done; then advance to the next
     *      tuple as indicated by "dir"; return the next tuple in scan->rs_ctup,
     *      or set scan->rs_ctup.t_data = NULL if no more tuples.
     *      如尚未完成初始化，则初始化扫描;
     *          然后按照“dir”的方向指示前进到下一个元组;
     *          返回下一个元组scan->rs_ctup，如无元组则设置scan->rs_ctup.t_data = NULL。
     *
     * dir == NoMovementScanDirection means "re-fetch the tuple indicated
     * by scan->rs_ctup".
     * dir == NoMovementScanDirection意味着"使用scan->rs_ctup重新提取元组"
     *
     * Note: the reason nkeys/key are passed separately, even though they are
     * kept in the scan descriptor, is that the caller may not want us to check
     * the scankeys.
     * 注意:nkeys/key是单独传递的，即使它们保存在扫描描述符中，原因是调用者可能不希望我们检查scankeys。
     *
     * Note: when we fall off the end of the scan in either direction, we
     * reset rs_inited.  This means that a further request with the same
     * scan direction will restart the scan, which is a bit odd, but a
     * request with the opposite scan direction will start a fresh scan
     * in the proper direction.  The latter is required behavior for cursors,
     * while the former case is generally undefined behavior in Postgres
     * so we don't care too much.
     * 注意:当我们从扫描结束的任意方向前进时，需要重置rs_inited标记。
     * 这意味着具有相同扫描方向的进一步请求将重新启动扫描，这有点奇怪，
     * 但是具有相反扫描方向的请求将在正确的方向上重新启动扫描。
     * 后者是游标需要的行为，而前者通常是Postgres中未定义的行为，所以我们不太关心。
     * ----------------     */
    static void
    heapgettup(HeapScanDesc scan,//ScanDesc
               ScanDirection dir,//扫描方向
               int nkeys,//扫描键个数
               ScanKey key)//扫描键
    {
        HeapTuple   tuple = &(scan->rs_ctup);//当前的tuple
        Snapshot    snapshot = scan->rs_snapshot;//快照
        bool        backward = ScanDirectionIsBackward(dir);//
        BlockNumber page;
        bool        finished;
        Page        dp;
        int         lines;
        OffsetNumber lineoff;
        int         linesleft;
        ItemId      lpp;
    
        /*
         * calculate next starting lineoff, given scan direction
         * 给定扫描方向,计算下一个开始的偏移
         */
        if (ScanDirectionIsForward(dir))
        {
            //参照heapgettup_pagemode注释
            if (!scan->rs_inited)
            {
                /*
                 * return null immediately if relation is empty
                 */
                if (scan->rs_nblocks == 0 || scan->rs_numblocks == 0)
                {
                    Assert(!BufferIsValid(scan->rs_cbuf));
                    tuple->t_data = NULL;
                    return;
                }
                if (scan->rs_parallel != NULL)
                {
                    heap_parallelscan_startblock_init(scan);
    
                    page = heap_parallelscan_nextpage(scan);
    
                    /* Other processes might have already finished the scan. */
                    if (page == InvalidBlockNumber)
                    {
                        Assert(!BufferIsValid(scan->rs_cbuf));
                        tuple->t_data = NULL;
                        return;
                    }
                }
                else
                    page = scan->rs_startblock; /* first page */
                heapgetpage(scan, page);
                lineoff = FirstOffsetNumber;    /* first offnum */
                scan->rs_inited = true;
            }
            else
            {
                /* continue from previously returned page/tuple */
                page = scan->rs_cblock; /* current page */
                lineoff =           /* next offnum */
                    OffsetNumberNext(ItemPointerGetOffsetNumber(&(tuple->t_self)));
            }
    
            LockBuffer(scan->rs_cbuf, BUFFER_LOCK_SHARE);
    
            dp = BufferGetPage(scan->rs_cbuf);
            TestForOldSnapshot(snapshot, scan->rs_rd, dp);
            lines = PageGetMaxOffsetNumber(dp);
            /* page and lineoff now reference the physically next tid */
    
            linesleft = lines - lineoff + 1;
        }
        else if (backward)
        {
            //参照heapgettup_pagemode注释
            /* backward parallel scan not supported */
            Assert(scan->rs_parallel == NULL);
    
            if (!scan->rs_inited)
            {
                /*
                 * return null immediately if relation is empty
                 */
                if (scan->rs_nblocks == 0 || scan->rs_numblocks == 0)
                {
                    Assert(!BufferIsValid(scan->rs_cbuf));
                    tuple->t_data = NULL;
                    return;
                }
    
                /*
                 * Disable reporting to syncscan logic in a backwards scan; it's
                 * not very likely anyone else is doing the same thing at the same
                 * time, and much more likely that we'll just bollix things for
                 * forward scanners.
                 */
                scan->rs_syncscan = false;
                /* start from last page of the scan */
                if (scan->rs_startblock > 0)
                    page = scan->rs_startblock - 1;
                else
                    page = scan->rs_nblocks - 1;
                heapgetpage(scan, page);
            }
            else
            {
                /* continue from previously returned page/tuple */
                page = scan->rs_cblock; /* current page */
            }
            //锁定buffer(BUFFER_LOCK_SHARE)
            //这里跟pagemode不同,需要锁定
            LockBuffer(scan->rs_cbuf, BUFFER_LOCK_SHARE);
            //
            dp = BufferGetPage(scan->rs_cbuf);
            TestForOldSnapshot(snapshot, scan->rs_rd, dp);
            //获取最大偏移
            lines = PageGetMaxOffsetNumber(dp);
    
            if (!scan->rs_inited)
            {
                lineoff = lines;    /* 设置为最后的偏移;final offnum */
                scan->rs_inited = true;
            }
            else
            {
                lineoff =           /* previous offnum */
                    OffsetNumberPrev(ItemPointerGetOffsetNumber(&(tuple->t_self)));
            }
            /* page and lineoff now reference the physically previous tid */
    
            linesleft = lineoff;
        }
        else
        {
            /*
             * ``no movement'' scan direction: refetch prior tuple
             */
            if (!scan->rs_inited)
            {
                Assert(!BufferIsValid(scan->rs_cbuf));
                tuple->t_data = NULL;
                return;
            }
    
            page = ItemPointerGetBlockNumber(&(tuple->t_self));
            if (page != scan->rs_cblock)
                heapgetpage(scan, page);
    
            /* Since the tuple was previously fetched, needn't lock page here */
            dp = BufferGetPage(scan->rs_cbuf);
            TestForOldSnapshot(snapshot, scan->rs_rd, dp);
            lineoff = ItemPointerGetOffsetNumber(&(tuple->t_self));
            lpp = PageGetItemId(dp, lineoff);
            Assert(ItemIdIsNormal(lpp));
    
            tuple->t_data = (HeapTupleHeader) PageGetItem((Page) dp, lpp);
            tuple->t_len = ItemIdGetLength(lpp);
    
            return;
        }
    
        /*
         * advance the scan until we find a qualifying tuple or run out of stuff
         * to scan
         */
        lpp = PageGetItemId(dp, lineoff);
        for (;;)
        {
            while (linesleft > 0)
            {
                if (ItemIdIsNormal(lpp))
                {
                    bool        valid;
    
                    tuple->t_data = (HeapTupleHeader) PageGetItem((Page) dp, lpp);
                    tuple->t_len = ItemIdGetLength(lpp);
                    ItemPointerSet(&(tuple->t_self), page, lineoff);
    
                    /*
                     * if current tuple qualifies, return it.
                     */
                    //判断是否满足可见性(MVCC机制)
                    valid = HeapTupleSatisfiesVisibility(tuple,
                                                         snapshot,
                                                         scan->rs_cbuf);
                    //检查是否存在Serializable冲突
                    CheckForSerializableConflictOut(valid, scan->rs_rd, tuple,
                                                    scan->rs_cbuf, snapshot);
    
                    if (valid && key != NULL)
                        HeapKeyTest(tuple, RelationGetDescr(scan->rs_rd),
                                    nkeys, key, valid);//扫描键验证
    
                    if (valid)
                    {
                        //解锁buffer,返回
                        LockBuffer(scan->rs_cbuf, BUFFER_LOCK_UNLOCK);
                        return;
                    }
                }
    
                /*
                 * otherwise move to the next item on the page
                 */
                --linesleft;//下一个Item
                if (backward)
                {
                    --lpp;          /* move back in this page's ItemId array */
                    --lineoff;
                }
                else
                {
                    ++lpp;          /* move forward in this page's ItemId array */
                    ++lineoff;
                }
            }
    
            /*
             * if we get here, it means we've exhausted the items on this page and
             * it's time to move to the next.
             * 下一页
             */
            //解锁
            LockBuffer(scan->rs_cbuf, BUFFER_LOCK_UNLOCK);
    
            /*
             * advance to next/prior page and detect end of scan
             */
            if (backward)
            {
                finished = (page == scan->rs_startblock) ||
                    (scan->rs_numblocks != InvalidBlockNumber ? --scan->rs_numblocks == 0 : false);
                if (page == 0)
                    page = scan->rs_nblocks;
                page--;
            }
            else if (scan->rs_parallel != NULL)
            {
                page = heap_parallelscan_nextpage(scan);
                finished = (page == InvalidBlockNumber);
            }
            else
            {
                page++;
                if (page >= scan->rs_nblocks)
                    page = 0;
                finished = (page == scan->rs_startblock) ||
                    (scan->rs_numblocks != InvalidBlockNumber ? --scan->rs_numblocks == 0 : false);
    
                /*
                 * Report our new scan position for synchronization purposes. We
                 * don't do that when moving backwards, however. That would just
                 * mess up any other forward-moving scanners.
                 *
                 * Note: we do this before checking for end of scan so that the
                 * final state of the position hint is back at the start of the
                 * rel.  That's not strictly necessary, but otherwise when you run
                 * the same query multiple times the starting position would shift
                 * a little bit backwards on every invocation, which is confusing.
                 * We don't guarantee any specific ordering in general, though.
                 */
                if (scan->rs_syncscan)
                    ss_report_location(scan->rs_rd, page);
            }
    
            /*
             * return NULL if we've exhausted all the pages
             */
            if (finished)
            {
                if (BufferIsValid(scan->rs_cbuf))
                    ReleaseBuffer(scan->rs_cbuf);
                scan->rs_cbuf = InvalidBuffer;
                scan->rs_cblock = InvalidBlockNumber;
                tuple->t_data = NULL;
                scan->rs_inited = false;
                return;
            }
    
            heapgetpage(scan, page);
            //锁定buffer
            LockBuffer(scan->rs_cbuf, BUFFER_LOCK_SHARE);
    
            dp = BufferGetPage(scan->rs_cbuf);
            TestForOldSnapshot(snapshot, scan->rs_rd, dp);
            lines = PageGetMaxOffsetNumber((Page) dp);
            linesleft = lines;
            if (backward)
            {
                lineoff = lines;
                lpp = PageGetItemId(dp, lines);
            }
            else
            {
                lineoff = FirstOffsetNumber;
                lpp = PageGetItemId(dp, FirstOffsetNumber);
            }
        }
    }
    
    
    //--------------------------------------------------- heapgetpage
    
    /*
     * heapgetpage - subroutine for heapgettup()
     * heapgettup()的子函数
     * 
     * This routine reads and pins the specified page of the relation.
     * In page-at-a-time mode it performs additional work, namely determining
     * which tuples on the page are visible.
     * 这个例程读取并pins固定关联的指定页面。
     * 在逐页page-at-a-time模式中，它执行额外的工作，即确定页面上哪些元组可见。
     */
    void
    heapgetpage(HeapScanDesc scan, BlockNumber page)
    {
        Buffer      buffer;
        Snapshot    snapshot;
        Page        dp;
        int         lines;
        int         ntup;
        OffsetNumber lineoff;
        ItemId      lpp;
        bool        all_visible;
    
        Assert(page < scan->rs_nblocks);
    
        /* release previous scan buffer, if any */
        //释放上一次扫描使用的buffer
        if (BufferIsValid(scan->rs_cbuf))
        {
            ReleaseBuffer(scan->rs_cbuf);
            scan->rs_cbuf = InvalidBuffer;
        }
    
        /*
         * Be sure to check for interrupts at least once per page.  Checks at
         * higher code levels won't be able to stop a seqscan that encounters many
         * pages' worth of consecutive dead tuples.
         * 务必记住每页扫描完毕都要检查一次中断!
         * 更上层代码的检查不能够停止遇到很多无效tuples的seqscan
         */
        CHECK_FOR_INTERRUPTS();
    
        /* read page using selected strategy */
        //使用选定的策略读取page
        //赋值:rs_cbuf & rs_cblock
        scan->rs_cbuf = ReadBufferExtended(scan->rs_rd, MAIN_FORKNUM, page,
                                           RBM_NORMAL, scan->rs_strategy);
        scan->rs_cblock = page;
        //如非page-at-a-time模式,直接返回
        if (!scan->rs_pageatatime)
            return;
    
        //page-at-a-time模式
        buffer = scan->rs_cbuf;
        snapshot = scan->rs_snapshot;
    
        /*
         * Prune and repair fragmentation for the whole page, if possible.
         * 如可能，修剪(Prune)和修复整个页面的碎片。
         */
        heap_page_prune_opt(scan->rs_rd, buffer);
    
        /*
         * We must hold share lock on the buffer content while examining tuple
         * visibility.  Afterwards, however, the tuples we have found to be
         * visible are guaranteed good as long as we hold the buffer pin.
         * 在检查元组可见性时,必须持有共享锁.
         * 在上锁之后，只要我们持有buffer pin，发现可见元组的逻辑会工作得很好。
         */
        //上锁BUFFER_LOCK_SHARE
        LockBuffer(buffer, BUFFER_LOCK_SHARE);
        //获取page
        dp = BufferGetPage(buffer);
        //验证快照是否过旧
        TestForOldSnapshot(snapshot, scan->rs_rd, dp);
        //行数
        lines = PageGetMaxOffsetNumber(dp);
        //初始化
        ntup = 0;
    
        /*
         * If the all-visible flag indicates that all tuples on the page are
         * visible to everyone, we can skip the per-tuple visibility tests.
         * 如all-visible标志表明页面上的所有元组对每个人都可见，那么可以跳过每个元组可见性测试。
         * 
         * Note: In hot standby, a tuple that's already visible to all
         * transactions in the master might still be invisible to a read-only
         * transaction in the standby. We partly handle this problem by tracking
         * the minimum xmin of visible tuples as the cut-off XID while marking a
         * page all-visible on master and WAL log that along with the visibility
         * map SET operation. In hot standby, we wait for (or abort) all
         * transactions that can potentially may not see one or more tuples on the
         * page. That's how index-only scans work fine in hot standby. A crucial
         * difference between index-only scans and heap scans is that the
         * index-only scan completely relies on the visibility map where as heap
         * scan looks at the page-level PD_ALL_VISIBLE flag. We are not sure if
         * the page-level flag can be trusted in the same way, because it might
         * get propagated somehow without being explicitly WAL-logged, e.g. via a
         * full page write. Until we can prove that beyond doubt, let's check each
         * tuple for visibility the hard way.
         * 注意:在热备份中，对于主服务器中的所有事务都可见的元组可能对备用服务器中的只读事务仍然不可见。
         *   通过跟踪可见元组的最小xmin作为截止XID来部分处理这个问题，
         * 同时在master和WAL log上标记一个页面全可见，以及可见映射设置操作。
         * 在热备份中，我们等待(或中止)所有可能在页面上看不到一个或多个元组的事务。
         * 这就是只有索引的扫描在热待机状态下运行良好的原因。
         * 仅索引扫描和堆扫描之间的一个关键区别是，仅索引扫描完全依赖于可见性映射，
         *   当堆扫描查看页面级别的PD_ALL_VISIBLE标志时，可见性映射将依赖于此。
         * 我们不确定是否可以以同样的方式信任页面级别的标志，因为它可能会以某种方式传播，
         *   而不会被显式地保留在日志中，例如通过整个页面写入。
         * 在我们能够毫无疑问地证明这一点之前，必须以艰苦的方式检查每个元组的可见性。
         */
        //验证可见性
        all_visible = PageIsAllVisible(dp) && !snapshot->takenDuringRecovery;
        //扫描Item
        for (lineoff = FirstOffsetNumber, lpp = PageGetItemId(dp, lineoff);
             lineoff <= lines;
             lineoff++, lpp++)
        {
            if (ItemIdIsNormal(lpp))
            {
                HeapTupleData loctup;
                bool        valid;
    
                loctup.t_tableOid = RelationGetRelid(scan->rs_rd);
                loctup.t_data = (HeapTupleHeader) PageGetItem((Page) dp, lpp);
                loctup.t_len = ItemIdGetLength(lpp);
                ItemPointerSet(&(loctup.t_self), page, lineoff);
    
                if (all_visible)
                    valid = true;
                else
                    valid = HeapTupleSatisfiesVisibility(&loctup, snapshot, buffer);
    
                CheckForSerializableConflictOut(valid, scan->rs_rd, &loctup,
                                                buffer, snapshot);
    
                if (valid)
                    scan->rs_vistuples[ntup++] = lineoff;
            }
        }
        //done,释放共享锁
        LockBuffer(buffer, BUFFER_LOCK_UNLOCK);
    
        Assert(ntup <= MaxHeapTuplesPerPage);
        scan->rs_ntuples = ntup;
    }
    
    //--------------------------------------------------- PageGetMaxOffsetNumber
    
    /*
     * PageGetMaxOffsetNumber
     *      Returns the maximum offset number used by the given page.
     *      Since offset numbers are 1-based, this is also the number
     *      of items on the page.
     *      返回给定页面使用的最大偏移。由于偏移量是基于1的，所以这也是页面上Item的数量。
     *
     *      NOTE: if the page is not initialized (pd_lower == 0), we must
     *      return zero to ensure sane behavior.  Accept double evaluation
     *      of the argument so that we can ensure this.
     *      注意:如果页面没有初始化(pd_lower == 0)，我们必须返回0以确保正常的行为。
     *           接受对参数的双重解析，这样我们才能确保这一点。
     */
    #define PageGetMaxOffsetNumber(page) \
        (((PageHeader) (page))->pd_lower <= SizeOfPageHeaderData ? 0 : \
         ((((PageHeader) (page))->pd_lower - SizeOfPageHeaderData) \
          / sizeof(ItemIdData)))
    
    
    //--------------------------------------------------- TestForOldSnapshot
    /*
     * Check whether the given snapshot is too old to have safely read the given
     * page from the given table.  If so, throw a "snapshot too old" error.
     * 检查给定的快照是否过旧，不能安全地从给定的表中读取给定的页面。如果是，抛出一个“snapshot too old”错误。
     *
     * This test generally needs to be performed after every BufferGetPage() call
     * that is executed as part of a scan.  It is not needed for calls made for
     * modifying the page (for example, to position to the right place to insert a
     * new index tuple or for vacuuming).  It may also be omitted where calls to
     * lower-level functions will have already performed the test.
     * 这个测试通常需要在作为扫描的一部分执行,在每个BufferGetPage()调用之后执行。
     * 不需要通过调用修改页面(例如，定位到正确的位置以插入一个新的索引元组或进行清理)。
     * 对低级函数的调用已经执行测试的地方也可以省略。
     *
     * Note that a NULL snapshot argument is allowed and causes a fast return
     * without error; this is to support call sites which can be called from
     * either scans or index modification areas.
     * 注意，snapshot为NULL是允许的，可快速返回，没有错误;
     * 这是为了支持可以从扫描或索引修改区域调用的调用位置。
     *
     * For best performance, keep the tests that are fastest and/or most likely to
     * exclude a page from old snapshot testing near the front.
     * 为了获得最好的性能，保持最快的测试和/或最可能从最前面的旧快照测试中排除一个页面。
     */
    static inline void
    TestForOldSnapshot(Snapshot snapshot, Relation relation, Page page)
    {
        Assert(relation != NULL);
    
        if (old_snapshot_threshold >= 0
            && (snapshot) != NULL
            && ((snapshot)->satisfies == HeapTupleSatisfiesMVCC
                || (snapshot)->satisfies == HeapTupleSatisfiesToast)
            && !XLogRecPtrIsInvalid((snapshot)->lsn)
            && PageGetLSN(page) > (snapshot)->lsn)
            TestForOldSnapshot_impl(snapshot, relation);
    }
    
    //--------------------------------------------------- TestForOldSnapshot_impl
    
    /*
     * Implement slower/larger portions of TestForOldSnapshot
     * 实现TestForOldSnapshot
     * 
     * Smaller/faster portions are put inline, but the entire set of logic is too
     * big for that.
     * 更小/更快的部分是内联的，但是整个实现逻辑显得过大了。
     */
    void
    TestForOldSnapshot_impl(Snapshot snapshot, Relation relation)
    {
        if (RelationAllowsEarlyPruning(relation)
            && (snapshot)->whenTaken < GetOldSnapshotThresholdTimestamp())
            ereport(ERROR,
                    (errcode(ERRCODE_SNAPSHOT_TOO_OLD),
                     errmsg("snapshot too old")));
    }
    
    
    //--------------------------------------------------- HeapKeyTest
    
    /*
     *      HeapKeyTest
     *
     *      Test a heap tuple to see if it satisfies a scan key.
     *       验证heap tuple是否满足扫描键要求
     */
    #define HeapKeyTest(tuple, \
                        tupdesc, \
                        nkeys, \
                        keys, \
                        result) \
    do \
    { \
        /* Use underscores to protect the variables passed in as parameters */ \
        /* 使用下划线来保护作为参数传入的变量*/ \
        int         __cur_nkeys = (nkeys); \
        ScanKey     __cur_keys = (keys); \
     \
        (result) = true; /* may change */ \
        for (; __cur_nkeys--; __cur_keys++) \
        { \
            Datum   __atp; \
            bool    __isnull; \
            Datum   __test; \
     \
            if (__cur_keys->sk_flags & SK_ISNULL) \
            { \
                (result) = false; \
                break; \
            } \
     \
            __atp = heap_getattr((tuple), \
                                 __cur_keys->sk_attno, \
                                 (tupdesc), \
                                 &__isnull); \
     \
            if (__isnull) \
            { \
                (result) = false; \
                break; \
            } \
     \
            __test = FunctionCall2Coll(&__cur_keys->sk_func, \
                                       __cur_keys->sk_collation, \
                                       __atp, __cur_keys->sk_argument); \
     \
            if (!DatumGetBool(__test)) \
            { \
                (result) = false; \
                break; \
            } \
        } \
    } while (0)
    
    

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
    
    

启动gdb,设置断点,进入heap_getnext

    
    
    (gdb) b heap_getnext
    Breakpoint 1 at 0x4de01f: file heapam.c, line 1841.
    (gdb) c
    Continuing.
    
    Breakpoint 1, heap_getnext (scan=0x2aadc18, direction=ForwardScanDirection) at heapam.c:1841
    1841        if (scan->rs_pageatatime)
    

查看输入参数,注意rs_pageatatime = true,使用page-at-a-time模式查询

    
    
    (gdb) p *scan
    $1 = {rs_rd = 0x7efdb8f2dfd8, rs_snapshot = 0x2a2a6d0, rs_nkeys = 0, rs_key = 0x0, rs_bitmapscan = false, 
      rs_samplescan = false, rs_pageatatime = true, rs_allow_strat = true, rs_allow_sync = true, rs_temp_snap = false, 
      rs_nblocks = 726, rs_startblock = 0, rs_numblocks = 4294967295, rs_strategy = 0x0, rs_syncscan = false, 
      rs_inited = false, rs_ctup = {t_len = 2139062143, t_self = {ip_blkid = {bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, 
        t_tableOid = 16742, t_data = 0x0}, rs_cblock = 4294967295, rs_cbuf = 0, rs_parallel = 0x0, rs_cindex = 2139062143, 
      rs_ntuples = 2139062143, rs_vistuples = {32639 <repeats 291 times>}}
    

进入heapgettup_pagemode函数

    
    
    (gdb) n
    1842            heapgettup_pagemode(scan, direction,
    (gdb) step
    heapgettup_pagemode (scan=0x2aadc18, dir=ForwardScanDirection, nkeys=0, key=0x0) at heapam.c:794
    794     HeapTuple   tuple = &(scan->rs_ctup);
    (gdb) 
    

heapgettup_pagemode->变量赋值,注意tuple还是一个"野"指针;尚未初始化p scan->rs_inited = false

    
    
    794     HeapTuple   tuple = &(scan->rs_ctup);
    (gdb) n
    795     bool        backward = ScanDirectionIsBackward(dir);
    (gdb) p *tuple
    $2 = {t_len = 2139062143, t_self = {ip_blkid = {bi_hi = 65535, bi_lo = 65535}, ip_posid = 0}, t_tableOid = 16742, 
      t_data = 0x0}
    (gdb) n
    808     if (ScanDirectionIsForward(dir))
    (gdb) p scan->rs_inited
    $3 = false
    

heapgettup_pagemode->非并行扫描,page = scan->rs_startblock(即page = 0)

    
    
    (gdb) n
    815             if (scan->rs_nblocks == 0 || scan->rs_numblocks == 0)
    (gdb) n
    821             if (scan->rs_parallel != NULL)
    (gdb) 
    836                 page = scan->rs_startblock; /* first page */
    (gdb) 
    

进入heapgetpage

    
    
    (gdb) n
    837             heapgetpage(scan, page);
    (gdb) step
    heapgetpage (scan=0x2aadc18, page=0) at heapam.c:362
    362     Assert(page < scan->rs_nblocks);
    

heapgetpage->检查验证&读取page

    
    
    362     Assert(page < scan->rs_nblocks);
    (gdb) n
    365     if (BufferIsValid(scan->rs_cbuf))
    (gdb) p scan->rs_cbuf
    $4 = 0
    (gdb) n
    376     CHECK_FOR_INTERRUPTS();
    (gdb) 
    379     scan->rs_cbuf = ReadBufferExtended(scan->rs_rd, MAIN_FORKNUM, page,
    (gdb) 
    381     scan->rs_cblock = page;
    (gdb) 
    383     if (!scan->rs_pageatatime)
    

heapgetpage->rs_cbuf为346/rs_cblock为0

    
    
    (gdb) p scan->rs_cbuf
    $5 = 346
    (gdb) p scan->rs_cblock
    $6 = 0
    

heapgetpage->page-at-a-time模式读取,变量赋值,锁定缓冲区

    
    
    (gdb) n
    386     buffer = scan->rs_cbuf;
    (gdb) n
    386     buffer = scan->rs_cbuf;
    (gdb) 
    387     snapshot = scan->rs_snapshot;
    (gdb) 
    392     heap_page_prune_opt(scan->rs_rd, buffer);
    (gdb) 
    399     LockBuffer(buffer, BUFFER_LOCK_SHARE);
    (gdb) p buffer
    $7 = 346
    

heapgetpage->获取page,检查快照是否过旧,获取行数

    
    
    (gdb) n
    401     dp = BufferGetPage(buffer);
    (gdb) 
    402     TestForOldSnapshot(snapshot, scan->rs_rd, dp);
    (gdb) p dp
    $8 = (Page) 0x7efda4b7ac00 "\001"
    (gdb) n
    403     lines = PageGetMaxOffsetNumber(dp);
    (gdb) 
    404     ntup = 0;
    (gdb) p lines
    $11 = 158
    

heapgetpage->验证可见性

    
    
    (gdb) n
    426     all_visible = PageIsAllVisible(dp) && !snapshot->takenDuringRecovery;
    (gdb) 
    428     for (lineoff = FirstOffsetNumber, lpp = PageGetItemId(dp, lineoff);
    (gdb) p all_visible
    $12 = false
    (gdb) n
    429          lineoff <= lines;
    (gdb) 
    428     for (lineoff = FirstOffsetNumber, lpp = PageGetItemId(dp, lineoff);
    (gdb) p lineoff
    $13 = 1
    (gdb) n
    432         if (ItemIdIsNormal(lpp))
    (gdb) 
    437             loctup.t_tableOid = RelationGetRelid(scan->rs_rd);
    (gdb) 
    438             loctup.t_data = (HeapTupleHeader) PageGetItem((Page) dp, lpp);
    (gdb) 
    439             loctup.t_len = ItemIdGetLength(lpp);
    (gdb) 
    440             ItemPointerSet(&(loctup.t_self), page, lineoff);
    (gdb) 
    442             if (all_visible)
    (gdb) 
    445                 valid = HeapTupleSatisfiesVisibility(&loctup, snapshot, buffer);
    (gdb) 
    447             CheckForSerializableConflictOut(valid, scan->rs_rd, &loctup,
    (gdb) 
    450             if (valid)
    (gdb) 
    451                 scan->rs_vistuples[ntup++] = lineoff;
    (gdb) 
    430          lineoff++, lpp++)
    (gdb) 
    ...
    

heapgettup_pagemode->退出heapgetpage,回到heapgettup_pagemode,初始化lineindex为0,设置rs_inited为T

    
    
    (gdb) finish
    Run till exit from #0  heapgetpage (scan=0x2aadc18, page=0) at heapam.c:430
    heapgettup_pagemode (scan=0x2aadc18, dir=ForwardScanDirection, nkeys=0, key=0x0) at heapam.c:838
    838             lineindex = 0;
    (gdb) n
    839             scan->rs_inited = true;
    (gdb) 
    848         dp = BufferGetPage(scan->rs_cbuf);
    

heapgettup_pagemode->获取page,验证快照是否过旧

    
    
    (gdb) n
    849         TestForOldSnapshot(scan->rs_snapshot, scan->rs_rd, dp);
    (gdb) p dp
    $18 = (Page) 0x7efda4b7ac00 "\001"
    

heapgettup_pagemode->计算Item数,开始循环

    
    
    (gdb) n
    850         lines = scan->rs_ntuples;
    (gdb) 
    853         linesleft = lines - lineindex;
    (gdb) 
    948         while (linesleft > 0)
    (gdb) p lines
    $19 = 158
    (gdb) p linesleft
    $20 = 158
    (gdb) 
    

heapgettup_pagemode->获取Item偏移(lineoff)和ItemId

    
    
    (gdb) p lineoff
    $21 = 1
    (gdb) p lpp
    $22 = (ItemId) 0x7efda4b7ac18
    (gdb) p *lpp
    $23 = {lp_off = 8152, lp_flags = 1, lp_len = 40}
    

heapgettup_pagemode->给tuple中的变量赋值,ItemPointer是ItemPointerData结构体指针

    
    
    (gdb) n
    954             tuple->t_data = (HeapTupleHeader) PageGetItem((Page) dp, lpp);
    (gdb) 
    955             tuple->t_len = ItemIdGetLength(lpp);
    (gdb) 
    956             ItemPointerSet(&(tuple->t_self), page, lineoff);
    (gdb) 
    961             if (key != NULL)
    (gdb) p *tuple->t_data
    $26 = {t_choice = {t_heap = {t_xmin = 862, t_xmax = 0, t_field3 = {t_cid = 0, t_xvac = 0}}, t_datum = {datum_len_ = 862, 
          datum_typmod = 0, datum_typeid = 0}}, t_ctid = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 1}, t_infomask2 = 5, 
      t_infomask = 2306, t_hoff = 24 '\030', t_bits = 0x7efda4b7cbef ""}
    (gdb) p tuple->t_len
    $28 = 40
    (gdb) p tuple->t_self
    $29 = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 1}
    

heapgettup_pagemode->设置scan->rs_cindex,返回

    
    
    (gdb) n
    975                 scan->rs_cindex = lineindex;
    (gdb) n
    976                 return;
    (gdb) p scan->rs_cindex 
    $30 = 0
    

回到heap_getnext

    
    
    (gdb) 
    heap_getnext (scan=0x2aadc18, direction=ForwardScanDirection) at heapam.c:1847
    1847        if (scan->rs_ctup.t_data == NULL)
    

返回获得的tuple

    
    
    1847        if (scan->rs_ctup.t_data == NULL)
    (gdb) n
    1859        pgstat_count_heap_getnext(scan->rs_rd);
    (gdb) 
    1861        return &(scan->rs_ctup);
    (gdb) p scan->rs_ctup
    $31 = {t_len = 40, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 1}, t_tableOid = 16742, t_data = 0x7efda4b7cbd8}
    

结束第一次调用,再次进入该函数

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 1, heap_getnext (scan=0x2aadc18, direction=ForwardScanDirection) at heapam.c:1841
    1841        if (scan->rs_pageatatime)
    (gdb) n
    1842            heapgettup_pagemode(scan, direction,
    (gdb) step
    heapgettup_pagemode (scan=0x2aadc18, dir=ForwardScanDirection, nkeys=0, key=0x0) at heapam.c:794
    794     HeapTuple   tuple = &(scan->rs_ctup);
    (gdb) n
    795     bool        backward = ScanDirectionIsBackward(dir);
    (gdb) 
    808     if (ScanDirectionIsForward(dir))
    (gdb) 
    810         if (!scan->rs_inited)
    (gdb) 
    844             page = scan->rs_cblock; /* current page */
    

查看输入参数scan,与上一次有所不同,存储了上一次调用返回的一些信息,如rs_vistuples等

    
    
    (gdb) p *scan
    $32 = {rs_rd = 0x7efdb8f2dfd8, rs_snapshot = 0x2a2a6d0, rs_nkeys = 0, rs_key = 0x0, rs_bitmapscan = false, 
      rs_samplescan = false, rs_pageatatime = true, rs_allow_strat = true, rs_allow_sync = true, rs_temp_snap = false, 
      rs_nblocks = 726, rs_startblock = 0, rs_numblocks = 4294967295, rs_strategy = 0x0, rs_syncscan = false, rs_inited = true, 
      rs_ctup = {t_len = 40, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 1}, t_tableOid = 16742, 
        t_data = 0x7efda4b7cbd8}, rs_cblock = 0, rs_cbuf = 346, rs_parallel = 0x0, rs_cindex = 0, rs_ntuples = 158, 
      rs_vistuples = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 
        29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 
        59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 
        89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100, 101, 102, 103, 104, 105, 106, 107, 108, 109, 110, 111, 112, 113, 114, 
        115, 116, 117, 118, 119, 120, 121, 122, 123, 124, 125, 126, 127, 128, 129, 130, 131, 132, 133, 134, 135, 136, 137, 138, 
        139, 140, 141, 142, 143, 144, 145, 146, 147, 148, 149, 150, 151, 152, 153, 154, 155, 156, 157, 158, 
        32639 <repeats 133 times>}}
    

DONE!

### 四、参考资料

[PostgreSQL Page页结构解析（1）-基础](https://www.jianshu.com/p/012643cfba25)  
[PostgreSQL Page页结构解析（2）- 页头和行数据指针](https://www.jianshu.com/p/3bc11bfa8fd1)  
[PostgreSQL Page页结构解析（3）- 行数据](https://www.jianshu.com/p/5ee448105555)

