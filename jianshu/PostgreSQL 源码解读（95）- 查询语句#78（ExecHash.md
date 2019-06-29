本节是ExecHashJoin函数介绍的第四部分，主要介绍了ExecHashJoin中依赖的其他函数的实现逻辑，这些函数在HJ_SCAN_BUCKET阶段中使用，主要的函数是ExecScanHashBucket。

### 一、数据结构

**JoinState**  
Hash/NestLoop/Merge Join的基类

    
    
    /* ----------------     *   JoinState information
     *
     *      Superclass for state nodes of join plans.
     *      Hash/NestLoop/Merge Join的基类
     * ----------------     */
    typedef struct JoinState
    {
        PlanState   ps;//基类PlanState
        JoinType    jointype;//连接类型
        //在找到一个匹配inner tuple的时候,如需要跳转到下一个outer tuple,则该值为T
        bool        single_match;   /* True if we should skip to next outer tuple
                                     * after finding one inner match */
        //连接条件表达式(除了ps.qual)
        ExprState  *joinqual;       /* JOIN quals (in addition to ps.qual) */
    } JoinState;
    
    

**HashJoinState**  
Hash Join运行期状态结构体

    
    
    /* these structs are defined in executor/hashjoin.h: */
    typedef struct HashJoinTupleData *HashJoinTuple;
    typedef struct HashJoinTableData *HashJoinTable;
    
    typedef struct HashJoinState
    {
        JoinState   js;             /* 基类;its first field is NodeTag */
        ExprState  *hashclauses;//hash连接条件
        List       *hj_OuterHashKeys;   /* 外表条件链表;list of ExprState nodes */
        List       *hj_InnerHashKeys;   /* 内表连接条件;list of ExprState nodes */
        List       *hj_HashOperators;   /* 操作符OIDs链表;list of operator OIDs */
        HashJoinTable hj_HashTable;//Hash表
        uint32      hj_CurHashValue;//当前的Hash值
        int         hj_CurBucketNo;//当前的bucket编号
        int         hj_CurSkewBucketNo;//行倾斜bucket编号
        HashJoinTuple hj_CurTuple;//当前元组
        TupleTableSlot *hj_OuterTupleSlot;//outer relation slot
        TupleTableSlot *hj_HashTupleSlot;//Hash tuple slot
        TupleTableSlot *hj_NullOuterTupleSlot;//用于外连接的outer虚拟slot
        TupleTableSlot *hj_NullInnerTupleSlot;//用于外连接的inner虚拟slot
        TupleTableSlot *hj_FirstOuterTupleSlot;//
        int         hj_JoinState;//JoinState状态
        bool        hj_MatchedOuter;//是否匹配
        bool        hj_OuterNotEmpty;//outer relation是否为空
    } HashJoinState;
    
    

