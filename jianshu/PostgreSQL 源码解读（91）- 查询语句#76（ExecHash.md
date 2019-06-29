本节是ExecHashJoin函数介绍的第二部分，主要介绍了ExecHashJoin中依赖的其他函数的实现逻辑，包括ExecHashTableCreate、ExecChooseHashTableSize等。

### 一、数据结构

**Plan**  
所有计划节点通过将Plan结构作为第一个字段从Plan结构“派生”。这确保了在将节点转换为计划节点时，一切都能正常工作。(在执行器中以通用方式传递时，节点指针经常被转换为Plan
*)

    
    
    /* ----------------     *      Plan node
     *
     * All plan nodes "derive" from the Plan structure by having the
     * Plan structure as the first field.  This ensures that everything works
     * when nodes are cast to Plan's.  (node pointers are frequently cast to Plan*
     * when passed around generically in the executor)
     * 所有计划节点通过将Plan结构作为第一个字段从Plan结构“派生”。
     * 这确保了在将节点转换为计划节点时，一切都能正常工作。
     * (在执行器中以通用方式传递时，节点指针经常被转换为Plan *)
     *
     * We never actually instantiate any Plan nodes; this is just the common
     * abstract superclass for all Plan-type nodes.
     * 从未实例化任何Plan节点;这只是所有Plan-type节点的通用抽象超类。
     * ----------------     */
    typedef struct Plan
    {
        NodeTag     type;//节点类型
    
        /*
         * 成本估算信息;estimated execution costs for plan (see costsize.c for more info)
         */
        Cost        startup_cost;   /* 启动成本;cost expended before fetching any tuples */
        Cost        total_cost;     /* 总成本;total cost (assuming all tuples fetched) */
    
        /*
         * 优化器估算信息;planner's estimate of result size of this plan step
         */
        double      plan_rows;      /* 行数;number of rows plan is expected to emit */
        int         plan_width;     /* 平均行大小(Byte为单位);average row width in bytes */
    
        /*
         * 并行执行相关的信息;information needed for parallel query
         */
        bool        parallel_aware; /* 是否参与并行执行逻辑?engage parallel-aware logic? */
        bool        parallel_safe;  /* 是否并行安全;OK to use as part of parallel plan? */
    
        /*
         * Plan类型节点通用的信息.Common structural data for all Plan types.
         */
        int         plan_node_id;   /* unique across entire final plan tree */
        List       *targetlist;     /* target list to be computed at this node */
        List       *qual;           /* implicitly-ANDed qual conditions */
        struct Plan *lefttree;      /* input plan tree(s) */
        struct Plan *righttree;
        List       *initPlan;       /* Init Plan nodes (un-correlated expr
                                     * subselects) */
    
        /*
         * Information for management of parameter-change-driven rescanning
         * parameter-change-driven重扫描的管理信息.
         * 
         * extParam includes the paramIDs of all external PARAM_EXEC params
         * affecting this plan node or its children.  setParam params from the
         * node's initPlans are not included, but their extParams are.
         *
         * allParam includes all the extParam paramIDs, plus the IDs of local
         * params that affect the node (i.e., the setParams of its initplans).
         * These are _all_ the PARAM_EXEC params that affect this node.
         */
        Bitmapset  *extParam;
        Bitmapset  *allParam;
    } Plan;
    
    

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

