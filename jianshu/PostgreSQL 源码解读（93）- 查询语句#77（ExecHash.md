本节是ExecHashJoin函数介绍的第三部分，主要介绍了ExecHashJoin中依赖的其他函数的实现逻辑，这些函数在HJ_NEED_NEW_OUTER阶段中使用，包括ExecHashJoinOuterGetTuple、ExecPrepHashTableForUnmatched、ExecHashGetBucketAndBatch、ExecHashGetSkewBucket、ExecHashJoinSaveTuple和ExecFetchSlotMinimalTuple等。

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

**ExecHashJoinOuterGetTuple**  
获取非并行模式下hashjoin的下一个外部元组:要么在第一次执行外部plan节点，要么从hashjoin批处理的临时文件中获取。

    
    
    /*----------------------------------------------------------------------------------------------------                                        HJ_NEED_NEW_OUTER 阶段
    ----------------------------------------------------------------------------------------------------*/
    /*
     * ExecHashJoinOuterGetTuple
     *
     *      get the next outer tuple for a parallel oblivious hashjoin: either by
     *      executing the outer plan node in the first pass, or from the temp
     *      files for the hashjoin batches.
     *      获取非并行模式下hashjoin的下一个外部元组:要么在第一次执行外部plan节点，要么从hashjoin批处理的临时文件中获取。
     *
     * Returns a null slot if no more outer tuples (within the current batch).
     * 如果没有更多外部元组(在当前批处理中)，则返回空slot。
     *
     * On success, the tuple's hash value is stored at *hashvalue --- this is
     * either originally computed, or re-read from the temp file.
     * 如果成功，tuple的散列值存储在输入参数*hashvalue中——这是最初计算的，或者是从临时文件中重新读取的。
     */
    static TupleTableSlot *
    ExecHashJoinOuterGetTuple(PlanState *outerNode,//outer 节点
                              HashJoinState *hjstate,//Hash Join执行状态
                              uint32 *hashvalue)//Hash值
    {
        HashJoinTable hashtable = hjstate->hj_HashTable;//hash表
        int         curbatch = hashtable->curbatch;//当前批次
        TupleTableSlot *slot;//返回的slot
    
        if (curbatch == 0)          /* 第一个批次;if it is the first pass */
        {
            /*
             * Check to see if first outer tuple was already fetched by
             * ExecHashJoin() and not used yet.
             * 检查第一个外部元组是否已经由ExecHashJoin()函数获取且尚未使用。
             */
            slot = hjstate->hj_FirstOuterTupleSlot;
            if (!TupIsNull(slot))
                hjstate->hj_FirstOuterTupleSlot = NULL;//重置slot
            else
                slot = ExecProcNode(outerNode);//如为NULL,则获取slot
    
            while (!TupIsNull(slot))//slot不为NULL
            {
                /*
                 * We have to compute the tuple's hash value.
                 * 计算hash值
                 */
                ExprContext *econtext = hjstate->js.ps.ps_ExprContext;//表达式计算上下文
    
                econtext->ecxt_outertuple = slot;//存储获取的slot
                if (ExecHashGetHashValue(hashtable, econtext,
                                         hjstate->hj_OuterHashKeys,
                                         true,  /* outer tuple */
                                         HJ_FILL_OUTER(hjstate),
                                         hashvalue))//计算Hash值
                {
                    /* remember outer relation is not empty for possible rescan */
                    hjstate->hj_OuterNotEmpty = true;//设置标记(outer不为空)
    
                    return slot;//返回匹配的slot
                }
    
                /*
                 * That tuple couldn't match because of a NULL, so discard it and
                 * continue with the next one.
                 * 该元组无法匹配，丢弃它，继续下一个元组。
                 */
                slot = ExecProcNode(outerNode);//继续获取下一个
            }
        }
        else if (curbatch < hashtable->nbatch)//不是第一个批次
        {
            BufFile    *file = hashtable->outerBatchFile[curbatch];//获取缓冲的文件
    
            /*
             * In outer-join cases, we could get here even though the batch file
             * is empty.
             * 在外连接的情况下，即使批处理文件是空的，也可以在这里进行处理。
             */
            if (file == NULL)
                return NULL;//如文件为NULL,则返回
    
            slot = ExecHashJoinGetSavedTuple(hjstate,
                                             file,
                                             hashvalue,
                                             hjstate->hj_OuterTupleSlot);//从文件中获取slot
            if (!TupIsNull(slot))
                return slot;//非NULL,则返回
        }
    
        /* End of this batch */
        //已完成,则返回NULL
        return NULL;
    }
    
    
    /*
     * ExecHashGetHashValue
     *      Compute the hash value for a tuple
     *      ExecHashGetHashValue - 计算元组的Hash值
     *
     * The tuple to be tested must be in either econtext->ecxt_outertuple or
     * econtext->ecxt_innertuple.  Vars in the hashkeys expressions should have
     * varno either OUTER_VAR or INNER_VAR.
     * 要测试的元组必须位于econtext->ecxt_outertuple或econtext->ecxt_innertuple中。
     * hashkeys表达式中的Vars应该具有varno，即OUTER_VAR或INNER_VAR。
     *
     * A true result means the tuple's hash value has been successfully computed
     * and stored at *hashvalue.  A false result means the tuple cannot match
     * because it contains a null attribute, and hence it should be discarded
     * immediately.  (If keep_nulls is true then false is never returned.)
     * T意味着tuple的散列值已经成功计算并存储在*hashvalue参数中。
     * F意味着元组不能匹配，因为它包含null属性，因此应该立即丢弃它。
     * (如果keep_nulls为真，则永远不会返回F。)
     */
    bool
    ExecHashGetHashValue(HashJoinTable hashtable,//Hash表
                         ExprContext *econtext,//上下文
                         List *hashkeys,//Hash键值链表
                         bool outer_tuple,//是否外表元组
                         bool keep_nulls,//是否保存NULL
                         uint32 *hashvalue)//返回的Hash值
    {
        uint32      hashkey = 0;//hash键
        FmgrInfo   *hashfunctions;//hash函数
        ListCell   *hk;//临时变量
        int         i = 0;
        MemoryContext oldContext;
    
        /*
         * We reset the eval context each time to reclaim any memory leaked in the
         * hashkey expressions.
         * 我们每次重置eval上下文来回收hashkey表达式中分配的内存。
         */
        ResetExprContext(econtext);
        //切换上下文
        oldContext = MemoryContextSwitchTo(econtext->ecxt_per_tuple_memory);
    
        if (outer_tuple)
            hashfunctions = hashtable->outer_hashfunctions;//外表元组
        else
            hashfunctions = hashtable->inner_hashfunctions;//内表元组
    
        foreach(hk, hashkeys)//遍历Hash键值
        {
            ExprState  *keyexpr = (ExprState *) lfirst(hk);//键值表达式
            Datum       keyval;
            bool        isNull;
    
            /* rotate hashkey left 1 bit at each step */
            //哈希键左移1位
            hashkey = (hashkey << 1) | ((hashkey & 0x80000000) ? 1 : 0);
    
            /*
             * Get the join attribute value of the tuple
             * 获取元组的连接属性值
             */
            keyval = ExecEvalExpr(keyexpr, econtext, &isNull);
    
            /*
             * If the attribute is NULL, and the join operator is strict, then
             * this tuple cannot pass the join qual so we can reject it
             * immediately (unless we're scanning the outside of an outer join, in
             * which case we must not reject it).  Otherwise we act like the
             * hashcode of NULL is zero (this will support operators that act like
             * IS NOT DISTINCT, though not any more-random behavior).  We treat
             * the hash support function as strict even if the operator is not.
             * 如果属性为NULL，并且join操作符是严格的，那么这个元组不能传递连接条件join qual，
             *   因此可以立即拒绝它(除非正在扫描外连接的外表，在这种情况下不能拒绝它)。
             * 否则，我们的行为就好像NULL的哈希码是零一样(这将支持IS NOT DISTINCT操作符，但不会有任何随机的情况出现)。
             * 即使操作符不是严格的，也将哈希函数视为严格的。
             *
             * Note: currently, all hashjoinable operators must be strict since
             * the hash index AM assumes that.  However, it takes so little extra
             * code here to allow non-strict that we may as well do it.
             * 注意:目前，所有哈希可连接操作符都必须严格，因为哈希索引AM假定如此。
             *      但是，这里只需要很少的额外代码就可以实现非严格性，我们也可以这样做。
             */
            if (isNull)
            {
                //NULL值
                if (hashtable->hashStrict[i] && !keep_nulls)
                {
                    MemoryContextSwitchTo(oldContext);
                    //不保持NULL值,不匹配
                    return false;   /* cannot match */
                }
                /* else, leave hashkey unmodified, equivalent to hashcode 0 */
                //否则的话,不修改hashkey,仍为0
            }
            else
            {
                //不为NULL
                /* Compute the hash function */
                //计算hash值
                uint32      hkey;
    
                hkey = DatumGetUInt32(FunctionCall1(&hashfunctions[i], keyval));
                hashkey ^= hkey;
            }
    
            i++;//下一个键
        }
        //切换上下文
        MemoryContextSwitchTo(oldContext);
        //返回Hash键值
        *hashvalue = hashkey;
        return true;//成功获取
    }
    
    