**HashJoinTable**  
Hash表数据结构

    
    
    typedef struct HashJoinTableData
    {
        int         nbuckets;       /* 内存中的hash桶数;# buckets in the in-memory hash table */
        int         log2_nbuckets;  /* 2的对数(nbuckets必须是2的幂);its log2 (nbuckets must be a power of 2) */
    
        int         nbuckets_original;  /* 首次hash时的桶数;# buckets when starting the first hash */
        int         nbuckets_optimal;   /* 优化后的桶数(每个批次);optimal # buckets (per batch) */
        int         log2_nbuckets_optimal;  /* 2的对数;log2(nbuckets_optimal) */
    
        /* buckets[i] is head of list of tuples in i'th in-memory bucket */
        //bucket [i]是内存中第i个桶中的元组链表的head item
        union
        {
            /* unshared array is per-batch storage, as are all the tuples */
            //未共享数组是按批处理存储的，所有元组均如此
            struct HashJoinTupleData **unshared;
            /* shared array is per-query DSA area, as are all the tuples */
            //共享数组是每个查询的DSA区域，所有元组均如此
            dsa_pointer_atomic *shared;
        }           buckets;
    
        bool        keepNulls;      /*如不匹配则存储NULL元组,该值为T;true to store unmatchable NULL tuples */
    
        bool        skewEnabled;    /*是否使用倾斜优化?;are we using skew optimization? */
        HashSkewBucket **skewBucket;    /* 倾斜的hash表桶数;hashtable of skew buckets */
        int         skewBucketLen;  /* skewBucket数组大小;size of skewBucket array (a power of 2!) */
        int         nSkewBuckets;   /* 活动的倾斜桶数;number of active skew buckets */
        int        *skewBucketNums; /* 活动倾斜桶数组索引;array indexes of active skew buckets */
    
        int         nbatch;         /* 批次数;number of batches */
        int         curbatch;       /* 当前批次,第一轮为0;current batch #; 0 during 1st pass */
    
        int         nbatch_original;    /* 在开始inner扫描时的批次;nbatch when we started inner scan */
        int         nbatch_outstart;    /* 在开始outer扫描时的批次;nbatch when we started outer scan */
    
        bool        growEnabled;    /* 关闭nbatch增加的标记;flag to shut off nbatch increases */
    
        double      totalTuples;    /* 从inner plan获得的元组数;# tuples obtained from inner plan */
        double      partialTuples;  /* 通过hashjoin获得的inner元组数;# tuples obtained from inner plan by me */
        double      skewTuples;     /* 倾斜元组数;# tuples inserted into skew tuples */
    
        /*
         * These arrays are allocated for the life of the hash join, but only if
         * nbatch > 1.  A file is opened only when we first write a tuple into it
         * (otherwise its pointer remains NULL).  Note that the zero'th array
         * elements never get used, since we will process rather than dump out any
         * tuples of batch zero.
         * 这些数组在散列连接的生命周期内分配，但仅当nbatch > 1时分配。
         * 只有当第一次将元组写入文件时，文件才会打开(否则它的指针将保持NULL)。
         * 注意，第0个数组元素永远不会被使用，因为批次0的元组永远不会转储.
         */
        BufFile   **innerBatchFile; /* 每个批次的inner虚拟临时文件缓存;buffered virtual temp file per batch */
        BufFile   **outerBatchFile; /* 每个批次的outer虚拟临时文件缓存;buffered virtual temp file per batch */
    
        /*
         * Info about the datatype-specific hash functions for the datatypes being
         * hashed. These are arrays of the same length as the number of hash join
         * clauses (hash keys).
         * 有关正在散列的数据类型的特定于数据类型的散列函数的信息。
         * 这些数组的长度与散列连接子句(散列键)的数量相同。
         */
        FmgrInfo   *outer_hashfunctions;    /* outer hash函数FmgrInfo结构体;lookup data for hash functions */
        FmgrInfo   *inner_hashfunctions;    /* inner hash函数FmgrInfo结构体;lookup data for hash functions */
        bool       *hashStrict;     /* 每个hash操作符是严格?is each hash join operator strict? */
    
        Size        spaceUsed;      /* 元组使用的当前内存空间大小;memory space currently used by tuples */
        Size        spaceAllowed;   /* 空间使用上限;upper limit for space used */
        Size        spacePeak;      /* 峰值的空间使用;peak space used */
        Size        spaceUsedSkew;  /* 倾斜哈希表的当前空间使用情况;skew hash table's current space usage */
        Size        spaceAllowedSkew;   /* 倾斜哈希表的使用上限;upper limit for skew hashtable */
    
        MemoryContext hashCxt;      /* 整个散列连接存储的上下文;context for whole-hash-join storage */
        MemoryContext batchCxt;     /* 该批次存储的上下文;context for this-batch-only storage */
    
        /* used for dense allocation of tuples (into linked chunks) */
        //用于密集分配元组(到链接块中)
        HashMemoryChunk chunks;     /* 整个批次使用一个链表;one list for the whole batch */
    
        /* Shared and private state for Parallel Hash. */
        //并行hash使用的共享和私有状态
        HashMemoryChunk current_chunk;  /* 后台进程的当前chunk;this backend's current chunk */
        dsa_area   *area;           /* 用于分配内存的DSA区域;DSA area to allocate memory from */
        ParallelHashJoinState *parallel_state;//并行执行状态
        ParallelHashJoinBatchAccessor *batches;//并行访问器
        dsa_pointer current_chunk_shared;//当前chunk的开始指针
    } HashJoinTableData;
    
    typedef struct HashJoinTableData *HashJoinTable;
    
    