**ExecHashTableCreate**  
ExecHashTableCreate函数初始化hashjoin需要使用的hashtable.

    
    
    /*----------------------------------------------------------------------------------------------------                                        HJ_BUILD_HASHTABLE 阶段
    -----------------------------------------------------------------------------------------------------*/
    
    /* ----------------     *  these are defined to avoid confusion problems with "left"
     *  and "right" and "inner" and "outer".  The convention is that
     *  the "left" plan is the "outer" plan and the "right" plan is
     *  the inner plan, but these make the code more readable.
     *  这些定义是为了避免“左”和“右”以及“内”和“外”的混淆问题。
     *  约定是，“左”计划是“外部”计划，“右”计划是内部计划，但是这些计划使代码更具可读性。
     * ----------------     */
    #define innerPlan(node)         (((Plan *)(node))->righttree)
    #define outerPlan(node)         (((Plan *)(node))->lefttree)
    
    /* ----------------------------------------------------------------     *      ExecHashTableCreate
     *
     *      create an empty hashtable data structure for hashjoin.
     *      初始化hashjoin需要使用的hashtable.
     * ----------------------------------------------------------------     */
    HashJoinTable
    ExecHashTableCreate(HashState *state, List *hashOperators, bool keepNulls)
    {
        Hash       *node;
        HashJoinTable hashtable;
        Plan       *outerNode;
        size_t      space_allowed;
        int         nbuckets;
        int         nbatch;
        double      rows;
        int         num_skew_mcvs;
        int         log2_nbuckets;
        int         nkeys;
        int         i;
        ListCell   *ho;
        MemoryContext oldcxt;
    
        /*
         * Get information about the size of the relation to be hashed (it's the
         * "outer" subtree of this node, but the inner relation of the hashjoin).
         * Compute the appropriate size of the hash table.
         * 获取有关要散列的关系大小的信息(它是该节点的“outer”子树，hashjoin的inner relation)。
         * 计算哈希表的适当大小。
         */
        node = (Hash *) state->ps.plan;//获取Hash节点
        outerNode = outerPlan(node);//获取outer relation Plan节点
    
        /*
         * If this is shared hash table with a partial plan, then we can't use
         * outerNode->plan_rows to estimate its size.  We need an estimate of the
         * total number of rows across all copies of the partial plan.
         * 如果这是带有部分计划(并行处理)的共享哈希表，那么不能使用outerNode->plan_rows来估计它的大小。
         * 需要估算跨部分计划的所有副本的行总数。
         */
        rows = node->plan.parallel_aware ? node->rows_total : outerNode->plan_rows;//获取总行数
    
        ExecChooseHashTableSize(rows, outerNode->plan_width,
                                OidIsValid(node->skewTable),
                                state->parallel_state != NULL,
                                state->parallel_state != NULL ?
                                state->parallel_state->nparticipants - 1 : 0,
                                &space_allowed,
                                &nbuckets, &nbatch, &num_skew_mcvs);//计算Hash Table的大小尺寸
    
        /* nbuckets must be a power of 2 */
        //nbuckets(hash桶数)必须是2的n次方
        log2_nbuckets = my_log2(nbuckets);
        Assert(nbuckets == (1 << log2_nbuckets));
    
        /*
         * Initialize the hash table control block.
         * 初始化hash表的控制块
         *
         * The hashtable control block is just palloc'd from the executor's
         * per-query memory context.  Everything else should be kept inside the
         * subsidiary hashCxt or batchCxt.
         * hashtable控件块是从执行程序的每个查询内存上下文中调取的。
         * 其他内容都应该保存在附属hashCxt或batchCxt中。
         */
        hashtable = (HashJoinTable) palloc(sizeof(HashJoinTableData));//分配内存
        hashtable->nbuckets = nbuckets;//桶数
        hashtable->nbuckets_original = nbuckets;
        hashtable->nbuckets_optimal = nbuckets;
        hashtable->log2_nbuckets = log2_nbuckets;
        hashtable->log2_nbuckets_optimal = log2_nbuckets;
        hashtable->buckets.unshared = NULL;
        hashtable->keepNulls = keepNulls;
        hashtable->skewEnabled = false;
        hashtable->skewBucket = NULL;
        hashtable->skewBucketLen = 0;
        hashtable->nSkewBuckets = 0;
        hashtable->skewBucketNums = NULL;
        hashtable->nbatch = nbatch;
        hashtable->curbatch = 0;
        hashtable->nbatch_original = nbatch;
        hashtable->nbatch_outstart = nbatch;
        hashtable->growEnabled = true;
        hashtable->totalTuples = 0;
        hashtable->partialTuples = 0;
        hashtable->skewTuples = 0;
        hashtable->innerBatchFile = NULL;
        hashtable->outerBatchFile = NULL;
        hashtable->spaceUsed = 0;
        hashtable->spacePeak = 0;
        hashtable->spaceAllowed = space_allowed;
        hashtable->spaceUsedSkew = 0;
        hashtable->spaceAllowedSkew =
            hashtable->spaceAllowed * SKEW_WORK_MEM_PERCENT / 100;
        hashtable->chunks = NULL;
        hashtable->current_chunk = NULL;
        hashtable->parallel_state = state->parallel_state;
        hashtable->area = state->ps.state->es_query_dsa;
        hashtable->batches = NULL;
    
    #ifdef HJDEBUG
        printf("Hashjoin %p: initial nbatch = %d, nbuckets = %d\n",
               hashtable, nbatch, nbuckets);
    #endif
    
        /*
         * Create temporary memory contexts in which to keep the hashtable working
         * storage.  See notes in executor/hashjoin.h.
         * 创建临时内存上下文，以便在其中保持散列表的相关信息。
         * 参见executor/hashjoin.h中的注释。
         */
        hashtable->hashCxt = AllocSetContextCreate(CurrentMemoryContext,
                                                   "HashTableContext",
                                                   ALLOCSET_DEFAULT_SIZES);
    
        hashtable->batchCxt = AllocSetContextCreate(hashtable->hashCxt,
                                                    "HashBatchContext",
                                                    ALLOCSET_DEFAULT_SIZES);
    
        /* Allocate data that will live for the life of the hashjoin */
        //分配内存,切换至hashCxt
        oldcxt = MemoryContextSwitchTo(hashtable->hashCxt);
    
        /*
         * Get info about the hash functions to be used for each hash key. Also
         * remember whether the join operators are strict.
         * 获取关于每个散列键要使用的散列函数的信息。
         * 还要记住连接操作符是否严格。
         */
        nkeys = list_length(hashOperators);//键值数
        hashtable->outer_hashfunctions =
            (FmgrInfo *) palloc(nkeys * sizeof(FmgrInfo));//outer relation所使用的hash函数
        hashtable->inner_hashfunctions =
            (FmgrInfo *) palloc(nkeys * sizeof(FmgrInfo));//inner relation所使用的hash函数
        hashtable->hashStrict = (bool *) palloc(nkeys * sizeof(bool));//是否严格的操作符
        i = 0;
        foreach(ho, hashOperators)//遍历hash操作符
        {
            Oid         hashop = lfirst_oid(ho);//hash操作符
            Oid         left_hashfn;//左函数
            Oid         right_hashfn;//右函数
            //获取与给定操作符兼容的标准哈希函数的OID，并根据需要对其LHS和/或RHS数据类型进行操作。
            if (!get_op_hash_functions(hashop, &left_hashfn, &right_hashfn))//获取hash函数
                elog(ERROR, "could not find hash function for hash operator %u",
                     hashop);
            fmgr_info(left_hashfn, &hashtable->outer_hashfunctions[i]);
            fmgr_info(right_hashfn, &hashtable->inner_hashfunctions[i]);
            hashtable->hashStrict[i] = op_strict(hashop);
            i++;
        }
    
        if (nbatch > 1 && hashtable->parallel_state == NULL)//批次>1而且并行状态为NULL
        {
            /*
             * allocate and initialize the file arrays in hashCxt (not needed for
             * parallel case which uses shared tuplestores instead of raw files)
             * 在hashCxt中分配和初始化文件数组(对于使用共享tuplestore而不是原始文件的并行情况不需要)
             */
            hashtable->innerBatchFile = (BufFile **)
                palloc0(nbatch * sizeof(BufFile *));//用于缓存该批次的inner relation的tuple
            hashtable->outerBatchFile = (BufFile **)
                palloc0(nbatch * sizeof(BufFile *));//用于缓存该批次的outerr relation的tuple
            /* The files will not be opened until needed... */
            /* ... but make sure we have temp tablespaces established for them */
            //这些文件需要时才会打开……
            //…但是要确保为它们建立了临时表空间
            PrepareTempTablespaces();
        }
    
        MemoryContextSwitchTo(oldcxt);//切换回原内存上下文
    
        if (hashtable->parallel_state)//并行处理
        {
            ParallelHashJoinState *pstate = hashtable->parallel_state;
            Barrier    *build_barrier;
    
            /*
             * Attach to the build barrier.  The corresponding detach operation is
             * in ExecHashTableDetach.  Note that we won't attach to the
             * batch_barrier for batch 0 yet.  We'll attach later and start it out
             * in PHJ_BATCH_PROBING phase, because batch 0 is allocated up front
             * and then loaded while hashing (the standard hybrid hash join
             * algorithm), and we'll coordinate that using build_barrier.
             */
            build_barrier = &pstate->build_barrier;
            BarrierAttach(build_barrier);
    
            /*
             * So far we have no idea whether there are any other participants,
             * and if so, what phase they are working on.  The only thing we care
             * about at this point is whether someone has already created the
             * SharedHashJoinBatch objects and the hash table for batch 0.  One
             * backend will be elected to do that now if necessary.
             */
            if (BarrierPhase(build_barrier) == PHJ_BUILD_ELECTING &&
                BarrierArriveAndWait(build_barrier, WAIT_EVENT_HASH_BUILD_ELECTING))
            {
                pstate->nbatch = nbatch;
                pstate->space_allowed = space_allowed;
                pstate->growth = PHJ_GROWTH_OK;
    
                /* Set up the shared state for coordinating batches. */
                ExecParallelHashJoinSetUpBatches(hashtable, nbatch);
    
                /*
                 * Allocate batch 0's hash table up front so we can load it
                 * directly while hashing.
                 */
                pstate->nbuckets = nbuckets;
                ExecParallelHashTableAlloc(hashtable, 0);
            }
    
            /*
             * The next Parallel Hash synchronization point is in
             * MultiExecParallelHash(), which will progress it all the way to
             * PHJ_BUILD_DONE.  The caller must not return control from this
             * executor node between now and then.
             */
        }
        else//非并行处理
        {
            /*
             * Prepare context for the first-scan space allocations; allocate the
             * hashbucket array therein, and set each bucket "empty".
             * 为第一次扫描空间分配准备上下文;在其中分配hashbucket数组，并将每个bucket设置为“空”。
             */
            MemoryContextSwitchTo(hashtable->batchCxt);//切换上下文
    
            hashtable->buckets.unshared = (HashJoinTuple *)
                palloc0(nbuckets * sizeof(HashJoinTuple));//分配内存空间
    
            /*
             * Set up for skew optimization, if possible and there's a need for
             * more than one batch.  (In a one-batch join, there's no point in
             * it.)
             * 如需要多个批处理,设置倾斜优化。(在单批处理连接中，这是没有意义的。)
             */
            if (nbatch > 1)
                ExecHashBuildSkewHash(hashtable, node, num_skew_mcvs);
    
            MemoryContextSwitchTo(oldcxt);//切换上下文
        }
    
        return hashtable;//返回Hash表
    }
    
     /*
      * This routine fills a FmgrInfo struct, given the OID
      * of the function to be called.
      * 给定要调用的函数的OID，这个例程填充一个FmgrInfo结构体。
      *
      * The caller's CurrentMemoryContext is used as the fn_mcxt of the info
      * struct; this means that any subsidiary data attached to the info struct
      * (either by fmgr_info itself, or later on by a function call handler)
      * will be allocated in that context.  The caller must ensure that this
      * context is at least as long-lived as the info struct itself.  This is
      * not a problem in typical cases where the info struct is on the stack or
      * in freshly-palloc'd space.  However, if one intends to store an info
      * struct in a long-lived table, it's better to use fmgr_info_cxt.
      * 调用方的CurrentMemoryContext用作info结构体的fn_mcxt;
      * 这意味着附加到info结构体的任何附属数据(通过fmgr_info本身，或者稍后通过函数调用处理程序)将在该上下文中分配。
      * 调用者必须确保这个上下文的生命周期至少与info结构本身一样。
      * 在信息结构位于堆栈上或在新palloc空间中的典型情况下，这不是一个问题。
      * 但是，如果希望在long-lived表中存储信息结构，最好使用fmgr_info_cxt。
      */
     void
     fmgr_info(Oid functionId, FmgrInfo *finfo)
     {
         fmgr_info_cxt_security(functionId, finfo, CurrentMemoryContext, false);
     }
     
    
    