**ExecPrepHashTableForUnmatched**  
为ExecScanHashTableForUnmatched函数调用作准备

    
    
    /*
     * ExecPrepHashTableForUnmatched
     *      set up for a series of ExecScanHashTableForUnmatched calls
     *      为ExecScanHashTableForUnmatched函数调用作准备
     */
    void
    ExecPrepHashTableForUnmatched(HashJoinState *hjstate)
    {
        /*----------         * During this scan we use the HashJoinState fields as follows:
         *
         * hj_CurBucketNo: next regular bucket to scan
         * hj_CurSkewBucketNo: next skew bucket (an index into skewBucketNums)
         * hj_CurTuple: last tuple returned, or NULL to start next bucket
         * 在这次扫描期间,我们使用HashJoinState结构体中的字段如下:
         * hj_CurBucketNo: 下一个常规的bucket
         * hj_CurSkewBucketNo: 下一个个倾斜的bucket
         * hj_CurTuple: 最后返回的元组,或者为NULL(下一个bucket开始)
         *----------         */
        hjstate->hj_CurBucketNo = 0;
        hjstate->hj_CurSkewBucketNo = 0;
        hjstate->hj_CurTuple = NULL;
    }
    
    
    

**ExecHashGetBucketAndBatch**  
确定哈希值的bucket号和批处理号

    
    
    /*
     * ExecHashGetBucketAndBatch
     *      Determine the bucket number and batch number for a hash value
     * ExecHashGetBucketAndBatch
     *      确定哈希值的bucket号和批处理号
     * 
     * Note: on-the-fly increases of nbatch must not change the bucket number
     * for a given hash code (since we don't move tuples to different hash
     * chains), and must only cause the batch number to remain the same or
     * increase.  Our algorithm is
     *      bucketno = hashvalue MOD nbuckets
     *      batchno = (hashvalue DIV nbuckets) MOD nbatch
     * where nbuckets and nbatch are both expected to be powers of 2, so we can
     * do the computations by shifting and masking.  (This assumes that all hash
     * functions are good about randomizing all their output bits, else we are
     * likely to have very skewed bucket or batch occupancy.)
     * 注意:nbatch的动态增加不能更改给定哈希码的桶号(因为我们不将元组移动到不同的哈希链)，
     *   并且只能使批号保持不变或增加。我们的算法是:
     *      bucketno = hashvalue MOD nbuckets
     *      batchno = (hashvalue DIV nbuckets) MOD nbatch
     * 这里nbucket和nbatch都是2的幂，所以我们可以通过移动和屏蔽来进行计算。
     * (这假定所有哈希函数都能很好地随机化它们的所有输出位，否则很可能会出现非常倾斜的桶或批处理占用。)
     *
     * nbuckets and log2_nbuckets may change while nbatch == 1 because of dynamic
     * bucket count growth.  Once we start batching, the value is fixed and does
     * not change over the course of the join (making it possible to compute batch
     * number the way we do here).
     * 当nbatch == 1时，由于动态bucket计数的增长，nbucket和log2_nbucket可能会发生变化。
     * 一旦开始批处理，这个值就固定了，并且在连接过程中不会改变(这使得我们可以像这里那样计算批号)。
     *
     * nbatch is always a power of 2; we increase it only by doubling it.  This
     * effectively adds one more bit to the top of the batchno.
     * nbatch总是2的幂;我们只是通过x2来调整。这相当于为批号的头部增加了一位。
     */
    void
    ExecHashGetBucketAndBatch(HashJoinTable hashtable,
                              uint32 hashvalue,
                              int *bucketno,
                              int *batchno)
    {
        uint32      nbuckets = (uint32) hashtable->nbuckets;//桶数
        uint32      nbatch = (uint32) hashtable->nbatch;//批次号
    
        if (nbatch > 1)//批次>1
        {
            /* we can do MOD by masking, DIV by shifting */
            //我们可以通过屏蔽来实现MOD，通过移动来实现DIV
            *bucketno = hashvalue & (nbuckets - 1);//nbuckets - 1后相当于N个1
            *batchno = (hashvalue >> hashtable->log2_nbuckets) & (nbatch - 1);
        }
        else
        {
            *bucketno = hashvalue & (nbuckets - 1);//只有一个批次,简单处理即可
            *batchno = 0;
        }
    }
    
    
    