**HashJoinTupleData**  
Hash连接元组数据

    
    
    /* ----------------------------------------------------------------     *              hash-join hash table structures
     *
     * Each active hashjoin has a HashJoinTable control block, which is
     * palloc'd in the executor's per-query context.  All other storage needed
     * for the hashjoin is kept in private memory contexts, two for each hashjoin.
     * This makes it easy and fast to release the storage when we don't need it
     * anymore.  (Exception: data associated with the temp files lives in the
     * per-query context too, since we always call buffile.c in that context.)
     * 每个活动的hashjoin都有一个可散列的控制块，它在执行程序的每个查询上下文中都是通过palloc分配的。
     * hashjoin所需的所有其他存储都保存在私有内存上下文中，每个hashjoin有两个。
     * 当不再需要它的时候，这使得释放它变得简单和快速。
     * (例外:与临时文件相关的数据也存在于每个查询上下文中，因为在这种情况下总是调用buffile.c。)
     *
     * The hashtable contexts are made children of the per-query context, ensuring
     * that they will be discarded at end of statement even if the join is
     * aborted early by an error.  (Likewise, any temporary files we make will
     * be cleaned up by the virtual file manager in event of an error.)
     * hashtable上下文是每个查询上下文的子上下文，确保在语句结束时丢弃它们，即使连接因错误而提前中止。
     *   (同样，如果出现错误，虚拟文件管理器将清理创建的任何临时文件。)
     *
     * Storage that should live through the entire join is allocated from the
     * "hashCxt", while storage that is only wanted for the current batch is
     * allocated in the "batchCxt".  By resetting the batchCxt at the end of
     * each batch, we free all the per-batch storage reliably and without tedium.
     * 通过整个连接的存储空间应从“hashCxt”分配，而只需要当前批处理的存储空间在“batchCxt”中分配。
     * 通过在每个批处理结束时重置batchCxt，可以可靠地释放每个批处理的所有存储，而不会感到单调乏味。
     * 
     * During first scan of inner relation, we get its tuples from executor.
     * If nbatch > 1 then tuples that don't belong in first batch get saved
     * into inner-batch temp files. The same statements apply for the
     * first scan of the outer relation, except we write tuples to outer-batch
     * temp files.  After finishing the first scan, we do the following for
     * each remaining batch:
     *  1. Read tuples from inner batch file, load into hash buckets.
     *  2. Read tuples from outer batch file, match to hash buckets and output.
     * 在内部关系的第一次扫描中，从执行者那里得到了它的元组。
     * 如果nbatch > 1，那么不属于第一批的元组将保存到批内临时文件中。
     * 相同的语句适用于外关系的第一次扫描，但是我们将元组写入外部批处理临时文件。
     * 完成第一次扫描后，我们对每批剩余的元组做如下处理: 
     * 1.从内部批处理文件读取元组，加载到散列桶中。
     * 2.从外部批处理文件读取元组，匹配哈希桶和输出。 
     *
     * It is possible to increase nbatch on the fly if the in-memory hash table
     * gets too big.  The hash-value-to-batch computation is arranged so that this
     * can only cause a tuple to go into a later batch than previously thought,
     * never into an earlier batch.  When we increase nbatch, we rescan the hash
     * table and dump out any tuples that are now of a later batch to the correct
     * inner batch file.  Subsequently, while reading either inner or outer batch
     * files, we might find tuples that no longer belong to the current batch;
     * if so, we just dump them out to the correct batch file.
     * 如果内存中的哈希表太大，可以动态增加nbatch。
     * 散列值到批处理的计算是这样安排的:
     *   这只会导致元组进入比以前认为的更晚的批处理，而不会进入更早的批处理。
     * 当增加nbatch时，重新扫描哈希表，并将现在属于后面批处理的任何元组转储到正确的内部批处理文件。
     * 随后，在读取内部或外部批处理文件时，可能会发现不再属于当前批处理的元组;
     *   如果是这样，只需将它们转储到正确的批处理文件即可。
     * ----------------------------------------------------------------     */
    
    /* these are in nodes/execnodes.h: */
    /* typedef struct HashJoinTupleData *HashJoinTuple; */
    /* typedef struct HashJoinTableData *HashJoinTable; */
    
    typedef struct HashJoinTupleData
    {
        /* link to next tuple in same bucket */
        //link同一个桶中的下一个元组
        union
        {
            struct HashJoinTupleData *unshared;
            dsa_pointer shared;
        }           next;
        uint32      hashvalue;      /* 元组的hash值;tuple's hash code */
        /* Tuple data, in MinimalTuple format, follows on a MAXALIGN boundary */
    }           HashJoinTupleData;
    
    #define HJTUPLE_OVERHEAD  MAXALIGN(sizeof(HashJoinTupleData))
    #define HJTUPLE_MINTUPLE(hjtup)  \
        ((MinimalTuple) ((char *) (hjtup) + HJTUPLE_OVERHEAD))
    
    