**ExecChooseHashTableSize**  
ExecChooseHashTableSize函数根据给定要散列的关系的估计大小(行数和平均行宽)，计算适当的散列表大小。

    
    
    /*
     * Compute appropriate size for hashtable given the estimated size of the
     * relation to be hashed (number of rows and average row width).
     * 给定要散列的关系的估计大小(行数和平均行宽)，计算适当的散列表大小。
     *
     * This is exported so that the planner's costsize.c can use it.
     * 这些信息已导出以便计划器costsize.c可以使用
     */
    
    /* Target bucket loading (tuples per bucket) */
    #define NTUP_PER_BUCKET         1
    
    void
    ExecChooseHashTableSize(double ntuples, int tupwidth, bool useskew,
                            bool try_combined_work_mem,
                            int parallel_workers,
                            size_t *space_allowed,
                            int *numbuckets,
                            int *numbatches,
                            int *num_skew_mcvs)
    {
        int         tupsize;//元组大小
        double      inner_rel_bytes;//inner relation大小
        long        bucket_bytes;//桶大小
        long        hash_table_bytes;//hash table大小
        long        skew_table_bytes;//倾斜表大小
        long        max_pointers;//最大的指针数
        long        mppow2;//
        int         nbatch = 1;//批次
        int         nbuckets;//桶数
        double      dbuckets;//
    
        /* Force a plausible relation size if no info */
        //如relation大小没有信息,则设定为默认值1000.0
        if (ntuples <= 0.0)
            ntuples = 1000.0;
    
        /*
         * Estimate tupsize based on footprint of tuple in hashtable... note this
         * does not allow for any palloc overhead.  The manipulations of spaceUsed
         * don't count palloc overhead either.
         * 根据哈希表中tuple的占用空间估计tupsize…
         * 注意，这不允许任何palloc开销。使用的空间操作也不包括palloc开销。
         */
        tupsize = HJTUPLE_OVERHEAD +
            MAXALIGN(SizeofMinimalTupleHeader) +
            MAXALIGN(tupwidth);//估算元组大小
        inner_rel_bytes = ntuples * tupsize;//inner relation大小
    
        /*
         * Target in-memory hashtable size is work_mem kilobytes.
         * 目标内存中的散列表大小为work_mem KB。
         */
        hash_table_bytes = work_mem * 1024L;
    
        /*
         * Parallel Hash tries to use the combined work_mem of all workers to
         * avoid the need to batch.  If that won't work, it falls back to work_mem
         * per worker and tries to process batches in parallel.
         * 并行散列试图使用所有worker的所有work_mem来避免分批处理。
         * 如果这不起作用，它将返回到每个worker的work_mem，并尝试并行处理批处理。
         */
        if (try_combined_work_mem)//尝试融合work_mem
            hash_table_bytes += hash_table_bytes * parallel_workers;
    
        *space_allowed = hash_table_bytes;
    
        /*
         * If skew optimization is possible, estimate the number of skew buckets
         * that will fit in the memory allowed, and decrement the assumed space
         * available for the main hash table accordingly.
         * 如果可以进行倾斜优化，估算允许内存中容纳的倾斜桶的数量，并相应地减少主哈希表的假定可用空间。
         *
         * We make the optimistic assumption that each skew bucket will contain
         * one inner-relation tuple.  If that turns out to be low, we will recover
         * at runtime by reducing the number of skew buckets.
         * 我们乐观地假设，每个倾斜桶将包含一个内部关系元组。
         * 如果结果很低，将通过减少倾斜桶的数量在运行时进行恢复。
         *
         * hashtable->skewBucket will have up to 8 times as many HashSkewBucket
         * pointers as the number of MCVs we allow, since ExecHashBuildSkewHash
         * will round up to the next power of 2 and then multiply by 4 to reduce
         * collisions.
         * hashtable->skewBucket的指针数量将是允许的mcv数量的8倍，
         *   因为ExecHashBuildSkewHash将四舍五入到下一个2次方，然后乘以4以减少冲突。
         */
        if (useskew)
        {
            //倾斜优化
            skew_table_bytes = hash_table_bytes * SKEW_WORK_MEM_PERCENT / 100;
    
            /*----------             * Divisor is:
             * size of a hash tuple +
             * worst-case size of skewBucket[] per MCV +
             * size of skewBucketNums[] entry +
             * size of skew bucket struct itself
             *----------             */
            *num_skew_mcvs = skew_table_bytes / (tupsize +
                                                 (8 * sizeof(HashSkewBucket *)) +
                                                 sizeof(int) +
                                                 SKEW_BUCKET_OVERHEAD);
            if (*num_skew_mcvs > 0)
                hash_table_bytes -= skew_table_bytes;
        }
        else
            *num_skew_mcvs = 0;//不使用倾斜优化,默认为0
    
        /*
         * Set nbuckets to achieve an average bucket load of NTUP_PER_BUCKET when
         * memory is filled, assuming a single batch; but limit the value so that
         * the pointer arrays we'll try to allocate do not exceed work_mem nor
         * MaxAllocSize.
         * 设置nbuckets，假设为单批处理，当内存被填满时，实现NTUP_PER_BUCKET的平均桶负载;
         *   但是要限制这个值，以便试图分配的指针数组不会超过work_mem或MaxAllocSize。
         *
         * Note that both nbuckets and nbatch must be powers of 2 to make
         * ExecHashGetBucketAndBatch fast.
         * 注意，nbucket和nbatch都必须是2的幂，才能使ExecHashGetBucketAndBatch更快。
         */
        max_pointers = *space_allowed / sizeof(HashJoinTuple);//最大指针数
        max_pointers = Min(max_pointers, MaxAllocSize / sizeof(HashJoinTuple));//控制上限
        /* If max_pointers isn't a power of 2, must round it down to one */
        //如果max_pointer不是2的幂，则必须四舍五入到符合规则的某个值(如110.1 --> 128)
        mppow2 = 1L << my_log2(max_pointers);
        if (max_pointers != mppow2)
            max_pointers = mppow2 / 2;
    
        /* Also ensure we avoid integer overflow in nbatch and nbuckets */
        /* (this step is redundant given the current value of MaxAllocSize) */
        //还要确保在nbatch和nbucket中避免整数溢出
        //(鉴于MaxAllocSize的当前值，此步骤是多余的)
        max_pointers = Min(max_pointers, INT_MAX / 2);//设定上限
    
        dbuckets = ceil(ntuples / NTUP_PER_BUCKET);//取整
        dbuckets = Min(dbuckets, max_pointers);//设定上限
        nbuckets = (int) dbuckets;//桶数
        /* don't let nbuckets be really small, though ... */
        //但是，不要让nbucket非常小……
        nbuckets = Max(nbuckets, 1024);//设定下限(1024)
        /* ... and force it to be a power of 2. */
        //2的幂
        nbuckets = 1 << my_log2(nbuckets);
    
        /*
         * If there's not enough space to store the projected number of tuples and
         * the required bucket headers, we will need multiple batches.
         * 如果没有足够的空间来存储预计的元组数量和所需的bucket headers，将需要多个批处理。
         */
        bucket_bytes = sizeof(HashJoinTuple) * nbuckets;
        if (inner_rel_bytes + bucket_bytes > hash_table_bytes)//inner relation大小 + 桶数大于可用空间
        {
            /* We'll need multiple batches */
            //需要多批次
            long        lbuckets;
            double      dbatch;
            int         minbatch;
            long        bucket_size;
    
            /*
             * If Parallel Hash with combined work_mem would still need multiple
             * batches, we'll have to fall back to regular work_mem budget.
             * 如果合并了work_mem的并行散列仍然需要多个批处理，将不得不回到常规的work_mem预算。
             */
            if (try_combined_work_mem)
            {
                ExecChooseHashTableSize(ntuples, tupwidth, useskew,
                                        false, parallel_workers,
                                        space_allowed,
                                        numbuckets,
                                        numbatches,
                                        num_skew_mcvs);
                return;
            }
    
            /*
             * Estimate the number of buckets we'll want to have when work_mem is
             * entirely full.  Each bucket will contain a bucket pointer plus
             * NTUP_PER_BUCKET tuples, whose projected size already includes
             * overhead for the hash code, pointer to the next tuple, etc.
             * 估计work_mem完全用完时需要的桶数。
             * 每个桶将包含一个桶指针和NTUP_PER_BUCKET元组，
             *   其投影大小已经包括哈希码的开销、指向下一个元组的指针等等。
             */
            bucket_size = (tupsize * NTUP_PER_BUCKET + sizeof(HashJoinTuple));//桶大小
            lbuckets = 1L << my_log2(hash_table_bytes / bucket_size);
            lbuckets = Min(lbuckets, max_pointers);
            nbuckets = (int) lbuckets;
            nbuckets = 1 << my_log2(nbuckets);
            bucket_bytes = nbuckets * sizeof(HashJoinTuple);
    
            /*
             * Buckets are simple pointers to hashjoin tuples, while tupsize
             * includes the pointer, hash code, and MinimalTupleData.  So buckets
             * should never really exceed 25% of work_mem (even for
             * NTUP_PER_BUCKET=1); except maybe for work_mem values that are not
             * 2^N bytes, where we might get more because of doubling. So let's
             * look for 50% here.
             * Buckets是指向hashjoin元组的简单指针，而tupsize包含指针、散列代码和MinimalTupleData。
             * 所以Buckets的实际大小不应该超过work_mem的25%(即使对于NTUP_PER_BUCKET=1);
             *   除了work_mem值不是2 ^ N个字节这个原因外,翻倍可能会得到更多的,这里试着使用50%
             */
            Assert(bucket_bytes <= hash_table_bytes / 2);
    
            /* Calculate required number of batches. */
            //计算批次数
            dbatch = ceil(inner_rel_bytes / (hash_table_bytes - bucket_bytes));
            dbatch = Min(dbatch, max_pointers);
            minbatch = (int) dbatch;
            nbatch = 2;
            while (nbatch < minbatch)
                nbatch <<= 1;
        }
    
        Assert(nbuckets > 0);
        Assert(nbatch > 0);
    
        *numbuckets = nbuckets;
        *numbatches = nbatch;
    }
    
    

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
    