**ExecHashGetSkewBucket**  
返回这个哈希值的倾斜桶的索引，如果哈希值与任何活动的倾斜桶没有关联，则返回INVALID_SKEW_BUCKET_NO。

    
    
    /*
     * ExecHashGetSkewBucket
     *
     *      Returns the index of the skew bucket for this hashvalue,
     *      or INVALID_SKEW_BUCKET_NO if the hashvalue is not
     *      associated with any active skew bucket.
     *      返回这个哈希值的倾斜桶的索引，如果哈希值与任何活动的倾斜桶没有关联，则返回INVALID_SKEW_BUCKET_NO。
     */
    int
    ExecHashGetSkewBucket(HashJoinTable hashtable, uint32 hashvalue)
    {
        int         bucket;
    
        /*
         * Always return INVALID_SKEW_BUCKET_NO if not doing skew optimization (in
         * particular, this happens after the initial batch is done).
         * 如果不进行倾斜优化(特别是在初始批处理完成之后)，则返回INVALID_SKEW_BUCKET_NO。
         */
        if (!hashtable->skewEnabled)
            return INVALID_SKEW_BUCKET_NO;
    
        /*
         * Since skewBucketLen is a power of 2, we can do a modulo by ANDing.'
         * 由于skewBucketLen是2的幂，可以通过AND操作来做一个模。
         */
        bucket = hashvalue & (hashtable->skewBucketLen - 1);
    
        /*
         * While we have not hit a hole in the hashtable and have not hit the
         * desired bucket, we have collided with some other hash value, so try the
         * next bucket location.
         * 虽然我们没有在哈希表中找到一个hole，也没有找到所需的bucket，
         *   但是与其他一些哈希值发生了冲突，所以尝试下一个bucket位置。
         */
        while (hashtable->skewBucket[bucket] != NULL &&
               hashtable->skewBucket[bucket]->hashvalue != hashvalue)
            bucket = (bucket + 1) & (hashtable->skewBucketLen - 1);
    
        /*
         * Found the desired bucket?
         * 找到了bucket,返回
         */
        if (hashtable->skewBucket[bucket] != NULL)
            return bucket;
    
        /*
         * There must not be any hashtable entry for this hash value.
         */
        //否则返回INVALID_SKEW_BUCKET_NO
        return INVALID_SKEW_BUCKET_NO;
    }
    
    