### 二、源码解读

**ExecScanHashBucket**  
搜索匹配当前outer relation tuple的hash桶,寻找匹配的inner relation元组。

    
    
    /*----------------------------------------------------------------------------------------------------                                        HJ_SCAN_BUCKET 阶段
    ----------------------------------------------------------------------------------------------------*/
    
    /*
     * ExecScanHashBucket
     *      scan a hash bucket for matches to the current outer tuple
     *      搜索匹配当前outer relation tuple的hash桶
     * 
     * The current outer tuple must be stored in econtext->ecxt_outertuple.
     * 当前的outer relation tuple必须存储在econtext->ecxt_outertuple中
     * 
     * On success, the inner tuple is stored into hjstate->hj_CurTuple and
     * econtext->ecxt_innertuple, using hjstate->hj_HashTupleSlot as the slot
     * for the latter.
     * 成功后，内部元组存储到hjstate->hj_CurTuple和econtext->ecxt_innertuple中，
     *   使用hjstate->hj_HashTupleSlot作为后者的slot。
     */
    bool
    ExecScanHashBucket(HashJoinState *hjstate,
                       ExprContext *econtext)
    {
        ExprState  *hjclauses = hjstate->hashclauses;//hash连接条件表达式
        HashJoinTable hashtable = hjstate->hj_HashTable;//Hash表
        HashJoinTuple hashTuple = hjstate->hj_CurTuple;//当前的Tuple
        uint32      hashvalue = hjstate->hj_CurHashValue;//hash值
    
        /*
         * hj_CurTuple is the address of the tuple last returned from the current
         * bucket, or NULL if it's time to start scanning a new bucket.
         * hj_CurTuple是最近从当前桶返回的元组的地址，如果需要开始扫描新桶，则为NULL。
         *
         * If the tuple hashed to a skew bucket then scan the skew bucket
         * otherwise scan the standard hashtable bucket.
         * 如果元组散列到倾斜桶，则扫描倾斜桶，否则扫描标准哈希表桶。
         */
        if (hashTuple != NULL)
            hashTuple = hashTuple->next.unshared;//hashTuple,通过指针获取下一个
        else if (hjstate->hj_CurSkewBucketNo != INVALID_SKEW_BUCKET_NO)
            //如为NULL,而且使用倾斜优化,则从倾斜桶中获取
            hashTuple = hashtable->skewBucket[hjstate->hj_CurSkewBucketNo]->tuples;
        else
            ////如为NULL,不使用倾斜优化,从常规的bucket中获取
            hashTuple = hashtable->buckets.unshared[hjstate->hj_CurBucketNo];
    
        while (hashTuple != NULL)//循环
        {
            if (hashTuple->hashvalue == hashvalue)//hash值一致
            {
                TupleTableSlot *inntuple;//inner tuple
    
                /* insert hashtable's tuple into exec slot so ExecQual sees it */
                //把Hash表中的tuple插入到执行器的slot中,作为函数ExecQual的输入使用
                inntuple = ExecStoreMinimalTuple(HJTUPLE_MINTUPLE(hashTuple),
                                                 hjstate->hj_HashTupleSlot,
                                                 false);    /* do not pfree */
                econtext->ecxt_innertuple = inntuple;//赋值
    
                if (ExecQualAndReset(hjclauses, econtext))//判断连接条件是否满足
                {
                    hjstate->hj_CurTuple = hashTuple;//满足,则赋值&返回T
                    return true;
                }
            }
    
            hashTuple = hashTuple->next.unshared;//从Hash表中获取下一个tuple
        }
    
        /*
         * no match
         * 不匹配,返回F
         */
        return false;
    }
    
    
    /*
     * Store a minimal tuple into TTSOpsMinimalTuple type slot.
     * 存储最小化的tuple到TTSOpsMinimalTuple类型的slot中
     *
     * If the target slot is not guaranteed to be TTSOpsMinimalTuple type slot,
     * use the, more expensive, ExecForceStoreMinimalTuple().
     * 如果目标slot不能确保是TTSOpsMinimalTuple类型,使用代价更高的ExecForceStoreMinimalTuple()函数
     */
    TupleTableSlot *
    ExecStoreMinimalTuple(MinimalTuple mtup,
                          TupleTableSlot *slot,
                          bool shouldFree)
    {
        /*
         * sanity checks
         * 安全检查
         */
        Assert(mtup != NULL);
        Assert(slot != NULL);
        Assert(slot->tts_tupleDescriptor != NULL);
    
        if (unlikely(!TTS_IS_MINIMALTUPLE(slot)))//类型检查
            elog(ERROR, "trying to store a minimal tuple into wrong type of slot");
        tts_minimal_store_tuple(slot, mtup, shouldFree);//存储
    
        return slot;//返回slot
    }
    
    
    static void
    tts_minimal_store_tuple(TupleTableSlot *slot, MinimalTuple mtup, bool shouldFree)
    {
        MinimalTupleTableSlot *mslot = (MinimalTupleTableSlot *) slot;//获取slot
    
        tts_minimal_clear(slot);//清除原来的slot
        //安全检查
        Assert(!TTS_SHOULDFREE(slot));
        Assert(TTS_EMPTY(slot));
        //设置slot信息
        slot->tts_flags &= ~TTS_FLAG_EMPTY;
        slot->tts_nvalid = 0;
        mslot->off = 0;
        //存储到mslot中
        mslot->mintuple = mtup;
        Assert(mslot->tuple == &mslot->minhdr);
        mslot->minhdr.t_len = mtup->t_len + MINIMAL_TUPLE_OFFSET;
        mslot->minhdr.t_data = (HeapTupleHeader) ((char *) mtup - MINIMAL_TUPLE_OFFSET);
        /* no need to set t_self or t_tableOid since we won't allow access */
        //不需要设置t_sefl或者t_tableOid,因为不允许访问
        if (shouldFree)
            slot->tts_flags |= TTS_FLAG_SHOULDFREE;
        else
            Assert(!TTS_SHOULDFREE(slot));
    }
     
    
    /*
     * ExecQualAndReset() - evaluate qual with ExecQual() and reset expression
     * context.
     * ExecQualAndReset() - 使用ExecQual()解析并重置表达式
     */
    #ifndef FRONTEND
    static inline bool
    ExecQualAndReset(ExprState *state, ExprContext *econtext)
    {
        bool        ret = ExecQual(state, econtext);//调用ExecQual
    
        /* inline ResetExprContext, to avoid ordering issue in this file */
        //内联ResetExprContext,避免在这个文件中的ordering
        MemoryContextReset(econtext->ecxt_per_tuple_memory);
        return ret;
    }
    #endif
    
    
    #define HeapTupleHeaderSetMatch(tup) \
    ( \
      (tup)->t_infomask2 |= HEAP_TUPLE_HAS_MATCH \
    )
    
    