启动gdb,设置断点,进入ExecHashTableCreate

    
    
    (gdb) b ExecHashTableCreate
    Breakpoint 1 at 0x6fc75d: file nodeHash.c, line 449.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecHashTableCreate (state=0x1e3cbc8, hashOperators=0x1e59890, keepNulls=false) at nodeHash.c:449
    449     node = (Hash *) state->ps.plan;
    

获取相关信息

    
    
    449     node = (Hash *) state->ps.plan;
    (gdb) n
    450     outerNode = outerPlan(node);
    (gdb) 
    457     rows = node->plan.parallel_aware ? node->rows_total : outerNode->plan_rows;
    (gdb) 
    462                             state->parallel_state != NULL ?
    (gdb) 
    459     ExecChooseHashTableSize(rows, outerNode->plan_width,
    (gdb) 
    
    

获取Hash节点;  
outer节点为顺序扫描SeqScan节点  
inner(构造hash表的relation)行数为10000

    
    
    (gdb) p *node
    $1 = {plan = {type = T_Hash, startup_cost = 164, total_cost = 164, plan_rows = 10000, plan_width = 20, 
        parallel_aware = false, parallel_safe = true, plan_node_id = 4, targetlist = 0x1e4bf90, qual = 0x0, 
        lefttree = 0x1e493e8, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}, skewTable = 16977, 
      skewColumn = 1, skewInherit = false, rows_total = 0}
    (gdb) p *outerNode
    $2 = {type = T_SeqScan, startup_cost = 0, total_cost = 164, plan_rows = 10000, plan_width = 20, parallel_aware = false, 
      parallel_safe = true, plan_node_id = 5, targetlist = 0x1e492b0, qual = 0x0, lefttree = 0x0, righttree = 0x0, 
      initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    (gdb) p rows
    $3 = 10000
    (gdb) 
    

进入ExecChooseHashTableSize函数

    
    
    (gdb) step
    ExecChooseHashTableSize (ntuples=10000, tupwidth=20, useskew=true, try_combined_work_mem=false, parallel_workers=0, 
        space_allowed=0x7ffdcf148540, numbuckets=0x7ffdcf14853c, numbatches=0x7ffdcf148538, num_skew_mcvs=0x7ffdcf148534)
        at nodeHash.c:677
    677     int         nbatch = 1;
    

ExecChooseHashTableSize->计算元组大小(56B)/inner relation大小(约560K)/hash表空间(16M)

    
    
    (gdb) n
    682     if (ntuples <= 0.0)
    (gdb) 
    690     tupsize = HJTUPLE_OVERHEAD +
    (gdb) 
    693     inner_rel_bytes = ntuples * tupsize;
    (gdb) 
    698     hash_table_bytes = work_mem * 1024L;
    (gdb) 
    705     if (try_combined_work_mem)
    (gdb) p tupsize
    $4 = 56
    (gdb) p inner_rel_bytes
    $5 = 560000
    (gdb) p hash_table_bytes
    $6 = 16777216
    

ExecChooseHashTableSize->使用数据倾斜优化(所需空间从Hash Table中获取)

    
    
    (gdb) n
    708     *space_allowed = hash_table_bytes;
    (gdb) 
    724     if (useskew)
    (gdb) 
    726         skew_table_bytes = hash_table_bytes * SKEW_WORK_MEM_PERCENT / 100;
    (gdb) p useskew
    $8 = true
    (gdb) p hash_table_bytes
    $9 = 16441672
    (gdb) p skew_table_bytes
    $10 = 335544
    (gdb) p num_skew_mcvs
    $11 = (int *) 0x7ffdcf148534
    (gdb) p *num_skew_mcvs
    $12 = 2396
    (gdb) 
    

ExecChooseHashTableSize->获取最大指针数目(2097152)

    
    
    (gdb) n
    756     max_pointers = Min(max_pointers, MaxAllocSize / sizeof(HashJoinTuple));
    (gdb) 
    758     mppow2 = 1L << my_log2(max_pointers);
    (gdb) n
    759     if (max_pointers != mppow2)
    (gdb) p max_pointers
    $13 = 2097152
    (gdb) p mppow2
    $15 = 2097152
    

ExecChooseHashTableSize->计算Hash桶数

    
    
    (gdb) n
    764     max_pointers = Min(max_pointers, INT_MAX / 2);
    (gdb) 
    766     dbuckets = ceil(ntuples / NTUP_PER_BUCKET);
    (gdb) 
    767     dbuckets = Min(dbuckets, max_pointers);
    (gdb) 
    768     nbuckets = (int) dbuckets;
    (gdb) 
    770     nbuckets = Max(nbuckets, 1024);
    (gdb) 
    772     nbuckets = 1 << my_log2(nbuckets);
    (gdb) 
    778     bucket_bytes = sizeof(HashJoinTuple) * nbuckets;
    (gdb) n
    779     if (inner_rel_bytes + bucket_bytes > hash_table_bytes)
    (gdb) 
    834     Assert(nbuckets > 0);
    (gdb) p dbuckets
    $16 = 10000
    (gdb) p nbuckets
    $17 = 16384
    (gdb) p bucket_bytes
    $18 = 131072
    

ExecChooseHashTableSize->只需要一个批次,赋值,返回

    
    
    835     Assert(nbatch > 0);
    (gdb) 
    837     *numbuckets = nbuckets;
    (gdb) 
    838     *numbatches = nbatch;
    (gdb) 
    839 }
    (gdb) 
    (gdb) 
    ExecHashTableCreate (state=0x1e3cbc8, hashOperators=0x1e59890, keepNulls=false) at nodeHash.c:468
    468     log2_nbuckets = my_log2(nbuckets);
    

初始化Hash表

    
    
    468     log2_nbuckets = my_log2(nbuckets);
    (gdb) p nbuckets
    $19 = 16384
    (gdb) n
    469     Assert(nbuckets == (1 << log2_nbuckets));
    (gdb) 
    478     hashtable = (HashJoinTable) palloc(sizeof(HashJoinTableData));
    (gdb) 
    479     hashtable->nbuckets = nbuckets;
    ...
    

分配内存上下文

    
    
    ...
    (gdb) 
    522     hashtable->hashCxt = AllocSetContextCreate(CurrentMemoryContext,
    (gdb) 
    526     hashtable->batchCxt = AllocSetContextCreate(hashtable->hashCxt,
    (gdb) 
    532     oldcxt = MemoryContextSwitchTo(hashtable->hashCxt);
    (gdb) 
    

切换上下文,并初始化hash函数

    
    
    (gdb) 
    532     oldcxt = MemoryContextSwitchTo(hashtable->hashCxt);
    (gdb) n
    538     nkeys = list_length(hashOperators);
    (gdb) 
    540         (FmgrInfo *) palloc(nkeys * sizeof(FmgrInfo));
    (gdb) p nkeys
    $20 = 1
    (gdb) n
    539     hashtable->outer_hashfunctions =
    (gdb) 
    542         (FmgrInfo *) palloc(nkeys * sizeof(FmgrInfo));
    (gdb) 
    541     hashtable->inner_hashfunctions =
    (gdb) 
    543     hashtable->hashStrict = (bool *) palloc(nkeys * sizeof(bool));
    (gdb) 
    544     i = 0;
    

初始化Hash操作符

    
    
    (gdb) n
    545     foreach(ho, hashOperators)
    (gdb) 
    547         Oid         hashop = lfirst_oid(ho);
    (gdb) 
    551         if (!get_op_hash_functions(hashop, &left_hashfn, &right_hashfn))
    (gdb) 
    554         fmgr_info(left_hashfn, &hashtable->outer_hashfunctions[i]);
    (gdb) 
    555         fmgr_info(right_hashfn, &hashtable->inner_hashfunctions[i]);
    (gdb) 
    556         hashtable->hashStrict[i] = op_strict(hashop);
    (gdb) 
    557         i++;
    (gdb) 
    545     foreach(ho, hashOperators)
    (gdb) p *hashtable->hashStrict
    $21 = true
    (gdb) n
    560     if (nbatch > 1 && hashtable->parallel_state == NULL)
    

分配hash桶内存空间

    
    
    gdb) n
    575     MemoryContextSwitchTo(oldcxt);
    (gdb) 
    577     if (hashtable->parallel_state)
    (gdb) 
    631         MemoryContextSwitchTo(hashtable->batchCxt);
    (gdb) 
    634             palloc0(nbuckets * sizeof(HashJoinTuple));
    (gdb) 
    633         hashtable->buckets.unshared = (HashJoinTuple *)
    (gdb) p nbuckets
    $23 = 16384
    

构造完成,返回hash表

    
    
    (gdb) n
    641         if (nbatch > 1)
    (gdb) 
    644         MemoryContextSwitchTo(oldcxt);
    (gdb) 
    647     return hashtable;
    (gdb) 
    648 }
    (gdb) 
    ExecHashJoinImpl (pstate=0x1e3c048, parallel=false) at nodeHashjoin.c:282
    282                 node->hj_HashTable = hashtable;
    (gdb) 
    

DONE!

### 四、参考资料

[Hash Joins: Past, Present and Future/PGCon
2017](https://www.pgcon.org/2017/schedule/events/1053.en.html)  
[A Look at How Postgres Executes a Tiny Join - Part
1](https://dzone.com/articles/a-look-at-how-postgres-executes-a-tiny-join-part-1)  
[A Look at How Postgres Executes a Tiny Join - Part
2](https://dzone.com/articles/a-look-at-how-postgres-executes-a-tiny-join-part-2)  
[Assignment 2 Symmetric Hash
Join](https://cs.uwaterloo.ca/~david/cs448/A2.pdf)