**ExecHashJoinSaveTuple**  
在批处理文件中保存元组.每个元组在文件中记录的是它的散列值，然后是最小化格式的元组。

    
    
    /*
     * ExecHashJoinSaveTuple
     *      save a tuple to a batch file.
     *      在批处理文件中保存元组
     * 
     * The data recorded in the file for each tuple is its hash value,
     * then the tuple in MinimalTuple format.
     * 每个元组在文件中记录的是它的散列值，然后是最小化格式的元组。
     * 
     * Note: it is important always to call this in the regular executor
     * context, not in a shorter-lived context; else the temp file buffers
     * will get messed up.
     * 注意:在常规执行程序上下文中调用它总是很重要的，而不是在较短的生命周期中调用它;
     *   否则临时文件缓冲区就会出现混乱。
     */
    void
    ExecHashJoinSaveTuple(MinimalTuple tuple, uint32 hashvalue,
                          BufFile **fileptr)
    {
        BufFile    *file = *fileptr;//文件指针
        size_t      written;//写入大小
    
        if (file == NULL)
        {
            /* First write to this batch file, so open it. */
            //文件指针为NULL,首次写入,则打开批处理文件
            file = BufFileCreateTemp(false);
            *fileptr = file;
        }
        //首先写入hash值,返回写入的大小
        written = BufFileWrite(file, (void *) &hashvalue, sizeof(uint32));
        if (written != sizeof(uint32))//写入有误,报错
            ereport(ERROR,
                    (errcode_for_file_access(),
                     errmsg("could not write to hash-join temporary file: %m")));
        //写入tuple
        written = BufFileWrite(file, (void *) tuple, tuple->t_len);
        if (written != tuple->t_len)//写入有误,报错
            ereport(ERROR,
                    (errcode_for_file_access(),
                     errmsg("could not write to hash-join temporary file: %m")));
    }
    
    
    