### 三、跟踪分析

测试脚本如下

    
    
    testdb=# set enable_nestloop=false;
    SET
    testdb=# set enable_mergejoin=false;
    SET
    testdb=# explain verbose select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je 
    testdb-# from t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je 
    testdb(#                         from t_grxx gr inner join t_jfxx jf 
    testdb(#                                        on gr.dwbh = dw.dwbh 
    testdb(#                                           and gr.grbh = jf.grbh) grjf
    testdb-# order by dw.dwbh;
                                              QUERY PLAN                                           
    -----------------------------------------------------------------------------------------------     Sort  (cost=14828.83..15078.46 rows=99850 width=47)
       Output: dw.dwmc, dw.dwbh, dw.dwdz, gr.grbh, gr.xm, jf.ny, jf.je
       Sort Key: dw.dwbh
       ->  Hash Join  (cost=3176.00..6537.55 rows=99850 width=47)
             Output: dw.dwmc, dw.dwbh, dw.dwdz, gr.grbh, gr.xm, jf.ny, jf.je
             Hash Cond: ((gr.grbh)::text = (jf.grbh)::text)
             ->  Hash Join  (cost=289.00..2277.61 rows=99850 width=32)
                   Output: dw.dwmc, dw.dwbh, dw.dwdz, gr.grbh, gr.xm
                   Inner Unique: true
                   Hash Cond: ((gr.dwbh)::text = (dw.dwbh)::text)
                   ->  Seq Scan on public.t_grxx gr  (cost=0.00..1726.00 rows=100000 width=16)
                         Output: gr.dwbh, gr.grbh, gr.xm, gr.xb, gr.nl
                   ->  Hash  (cost=164.00..164.00 rows=10000 width=20)
                         Output: dw.dwmc, dw.dwbh, dw.dwdz
                         ->  Seq Scan on public.t_dwxx dw  (cost=0.00..164.00 rows=10000 width=20)
                               Output: dw.dwmc, dw.dwbh, dw.dwdz
             ->  Hash  (cost=1637.00..1637.00 rows=100000 width=20)
                   Output: jf.ny, jf.je, jf.grbh
                   ->  Seq Scan on public.t_jfxx jf  (cost=0.00..1637.00 rows=100000 width=20)
                         Output: jf.ny, jf.je, jf.grbh
    (20 rows)
    

启动gdb,设置断点

    
    
    (gdb) b ExecScanHashBucket
    Breakpoint 1 at 0x6ff25b: file nodeHash.c, line 1910.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecScanHashBucket (hjstate=0x2bb8738, econtext=0x2bb8950) at nodeHash.c:1910
    1910        ExprState  *hjclauses = hjstate->hashclauses;
    

设置相关变量

    
    
    1910        ExprState  *hjclauses = hjstate->hashclauses;
    (gdb) n
    1911        HashJoinTable hashtable = hjstate->hj_HashTable;
    (gdb) 
    1912        HashJoinTuple hashTuple = hjstate->hj_CurTuple;
    (gdb) 
    1913        uint32      hashvalue = hjstate->hj_CurHashValue;
    (gdb) 
    1922        if (hashTuple != NULL)
    

hash join连接条件

    
    
    (gdb) p *hjclauses
    $1 = {tag = {type = T_ExprState}, flags = 7 '\a', resnull = false, resvalue = 0, resultslot = 0x0, steps = 0x2bc4bc8, 
      evalfunc = 0x6d1a6e <ExecInterpExprStillValid>, expr = 0x2bb60c0, evalfunc_private = 0x6cf625 <ExecInterpExpr>, 
      steps_len = 7, steps_alloc = 16, parent = 0x2bb8738, ext_params = 0x0, innermost_caseval = 0x0, innermost_casenull = 0x0, 
      innermost_domainval = 0x0, innermost_domainnull = 0x0}
    

hash表

    
    
    (gdb) p hashtable
    $2 = (HashJoinTable) 0x2bc9de8
    (gdb) p *hashtable
    $3 = {nbuckets = 16384, log2_nbuckets = 14, nbuckets_original = 16384, nbuckets_optimal = 16384, 
      log2_nbuckets_optimal = 14, buckets = {unshared = 0x7f0fc1345050, shared = 0x7f0fc1345050}, keepNulls = false, 
      skewEnabled = false, skewBucket = 0x0, skewBucketLen = 0, nSkewBuckets = 0, skewBucketNums = 0x0, nbatch = 1, 
      curbatch = 0, nbatch_original = 1, nbatch_outstart = 1, growEnabled = true, totalTuples = 10000, partialTuples = 10000, 
      skewTuples = 0, innerBatchFile = 0x0, outerBatchFile = 0x0, outer_hashfunctions = 0x2bdc228, 
      inner_hashfunctions = 0x2bdc280, hashStrict = 0x2bdc2d8, spaceUsed = 677754, spaceAllowed = 16777216, spacePeak = 677754, 
      spaceUsedSkew = 0, spaceAllowedSkew = 335544, hashCxt = 0x2bdc110, batchCxt = 0x2bde120, chunks = 0x2c708f0, 
      current_chunk = 0x0, area = 0x0, parallel_state = 0x0, batches = 0x0, current_chunk_shared = 0}
    

hash桶中的元组&hash值

    
    
    (gdb) p *hashTuple
    Cannot access memory at address 0x0
    (gdb) p hashvalue
    $4 = 2324234220
    (gdb) 
    

从常规hash桶中获取hash元组

    
    
    (gdb) n
    1924        else if (hjstate->hj_CurSkewBucketNo != INVALID_SKEW_BUCKET_NO)
    (gdb) p hjstate->hj_CurSkewBucketNo
    $5 = -1
    (gdb) n
    1927            hashTuple = hashtable->buckets.unshared[hjstate->hj_CurBucketNo];
    (gdb) 
    1929        while (hashTuple != NULL)
    (gdb) p hjstate->hj_CurBucketNo
    $7 = 16364
    (gdb) p *hashTuple
    $6 = {next = {unshared = 0x0, shared = 0}, hashvalue = 1822113772}
    

判断hash值是否一致

    
    
    (gdb) n
    1931            if (hashTuple->hashvalue == hashvalue)
    (gdb) p hashTuple->hashvalue
    $8 = 1822113772
    (gdb) p hashvalue
    $9 = 2324234220
    (gdb) 
    

不一致,继续下一个元组

    
    
    (gdb) n
    1948            hashTuple = hashTuple->next.unshared;
    (gdb) 
    1929        while (hashTuple != NULL)
    

下一个元组为NULL,返回F,说明没有匹配的元组

    
    
    (gdb) p *hashTuple
    Cannot access memory at address 0x0
    (gdb) n
    1954        return false;
    

在ExecStoreMinimalTuple上设置断点(这时候Hash值是一致的)

    
    
    (gdb) b ExecStoreMinimalTuple
    Breakpoint 2 at 0x6e8cbf: file execTuples.c, line 427.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecScanHashBucket (hjstate=0x2bb8738, econtext=0x2bb8950) at nodeHash.c:1910
    1910        ExprState  *hjclauses = hjstate->hashclauses;
    (gdb) del 1
    (gdb) c
    Continuing.
    
    Breakpoint 2, ExecStoreMinimalTuple (mtup=0x2be81b0, slot=0x2bb9c18, shouldFree=false) at execTuples.c:427
    427     Assert(mtup != NULL);
    (gdb) finish
    Run till exit from #0  ExecStoreMinimalTuple (mtup=0x2be81b0, slot=0x2bb9c18, shouldFree=false) at execTuples.c:427
    0x00000000006ff335 in ExecScanHashBucket (hjstate=0x2bb8738, econtext=0x2bb8950) at nodeHash.c:1936
    1936                inntuple = ExecStoreMinimalTuple(HJTUPLE_MINTUPLE(hashTuple),
    Value returned is $10 = (TupleTableSlot *) 0x2bb9c18
    (gdb) n
    1939                econtext->ecxt_innertuple = inntuple;
    

匹配成功,返回T

    
    
    (gdb) n
    1941                if (ExecQualAndReset(hjclauses, econtext))
    (gdb) 
    1943                    hjstate->hj_CurTuple = hashTuple;
    (gdb) 
    1944                    return true;
    (gdb) 
    1955    }
    (gdb) 
    

DONE!

**HJ_SCAN_BUCKET阶段,实现的逻辑是扫描Hash桶,寻找inner relation中与outer
relation元组匹配的元组,如匹配,则把匹配的Tuple存储在hjstate- >hj_CurTuple中.**

### 四、参考资料

[Hash Joins: Past, Present and Future/PGCon
2017](https://www.pgcon.org/2017/schedule/events/1053.en.html)  
[A Look at How Postgres Executes a Tiny Join - Part
1](https://dzone.com/articles/a-look-at-how-postgres-executes-a-tiny-join-part-1)  
[A Look at How Postgres Executes a Tiny Join - Part
2](https://dzone.com/articles/a-look-at-how-postgres-executes-a-tiny-join-part-2)  
[Assignment 2 Symmetric Hash
Join](https://cs.uwaterloo.ca/~david/cs448/A2.pdf)