**ExecFetchSlotMinimalTuple**  
以最小化物理元组的格式提取slot的数据

    
    
    /* --------------------------------     *      ExecFetchSlotMinimalTuple
     *          Fetch the slot's minimal physical tuple.
     *          以最小化物理元组的格式提取slot的数据.
     *
     *      If the given tuple table slot can hold a minimal tuple, indicated by a
     *      non-NULL get_minimal_tuple callback, the function returns the minimal
     *      tuple returned by that callback. It assumes that the minimal tuple
     *      returned by the callback is "owned" by the slot i.e. the slot is
     *      responsible for freeing the memory consumed by the tuple. Hence it sets
     *      *shouldFree to false, indicating that the caller should not free the
     *      memory consumed by the minimal tuple. In this case the returned minimal
     *      tuple should be considered as read-only.
     *      如果给定的元组table slot可以保存由non-NULL get_minimal_tuple回调函数指示的最小元组，
     *        则函数将返回该回调函数返回的最小元组。
     *      它假定回调函数返回的最小元组由slot“拥有”，即slot负责释放元组所消耗的内存。
     *      因此，它将*shouldFree设置为false，表示调用方不应该释放内存。
     *      在这种情况下，返回的最小元组应该被认为是只读的。
     *
     *      If that callback is not supported, it calls copy_minimal_tuple callback
     *      which is expected to return a copy of minimal tuple represnting the
     *      contents of the slot. In this case *shouldFree is set to true,
     *      indicating the caller that it should free the memory consumed by the
     *      minimal tuple. In this case the returned minimal tuple may be written
     *      up.
     *      如果不支持该回调函数，则调用copy_minimal_tuple回调函数，
     *        该回调将返回一个表示slot内容的最小元组副本。
     *      *shouldFree被设置为true，这表示调用者应该释放内存。
     *      在这种情况下，可以写入返回的最小元组。
     * --------------------------------     */
    MinimalTuple
    ExecFetchSlotMinimalTuple(TupleTableSlot *slot,
                              bool *shouldFree)
    {
        /*
         * sanity checks
         * 安全检查
         */
        Assert(slot != NULL);
        Assert(!TTS_EMPTY(slot));
    
        if (slot->tts_ops->get_minimal_tuple)//调用slot->tts_ops->get_minimal_tuple
        {
            //调用成功,则该元组为只读,由slot负责释放
            if (shouldFree)
                *shouldFree = false;
            return slot->tts_ops->get_minimal_tuple(slot);
        }
        else
        {
            //调用不成功,设置为true,由调用方释放
            if (shouldFree)
                *shouldFree = true;
            return slot->tts_ops->copy_minimal_tuple(slot);//调用copy_minimal_tuple函数
        }
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
    

启动gdb,设置断点

    
    
    (gdb) b ExecHashJoinOuterGetTuple
    Breakpoint 1 at 0x702edc: file nodeHashjoin.c, line 807.
    (gdb) b ExecHashGetHashValue
    Breakpoint 2 at 0x6ff060: file nodeHash.c, line 1778.
    (gdb) b ExecHashGetBucketAndBatch
    Breakpoint 3 at 0x6ff1df: file nodeHash.c, line 1880.
    (gdb) b ExecHashJoinSaveTuple
    Breakpoint 4 at 0x703973: file nodeHashjoin.c, line 1214.
    (gdb) 
    

**ExecHashGetHashValue**  
ExecHashGetHashValue- >进入函数ExecHashGetHashValue

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 2, ExecHashGetHashValue (hashtable=0x14acde8, econtext=0x149c3d0, hashkeys=0x14a8e40, outer_tuple=false, 
        keep_nulls=false, hashvalue=0x7ffc7eba5c20) at nodeHash.c:1778
    1778        uint32      hashkey = 0;
    

ExecHashGetHashValue->初始化,切换内存上下文

    
    
    1778        uint32      hashkey = 0;
    (gdb) n
    1781        int         i = 0;
    (gdb) 
    1788        ResetExprContext(econtext);
    (gdb) 
    1790        oldContext = MemoryContextSwitchTo(econtext->ecxt_per_tuple_memory);
    (gdb) 
    1792        if (outer_tuple)
    

ExecHashGetHashValue->inner hash函数

    
    
    1792        if (outer_tuple)
    (gdb) 
    1795            hashfunctions = hashtable->inner_hashfunctions;
    

ExecHashGetHashValue->获取hahs键信息  
1号RTE(varnoold = 1,即t_dwxx)的dwbh字段(varattno = 2)

    
    
    (gdb) 
    1797        foreach(hk, hashkeys)
    (gdb) 
    1799            ExprState  *keyexpr = (ExprState *) lfirst(hk);
    (gdb) 
    1804            hashkey = (hashkey << 1) | ((hashkey & 0x80000000) ? 1 : 0);
    (gdb) p *keyexpr
    $1 = {tag = {type = T_ExprState}, flags = 2 '\002', resnull = false, resvalue = 0, resultslot = 0x0, steps = 0x14a8a00, 
      evalfunc = 0x6d1a6e <ExecInterpExprStillValid>, expr = 0x1498fc0, evalfunc_private = 0x6d1e97 <ExecJustInnerVar>, 
      steps_len = 3, steps_alloc = 16, parent = 0x149b738, ext_params = 0x0, innermost_caseval = 0x0, innermost_casenull = 0x0, 
      innermost_domainval = 0x0, innermost_domainnull = 0x0}
    (gdb) p *(RelabelType *)keyexpr->expr
    $3 = {xpr = {type = T_RelabelType}, arg = 0x1499018, resulttype = 25, resulttypmod = -1, resultcollid = 100, 
      relabelformat = COERCE_IMPLICIT_CAST, location = -1}
    (gdb) p *((RelabelType *)keyexpr->expr)->arg
    $4 = {type = T_Var}
    (gdb) p *(Var *)((RelabelType *)keyexpr->expr)->arg
    $5 = {xpr = {type = T_Var}, varno = 65000, varattno = 2, vartype = 1043, vartypmod = 24, varcollid = 100, varlevelsup = 0, 
      varnoold = 1, varoattno = 2, location = 218}
    (gdb) 
    

ExecHashGetHashValue->获取hash值,解析表达式

    
    
    (gdb) n
    1809            keyval = ExecEvalExpr(keyexpr, econtext, &isNull);
    (gdb) 
    1824            if (isNull)
    (gdb) p hashkey
    $6 = 0
    (gdb) p keyval
    $7 = 140460362257270
    (gdb) 
    

ExecHashGetHashValue->返回值不为NULL

    
    
    (gdb) p isNull
    $8 = false
    (gdb) n
    1838                hkey = DatumGetUInt32(FunctionCall1(&hashfunctions[i], keyval));
    

ExecHashGetHashValue->计算Hash值

    
    
    (gdb) n
    1839                hashkey ^= hkey;
    (gdb) p hkey
    $9 = 3663833849
    (gdb) p hashkey
    $10 = 0
    (gdb) n
    1842            i++;
    (gdb) p hashkey
    $11 = 3663833849
    (gdb) 
    

ExecHashGetHashValue->返回计算结果

    
    
    (gdb) n
    1797        foreach(hk, hashkeys)
    (gdb) 
    1845        MemoryContextSwitchTo(oldContext);
    (gdb) 
    1847        *hashvalue = hashkey;
    (gdb) 
    1848        return true;
    (gdb) 
    1849    }
    

**ExecHashGetBucketAndBatch**  
ExecHashGetBucketAndBatch- >进入ExecHashGetBucketAndBatch

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 3, ExecHashGetBucketAndBatch (hashtable=0x14acde8, hashvalue=3663833849, bucketno=0x7ffc7eba5bdc, 
        batchno=0x7ffc7eba5bd8) at nodeHash.c:1880
    1880        uint32      nbuckets = (uint32) hashtable->nbuckets;
    

ExecHashGetBucketAndBatch->获取bucket数和批次数

    
    
    1880        uint32      nbuckets = (uint32) hashtable->nbuckets;
    (gdb) n
    1881        uint32      nbatch = (uint32) hashtable->nbatch;
    (gdb) 
    1883        if (nbatch > 1)
    (gdb) p nbuckets
    $12 = 16384
    (gdb) p nbatch
    $13 = 1
    (gdb) 
    

ExecHashGetBucketAndBatch->计算桶号和批次号(只有一个批次,设置为0)

    
    
    (gdb) n
    1891            *bucketno = hashvalue & (nbuckets - 1);
    (gdb) 
    1892            *batchno = 0;
    (gdb) 
    1894    }
    (gdb) p bucketno
    $14 = (int *) 0x7ffc7eba5bdc
    (gdb) p *bucketno
    $15 = 11001
    (gdb) 
    

**ExecHashJoinOuterGetTuple**  
ExecHashJoinOuterGetTuple- >进入ExecHashJoinOuterGetTuple函数

    
    
    (gdb) info break
    Num     Type           Disp Enb Address            What
    1       breakpoint     keep y   0x0000000000702edc in ExecHashJoinOuterGetTuple at nodeHashjoin.c:807
    2       breakpoint     keep y   0x00000000006ff060 in ExecHashGetHashValue at nodeHash.c:1778
        breakpoint already hit 4 times
    3       breakpoint     keep y   0x00000000006ff1df in ExecHashGetBucketAndBatch at nodeHash.c:1880
        breakpoint already hit 4 times
    4       breakpoint     keep y   0x0000000000703973 in ExecHashJoinSaveTuple at nodeHashjoin.c:1214
    (gdb) del 2
    (gdb) del 3
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecHashJoinOuterGetTuple (outerNode=0x149ba10, hjstate=0x149b738, hashvalue=0x7ffc7eba5ccc)
        at nodeHashjoin.c:807
    807     HashJoinTable hashtable = hjstate->hj_HashTable;
    (gdb) 
    

ExecHashJoinOuterGetTuple->查看输入参数  
outerNode:outer relation为顺序扫描得到的relation(对t_jfxx进行顺序扫描)  
hjstate:Hash Join执行状态  
hashvalue:Hash值

    
    
    (gdb) p *outerNode
    $16 = {type = T_SeqScanState, plan = 0x1494d10, state = 0x149b0f8, ExecProcNode = 0x71578d <ExecSeqScan>, 
      ExecProcNodeReal = 0x71578d <ExecSeqScan>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
      qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
      ps_ResultTupleSlot = 0x149c178, ps_ExprContext = 0x149bb28, ps_ProjInfo = 0x0, scandesc = 0x7fbfa69a8308}
    (gdb) p *hjstate
    $17 = {js = {ps = {type = T_HashJoinState, plan = 0x1496d18, state = 0x149b0f8, ExecProcNode = 0x70291d <ExecHashJoin>, 
          ExecProcNodeReal = 0x70291d <ExecHashJoin>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
          qual = 0x0, lefttree = 0x149ba10, righttree = 0x149c2b8, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
          ps_ResultTupleSlot = 0x14a7498, ps_ExprContext = 0x149b950, ps_ProjInfo = 0x149cef0, scandesc = 0x0}, 
        jointype = JOIN_INNER, single_match = true, joinqual = 0x0}, hashclauses = 0x14a7b30, hj_OuterHashKeys = 0x14a8930, 
      hj_InnerHashKeys = 0x14a8e40, hj_HashOperators = 0x14a8ea0, hj_HashTable = 0x14acde8, hj_CurHashValue = 0, 
      hj_CurBucketNo = 0, hj_CurSkewBucketNo = -1, hj_CurTuple = 0x0, hj_OuterTupleSlot = 0x14a79f0, 
      hj_HashTupleSlot = 0x149cc18, hj_NullOuterTupleSlot = 0x0, hj_NullInnerTupleSlot = 0x0, 
      hj_FirstOuterTupleSlot = 0x149bbe8, hj_JoinState = 2, hj_MatchedOuter = false, hj_OuterNotEmpty = false}
    (gdb) p *hashvalue
    $18 = 32703
    (gdb) 
    

ExecHashJoinOuterGetTuple->只有一个批次,批次号为0

    
    
    (gdb) n
    808     int         curbatch = hashtable->curbatch;
    (gdb) 
    811     if (curbatch == 0)          /* if it is the first pass */
    (gdb) p curbatch
    $20 = 0
    

ExecHashJoinOuterGetTuple->获取首个outer tuple
slot(不为NULL),重置hjstate->hj_FirstOuterTupleSlot为NULL

    
    
    (gdb) n
    817         slot = hjstate->hj_FirstOuterTupleSlot;
    (gdb) 
    818         if (!TupIsNull(slot))
    (gdb) p *slot
    $21 = {type = T_TupleTableSlot, tts_isempty = false, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, 
      tts_tuple = 0x14ac200, tts_tupleDescriptor = 0x7fbfa69a8308, tts_mcxt = 0x149afe0, tts_buffer = 345, tts_nvalid = 0, 
      tts_values = 0x149bc48, tts_isnull = 0x149bc70, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = true}
    (gdb) 
    (gdb) n
    819             hjstate->hj_FirstOuterTupleSlot = NULL;
    (gdb) 
    

ExecHashJoinOuterGetTuple->循环获取,找到匹配的slot

    
    
    (gdb) 
    823         while (!TupIsNull(slot))
    (gdb) n
    828             ExprContext *econtext = hjstate->js.ps.ps_ExprContext;
    (gdb) 
    

ExecHashJoinOuterGetTuple->成功匹配,返回slot

    
    
    (gdb) n
    830             econtext->ecxt_outertuple = slot;
    (gdb) 
    834                                      HJ_FILL_OUTER(hjstate),
    (gdb) 
    831             if (ExecHashGetHashValue(hashtable, econtext,
    (gdb) 
    838                 hjstate->hj_OuterNotEmpty = true;
    (gdb) 
    840                 return slot;
    (gdb) p *slot
    $22 = {type = T_TupleTableSlot, tts_isempty = false, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = true, 
      tts_tuple = 0x14ac200, tts_tupleDescriptor = 0x7fbfa69a8308, tts_mcxt = 0x149afe0, tts_buffer = 345, tts_nvalid = 1, 
      tts_values = 0x149bc48, tts_isnull = 0x149bc70, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x0}, tts_off = 2, tts_fixedTupleDescriptor = true}
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

