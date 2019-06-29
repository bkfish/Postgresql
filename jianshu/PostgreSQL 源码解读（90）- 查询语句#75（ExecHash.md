本节介绍了ExecProcNode的其中一个Real函数(ExecHashJoin)。ExecHashJoin函数实现了Hash Join算法。

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
    
    

### 二、源码解读

ExecHashJoin函数实现了Hash Join算法,实际实现的函数是ExecHashJoinImpl.  
ExecHashJoinImpl函数把Hash
Join划分为多个阶段/状态(有限状态机),保存在HashJoinState->hj_JoinState字段中,这些状态分别是分别为HJ_BUILD_HASHTABLE/HJ_NEED_NEW_OUTER/HJ_SCAN_BUCKET/HJ_FILL_OUTER_TUPLE/HJ_FILL_INNER_TUPLES/HJ_NEED_NEW_BATCH.  
HJ_BUILD_HASHTABLE:创建Hash表;  
HJ_NEED_NEW_OUTER:扫描outer relation,计算外表连接键的hash值,把相匹配元组放在合适的bucket中;  
HJ_SCAN_BUCKET:扫描bucket,匹配的tuple返回  
HJ_FILL_OUTER_TUPLE:当前outer relation元组已耗尽，因此检查是否发出一个虚拟的外连接元组。  
HJ_FILL_INNER_TUPLES:已完成一个批处理，但做的是右外连接/完全连接,填充虚拟连接元组  
HJ_NEED_NEW_BATCH:开启下一批次  
**注意** :在work_mem不足以装下Hash Table时,分批执行.每个批次执行时,会把outer relation与inner
relation匹配(指hash值一样)的tuple会存储起来,放在合适的批次文件中(hashtable->outerBatchFile[batchno]),以避免多次的outer
relation扫描.

    
    
    #define HJ_FILL_INNER(hjstate)  ((hjstate)->hj_NullOuterTupleSlot != NULL)
    
    /* ----------------------------------------------------------------     *      ExecHashJoin
     *
     *      Parallel-oblivious version.
     *      Parallel-oblivious版本。
     * ----------------------------------------------------------------     */
    static TupleTableSlot *         /* 返回元组或者NULL;return: a tuple or NULL */
    ExecHashJoin(PlanState *pstate)
    {
        /*
         * On sufficiently smart compilers this should be inlined with the
         * parallel-aware branches removed.
         * 在足够智能的编译器上，应该内联删除并行感知分支。
         */
        return ExecHashJoinImpl(pstate, false);
    }
    
    /*
     * States of the ExecHashJoin state machine
     */
    #define HJ_BUILD_HASHTABLE      1
    #define HJ_NEED_NEW_OUTER       2
    #define HJ_SCAN_BUCKET          3
    #define HJ_FILL_OUTER_TUPLE     4
    #define HJ_FILL_INNER_TUPLES    5
    #define HJ_NEED_NEW_BATCH       6
    
    /* Returns true if doing null-fill on outer relation */
    #define HJ_FILL_OUTER(hjstate)  ((hjstate)->hj_NullInnerTupleSlot != NULL)
    /* Returns true if doing null-fill on inner relation */
    #define HJ_FILL_INNER(hjstate)  ((hjstate)->hj_NullOuterTupleSlot != NULL)
    
    static TupleTableSlot *ExecHashJoinOuterGetTuple(PlanState *outerNode,
                              HashJoinState *hjstate,
                              uint32 *hashvalue);
    static TupleTableSlot *ExecParallelHashJoinOuterGetTuple(PlanState *outerNode,
                                      HashJoinState *hjstate,
                                      uint32 *hashvalue);
    static TupleTableSlot *ExecHashJoinGetSavedTuple(HashJoinState *hjstate,
                              BufFile *file,
                              uint32 *hashvalue,
                              TupleTableSlot *tupleSlot);
    static bool ExecHashJoinNewBatch(HashJoinState *hjstate);
    static bool ExecParallelHashJoinNewBatch(HashJoinState *hjstate);
    static void ExecParallelHashJoinPartitionOuter(HashJoinState *node);
    
    /* ----------------------------------------------------------------     *      ExecHashJoinImpl
     *
     *      This function implements the Hybrid Hashjoin algorithm.  It is marked
     *      with an always-inline attribute so that ExecHashJoin() and
     *      ExecParallelHashJoin() can inline it.  Compilers that respect the
     *      attribute should create versions specialized for parallel == true and
     *      parallel == false with unnecessary branches removed.
     *      这个函数实现了混合Hash Join算法。
     *      它被标记为一个always-inline的属性(pg_attribute_always_inline)，
     *        以便ExecHashJoin()和ExecParallelHashJoin()可以内联它。
     *      可以识别该属性的编译器应该创建专门针对parallel == true和parallel == false的版本，去掉不必要的分支。
     *
     *      Note: the relation we build hash table on is the "inner"
     *            the other one is "outer".
     *      注意:在inner上创建hash表,另外一个参与连接的成为outer
     * ----------------------------------------------------------------     */
    static pg_attribute_always_inline TupleTableSlot *
    ExecHashJoinImpl(PlanState *pstate, bool parallel)
    {
        HashJoinState *node = castNode(HashJoinState, pstate);
        PlanState  *outerNode;
        HashState  *hashNode;
        ExprState  *joinqual;
        ExprState  *otherqual;
        ExprContext *econtext;
        HashJoinTable hashtable;
        TupleTableSlot *outerTupleSlot;
        uint32      hashvalue;
        int         batchno;
        ParallelHashJoinState *parallel_state;
    
        /*
         * get information from HashJoin node
         * 从HashJon Node中获取信息
         */
        joinqual = node->js.joinqual;
        otherqual = node->js.ps.qual;
        hashNode = (HashState *) innerPlanState(node);
        outerNode = outerPlanState(node);
        hashtable = node->hj_HashTable;
        econtext = node->js.ps.ps_ExprContext;
        parallel_state = hashNode->parallel_state;
    
        /*
         * Reset per-tuple memory context to free any expression evaluation
         * storage allocated in the previous tuple cycle.
         * 重置每个元组内存上下文以释放在前一个元组处理周期中分配的所有表达式计算存储。
         */
        ResetExprContext(econtext);
    
        /*
         * run the hash join state machine
         * 执行hash join状态机
         */
        for (;;)
        {
            /*
             * It's possible to iterate this loop many times before returning a
             * tuple, in some pathological cases such as needing to move much of
             * the current batch to a later batch.  So let's check for interrupts
             * each time through.
             * 在返回元组之前，可以多次迭代此循环，在某些"变态"的情况下，
             *   例如需要将当前批处理的大部分转移到下一批处理。
             * 所以需要每次检查中断。
             */
            CHECK_FOR_INTERRUPTS();
    
            switch (node->hj_JoinState)
            {
                case HJ_BUILD_HASHTABLE://-->HJ_BUILD_HASHTABLE阶段
    
                    /*
                     * First time through: build hash table for inner relation.
                     * 第一次的处理逻辑:为inner relation建立hash表
                     */
                    Assert(hashtable == NULL);
    
                    /*
                     * If the outer relation is completely empty, and it's not
                     * right/full join, we can quit without building the hash
                     * table.  However, for an inner join it is only a win to
                     * check this when the outer relation's startup cost is less
                     * than the projected cost of building the hash table.
                     * Otherwise it's best to build the hash table first and see
                     * if the inner relation is empty.  (When it's a left join, we
                     * should always make this check, since we aren't going to be
                     * able to skip the join on the strength of an empty inner
                     * relation anyway.)
                     * 如果外部关系是空的，并且它不是右外/完全连接，可以在不构建哈希表的情况下退出。
                     * 但是，对于内连接，只有当外部关系的启动成本小于构建哈希表的预期成本时，才需要检查这一点。
                     * 否则，最好先构建哈希表，看看内部关系是否为空。
                    * (当它是左外连接时，应该始终进行检查，因为无论如何，都不能基于空的内部关系跳过连接。)
                     *
                     * If we are rescanning the join, we make use of information
                     * gained on the previous scan: don't bother to try the
                     * prefetch if the previous scan found the outer relation
                     * nonempty. This is not 100% reliable since with new
                     * parameters the outer relation might yield different
                     * results, but it's a good heuristic.
                     * 如果需要重新扫描连接，将利用上次扫描结果中获得的信息:
                     *   如果上次扫描发现外部关系非空，则不必尝试预取。
                     * 但这不是100%可靠的，因为有了新的参数，外部关系可能会产生不同的结果，但这是一个很好的启发式。
                     *
                     * The only way to make the check is to try to fetch a tuple
                     * from the outer plan node.  If we succeed, we have to stash
                     * it away for later consumption by ExecHashJoinOuterGetTuple.
                     * 进行检查的唯一方法是从外部plan节点获取一个元组。
                     * 如果成功了，就必须通过ExecHashJoinOuterGetTuple将其存储起来，以便以后使用。
                     */
                    if (HJ_FILL_INNER(node))
                    {
                        /* no chance to not build the hash table */
                        //不构建哈希表是不可能的了
                        node->hj_FirstOuterTupleSlot = NULL;
                    }
                    else if (parallel)
                    {
                        /*
                         * The empty-outer optimization is not implemented for
                         * shared hash tables, because no one participant can
                         * determine that there are no outer tuples, and it's not
                         * yet clear that it's worth the synchronization overhead
                         * of reaching consensus to figure that out.  So we have
                         * to build the hash table.
                         * 对于共享哈希表，并没有实现空外关系优化，因为没有任何参与者可以确定没有外部元组，
                         * 而且还不清楚是否值得为了解决这个问题而进行同步开销。
                         * 所以我们要建立哈希表。
                         */
                        node->hj_FirstOuterTupleSlot = NULL;
                    }
                    else if (HJ_FILL_OUTER(node) ||
                             (outerNode->plan->startup_cost < hashNode->ps.plan->total_cost &&
                              !node->hj_OuterNotEmpty))
                    {
                        node->hj_FirstOuterTupleSlot = ExecProcNode(outerNode);
                        if (TupIsNull(node->hj_FirstOuterTupleSlot))
                        {
                            node->hj_OuterNotEmpty = false;
                            return NULL;
                        }
                        else
                            node->hj_OuterNotEmpty = true;
                    }
                    else
                        node->hj_FirstOuterTupleSlot = NULL;
    
                    /*
                     * Create the hash table.  If using Parallel Hash, then
                     * whoever gets here first will create the hash table and any
                     * later arrivals will merely attach to it.
                     * 创建哈希表。
                     * 如果使用并行哈希，那么最先到达这里的worker将创建哈希表，之后到达的只会附加到它上面。
                     */
                    hashtable = ExecHashTableCreate(hashNode,
                                                    node->hj_HashOperators,
                                                    HJ_FILL_INNER(node));
                    node->hj_HashTable = hashtable;
    
                    /*
                     * Execute the Hash node, to build the hash table.  If using
                     * Parallel Hash, then we'll try to help hashing unless we
                     * arrived too late.
                     * 执行哈希节点，以构建哈希表。
                     * 如果使用并行哈希，那么将尝试协助哈希运算，除非太晚了。
                     */
                    hashNode->hashtable = hashtable;
                    (void) MultiExecProcNode((PlanState *) hashNode);
    
                    /*
                     * If the inner relation is completely empty, and we're not
                     * doing a left outer join, we can quit without scanning the
                     * outer relation.
                     * 如果内部关系是空的，并且没有执行左外连接，可以在不扫描外部关系的情况下退出。
                     */
                    if (hashtable->totalTuples == 0 && !HJ_FILL_OUTER(node))
                        return NULL;
    
                    /*
                     * need to remember whether nbatch has increased since we
                     * began scanning the outer relation
                     * 需要记住自开始扫描外部关系以来nbatch是否增加了
                     */
                    hashtable->nbatch_outstart = hashtable->nbatch;
    
                    /*
                     * Reset OuterNotEmpty for scan.  (It's OK if we fetched a
                     * tuple above, because ExecHashJoinOuterGetTuple will
                     * immediately set it again.)
                     * 扫描前重置OuterNotEmpty。
                     * (在其上获取一个tuple是可以的，因为ExecHashJoinOuterGetTuple将立即再次设置它。)
                     */
                    node->hj_OuterNotEmpty = false;//重置OuterNotEmpty为F
    
                    if (parallel)
                    {
                        //启用并行
                        Barrier    *build_barrier;
    
                        build_barrier = &parallel_state->build_barrier;
                        Assert(BarrierPhase(build_barrier) == PHJ_BUILD_HASHING_OUTER ||
                               BarrierPhase(build_barrier) == PHJ_BUILD_DONE);
                        if (BarrierPhase(build_barrier) == PHJ_BUILD_HASHING_OUTER)
                        {
                            /*
                             * If multi-batch, we need to hash the outer relation
                             * up front.
                             * 如果是多批处理，需要预先散列外部关系。
                             */
                            if (hashtable->nbatch > 1)
                                ExecParallelHashJoinPartitionOuter(node);
                            BarrierArriveAndWait(build_barrier,
                                                 WAIT_EVENT_HASH_BUILD_HASHING_OUTER);
                        }
                        Assert(BarrierPhase(build_barrier) == PHJ_BUILD_DONE);
    
                        /* Each backend should now select a batch to work on. */
                        //每一个后台worker需选择批次
                        hashtable->curbatch = -1;
                        node->hj_JoinState = HJ_NEED_NEW_BATCH;
    
                        continue;//下一循环
                    }
                    else
                        //非并行执行,设置hj_JoinState状态
                        node->hj_JoinState = HJ_NEED_NEW_OUTER;
    
                    /* FALL THRU */
    
                case HJ_NEED_NEW_OUTER://-->HJ_NEED_NEW_OUTER阶段
    
                    /*
                     * We don't have an outer tuple, try to get the next one
                     * 没有外部元组，试着获取下一个
                     */
                    if (parallel)
                        outerTupleSlot =
                            ExecParallelHashJoinOuterGetTuple(outerNode, node,
                                                              &hashvalue);//并行执行
                    else
                        outerTupleSlot =
                            ExecHashJoinOuterGetTuple(outerNode, node, &hashvalue);//普通执行
    
                    if (TupIsNull(outerTupleSlot))
                    {
                        //如outerTupleSlot为NULL
                        /* end of batch, or maybe whole join */
                        //完成此批数据处理,或者可能是全连接
                        if (HJ_FILL_INNER(node))//hj_NullOuterTupleSlot != NULL
                        {
                            /* set up to scan for unmatched inner tuples */
                            //不匹配的行,填充NULL(外连接)
                            ExecPrepHashTableForUnmatched(node);
                            node->hj_JoinState = HJ_FILL_INNER_TUPLES;
                        }
                        else
                            node->hj_JoinState = HJ_NEED_NEW_BATCH;//需要下一个批次
                        continue;
                    }
                    //设置变量
                    econtext->ecxt_outertuple = outerTupleSlot;
                    node->hj_MatchedOuter = false;
    
                    /*
                     * Find the corresponding bucket for this tuple in the main
                     * hash table or skew hash table.
                     * 在主哈希表或斜哈希表中为这个元组找到对应的bucket。
                     */
                    node->hj_CurHashValue = hashvalue;
                    //获取Hash Bucket并处理此批次
                    ExecHashGetBucketAndBatch(hashtable, hashvalue,
                                              &node->hj_CurBucketNo, &batchno);
                    //Hash倾斜优化(某个值的数据特别多)
                    node->hj_CurSkewBucketNo = ExecHashGetSkewBucket(hashtable,
                                                                     hashvalue);
                    node->hj_CurTuple = NULL;
    
                    /*
                     * The tuple might not belong to the current batch (where
                     * "current batch" includes the skew buckets if any).
                     * 元组可能不属于当前批处理(其中“当前批处理”包括倾斜桶-如果有的话)。
                     */
                    if (batchno != hashtable->curbatch &&
                        node->hj_CurSkewBucketNo == INVALID_SKEW_BUCKET_NO)
                    {
                        /*
                         * Need to postpone this outer tuple to a later batch.
                         * Save it in the corresponding outer-batch file.
                         * 需要将这个外部元组推迟到稍后的批处理。保存在相应的外部批处理文件中。
                         * 也就是说,INNER和OUTER属于此批次的数据都可能存储在外存中
                         */
                        Assert(parallel_state == NULL);
                        Assert(batchno > hashtable->curbatch);
                        ExecHashJoinSaveTuple(ExecFetchSlotMinimalTuple(outerTupleSlot),
                                              hashvalue,
                                              &hashtable->outerBatchFile[batchno]);
    
                        /* Loop around, staying in HJ_NEED_NEW_OUTER state */
                        //循环，保持HJ_NEED_NEW_OUTER状态
                        continue;
                    }
    
                    /* OK, let's scan the bucket for matches */
                    //已完成此阶段,切换至HJ_SCAN_BUCKET状态
                    node->hj_JoinState = HJ_SCAN_BUCKET;
    
                    /* FALL THRU */
    
                case HJ_SCAN_BUCKET://-->HJ_SCAN_BUCKET阶段
    
                    /*
                     * Scan the selected hash bucket for matches to current outer
                     * 扫描选定的散列桶，查找与当前外部匹配的散列桶
                     */
                    if (parallel)
                    {
                        //并行处理
                        if (!ExecParallelScanHashBucket(node, econtext))
                        {
                            /* out of matches; check for possible outer-join fill */
                            // 无法匹配,检查可能的外连接填充,状态切换为HJ_FILL_OUTER_TUPLE
                            node->hj_JoinState = HJ_FILL_OUTER_TUPLE;
                            continue;
                        }
                    }
                    else
                    {
                        //非并行执行
                        if (!ExecScanHashBucket(node, econtext))
                        {
                            /* out of matches; check for possible outer-join fill */
                            node->hj_JoinState = HJ_FILL_OUTER_TUPLE;//同上
                            continue;
                        }
                    }
    
                    /*
                     * We've got a match, but still need to test non-hashed quals.
                     * ExecScanHashBucket already set up all the state needed to
                     * call ExecQual.
                     * 发现一个匹配，但仍然需要测试非散列的quals。
                     * ExecScanHashBucket已经设置了调用ExecQual所需的所有状态。
                     * 
                     * If we pass the qual, then save state for next call and have
                     * ExecProject form the projection, store it in the tuple
                     * table, and return the slot.
                     * 如果我们传递了qual，那么将状态保存为下一次调用，
                     * 并让ExecProject形成投影，将其存储在tuple table中，并返回slot。
                     *
                     * Only the joinquals determine tuple match status, but all
                     * quals must pass to actually return the tuple.
                     * 只有连接条件joinquals确定元组匹配状态，但所有条件quals必须通过才能返回元组。
                     */
                    if (joinqual == NULL || ExecQual(joinqual, econtext))
                    {
                        node->hj_MatchedOuter = true;
                        HeapTupleHeaderSetMatch(HJTUPLE_MINTUPLE(node->hj_CurTuple));
    
                        /* In an antijoin, we never return a matched tuple */
                        //反连接,则不能返回匹配的元组
                        if (node->js.jointype == JOIN_ANTI)
                        {
                            node->hj_JoinState = HJ_NEED_NEW_OUTER;
                            continue;
                        }
    
                        /*
                         * If we only need to join to the first matching inner
                         * tuple, then consider returning this one, but after that
                         * continue with next outer tuple.
                         * 如果只需要连接到第一个匹配的内表元组，那么可以考虑返回这个元组，
                         * 但是在此之后可以继续使用下一个外表元组。
                         */
                        if (node->js.single_match)
                            node->hj_JoinState = HJ_NEED_NEW_OUTER;
    
                        if (otherqual == NULL || ExecQual(otherqual, econtext))
                            return ExecProject(node->js.ps.ps_ProjInfo);//执行投影操作
                        else
                            InstrCountFiltered2(node, 1);//其他条件不匹配
                    }
                    else
                        InstrCountFiltered1(node, 1);//连接条件不匹配
                    break;
    
                case HJ_FILL_OUTER_TUPLE://-->HJ_FILL_OUTER_TUPLE阶段
    
                    /*
                     * The current outer tuple has run out of matches, so check
                     * whether to emit a dummy outer-join tuple.  Whether we emit
                     * one or not, the next state is NEED_NEW_OUTER.
                     * 当前外部元组已耗尽匹配项，因此检查是否发出一个虚拟的外连接元组。
                     * 不管是否发出一个，下一个状态是NEED_NEW_OUTER。
                     */
                    node->hj_JoinState = HJ_NEED_NEW_OUTER;//切换状态为HJ_NEED_NEW_OUTER
    
                    if (!node->hj_MatchedOuter &&
                        HJ_FILL_OUTER(node))
                    {
                        /*
                         * Generate a fake join tuple with nulls for the inner
                         * tuple, and return it if it passes the non-join quals.
                         * 为内部元组生成一个带有null的假连接元组，并在满足非连接条件quals时返回它。
                         */
                        econtext->ecxt_innertuple = node->hj_NullInnerTupleSlot;
    
                        if (otherqual == NULL || ExecQual(otherqual, econtext))
                            return ExecProject(node->js.ps.ps_ProjInfo);//投影操作
                        else
                            InstrCountFiltered2(node, 1);
                    }
                    break;
    
                case HJ_FILL_INNER_TUPLES://-->HJ_FILL_INNER_TUPLES阶段
    
                    /*
                     * We have finished a batch, but we are doing right/full join,
                     * so any unmatched inner tuples in the hashtable have to be
                     * emitted before we continue to the next batch.
                     * 已经完成了一个批处理，但是做的是右外/完全连接，
                         所以必须在继续下一个批处理之前发出散列表中任何不匹配的内部元组。
                     */
                    if (!ExecScanHashTableForUnmatched(node, econtext))
                    {
                        /* no more unmatched tuples */
                        //不存在更多不匹配的元组,切换状态为HJ_NEED_NEW_BATCH(开始下一批次)
                        node->hj_JoinState = HJ_NEED_NEW_BATCH;
                        continue;
                    }
    
                    /*
                     * Generate a fake join tuple with nulls for the outer tuple,
                     * and return it if it passes the non-join quals.
                     * 为外表元组生成一个带有null的假连接元组，并在满足非连接条件quals时返回它。
                     */
                    econtext->ecxt_outertuple = node->hj_NullOuterTupleSlot;
    
                    if (otherqual == NULL || ExecQual(otherqual, econtext))
                        return ExecProject(node->js.ps.ps_ProjInfo);
                    else
                        InstrCountFiltered2(node, 1);
                    break;
    
                case HJ_NEED_NEW_BATCH://-->HJ_NEED_NEW_BATCH阶段
    
                    /*
                     * Try to advance to next batch.  Done if there are no more.
                     * 尽量提前到下一批。如果没有了，就结束。
                     */
                    if (parallel)
                    {
                        //并行处理
                        if (!ExecParallelHashJoinNewBatch(node))
                            return NULL;    /* end of parallel-aware join */
                    }
                    else
                    {
                        //非并行处理
                        if (!ExecHashJoinNewBatch(node))
                            return NULL;    /* end of parallel-oblivious join */
                    }
                    node->hj_JoinState = HJ_NEED_NEW_OUTER;//切换状态
                    break;
    
                default://非法的JoinState
                    elog(ERROR, "unrecognized hashjoin state: %d",
                         (int) node->hj_JoinState);
            }
        }
    }
    
    

### 三、跟踪分析

测试脚本如下

    
    
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
    

启动gdb,设置断点,进入ExecHashJoin

    
    
    (gdb) b ExecHashJoin
    Breakpoint 1 at 0x70292e: file nodeHashjoin.c, line 565.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecHashJoin (pstate=0x2ee1a88) at nodeHashjoin.c:565
    565     return ExecHashJoinImpl(pstate, false);
    

继续执行,进入第2个Hash Join,即t_grxx & t_dwxx的连接

    
    
    (gdb) n
    
    Breakpoint 1, ExecHashJoin (pstate=0x2ee1d98) at nodeHashjoin.c:565
    565     return ExecHashJoinImpl(pstate, false);
    

查看输入参数,ExecProcNode=ExecProcNodeReal=ExecHashJoin

    
    
    (gdb) p *pstate
    $8 = {type = T_HashJoinState, plan = 0x2faaff8, state = 0x2ee1758, ExecProcNode = 0x70291d <ExecHashJoin>, 
      ExecProcNodeReal = 0x70291d <ExecHashJoin>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
      qual = 0x0, lefttree = 0x2ee2070, righttree = 0x2ee2918, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
      ps_ResultTupleSlot = 0x2f20d98, ps_ExprContext = 0x2ee1fb0, ps_ProjInfo = 0x2ee3550, scandesc = 0x0}
    (gdb) 
    

pstate的lefttree对应的是SeqScan,righttree对应的是Hash,即左树(outer
relation)为t_grxx的顺序扫描运算生成的relation,右树(inner
relation)为t_dwxx的顺序扫描运算生成的relation(在此relation上创建Hash Table)

    
    
    (gdb) p *pstate->lefttree
    $6 = {type = T_SeqScanState, plan = 0x2fa8ff0, state = 0x2ee1758, ExecProcNode = 0x6e4bde <ExecProcNodeFirst>, 
      ExecProcNodeReal = 0x71578d <ExecSeqScan>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
      qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
      ps_ResultTupleSlot = 0x2ee27d8, ps_ExprContext = 0x2ee2188, ps_ProjInfo = 0x0, scandesc = 0x7f0710d02bd0}
    (gdb) p *pstate->righttree
    $9 = {type = T_HashState, plan = 0x2faaf60, state = 0x2ee1758, ExecProcNode = 0x6e4bde <ExecProcNodeFirst>, 
      ExecProcNodeReal = 0x6fc015 <ExecHash>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
      qual = 0x0, lefttree = 0x2ee2af0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
      ps_ResultTupleSlot = 0x2ee3278, ps_ExprContext = 0x2ee2a30, ps_ProjInfo = 0x0, scandesc = 0x0}
    

进入ExecHashJoinImpl函数

    
    
    (gdb) step
    ExecHashJoinImpl (pstate=0x2ee1d98, parallel=false) at nodeHashjoin.c:167
    167     HashJoinState *node = castNode(HashJoinState, pstate);
    

赋值,查看HashJoinState等变量值

    
    
    (gdb) n
    182     joinqual = node->js.joinqual;
    (gdb) n
    183     otherqual = node->js.ps.qual;
    (gdb) 
    184     hashNode = (HashState *) innerPlanState(node);
    (gdb) 
    185     outerNode = outerPlanState(node);
    (gdb) 
    186     hashtable = node->hj_HashTable;
    (gdb) 
    187     econtext = node->js.ps.ps_ExprContext;
    (gdb) 
    188     parallel_state = hashNode->parallel_state;
    (gdb) 
    194     ResetExprContext(econtext);
    (gdb) p *node
    $10 = {js = {ps = {type = T_HashJoinState, plan = 0x2faaff8, state = 0x2ee1758, ExecProcNode = 0x70291d <ExecHashJoin>, 
          ExecProcNodeReal = 0x70291d <ExecHashJoin>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
          qual = 0x0, lefttree = 0x2ee2070, righttree = 0x2ee2918, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
          ps_ResultTupleSlot = 0x2f20d98, ps_ExprContext = 0x2ee1fb0, ps_ProjInfo = 0x2ee3550, scandesc = 0x0}, 
        jointype = JOIN_INNER, single_match = true, joinqual = 0x0}, hashclauses = 0x2f21430, hj_OuterHashKeys = 0x2f22230, 
      hj_InnerHashKeys = 0x2f22740, hj_HashOperators = 0x2f227a0, hj_HashTable = 0x0, hj_CurHashValue = 0, hj_CurBucketNo = 0, 
      hj_CurSkewBucketNo = -1, hj_CurTuple = 0x0, hj_OuterTupleSlot = 0x2f212f0, hj_HashTupleSlot = 0x2ee3278, 
      hj_NullOuterTupleSlot = 0x0, hj_NullInnerTupleSlot = 0x0, hj_FirstOuterTupleSlot = 0x0, hj_JoinState = 1, 
      hj_MatchedOuter = false, hj_OuterNotEmpty = false}
    (gdb) p *otherqual
    Cannot access memory at address 0x0
    (gdb) p *hashNode
    $11 = {ps = {type = T_HashState, plan = 0x2faaf60, state = 0x2ee1758, ExecProcNode = 0x6e4bde <ExecProcNodeFirst>, 
        ExecProcNodeReal = 0x6fc015 <ExecHash>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
        qual = 0x0, lefttree = 0x2ee2af0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
        ps_ResultTupleSlot = 0x2ee3278, ps_ExprContext = 0x2ee2a30, ps_ProjInfo = 0x0, scandesc = 0x0}, hashtable = 0x0, 
      hashkeys = 0x2f22740, shared_info = 0x0, hinstrument = 0x0, parallel_state = 0x0}
    (gdb) p *hashtable
    Cannot access memory at address 0x0
    (gdb) p parallel_state
    $12 = (ParallelHashJoinState *) 0x0
    (gdb)   
    

进入HJ_BUILD_HASHTABLE处理逻辑,创建Hash表

    
    
    (gdb) p node->hj_JoinState
    $13 = 1
    

HJ_BUILD_HASHTABLE->执行相关判断,本例为内连接,因此不存在FILL_OUTER等情况

    
    
    (gdb) n
    216                 Assert(hashtable == NULL);
    (gdb) 
    241                 if (HJ_FILL_INNER(node))
    (gdb) 
    246                 else if (parallel)
    (gdb) 
    258                 else if (HJ_FILL_OUTER(node) ||
    (gdb) 
    259                          (outerNode->plan->startup_cost < hashNode->ps.plan->total_cost &&
    (gdb) 
    

HJ_BUILD_HASHTABLE->outer node的启动成本低于创建Hash表的总成本而且outer
relation为空(初始化node->hj_OuterNotEmpty为false),那么尝试获取outer
relation的第一个元组,如为NULL,则可快速返回NULL,否则设置node->hj_OuterNotEmpty标记为T

    
    
    258                 else if (HJ_FILL_OUTER(node) ||
    (gdb) 
    260                           !node->hj_OuterNotEmpty))
    (gdb) 
    259                          (outerNode->plan->startup_cost < hashNode->ps.plan->total_cost &&
    (gdb) 
    262                     node->hj_FirstOuterTupleSlot = ExecProcNode(outerNode);
    (gdb) 
    263                     if (TupIsNull(node->hj_FirstOuterTupleSlot))
    (gdb) 
    269                         node->hj_OuterNotEmpty = true;
    

HJ_BUILD_HASHTABLE->创建Hash Table

    
    
    (gdb) n
    263                     if (TupIsNull(node->hj_FirstOuterTupleSlot))
    (gdb) 
    281                                                 HJ_FILL_INNER(node));
    (gdb) 
    279                 hashtable = ExecHashTableCreate(hashNode,
    (gdb) 
    

HJ_BUILD_HASHTABLE->Hash Table(HashJoinTable结构体)的内存结构  
bucket数量为16384(16K),取对数结果为14(即log2_nbuckets/log2_nbuckets_optimal的结果值)  
skewEnabled为F,没有启用倾斜优化

    
    
    (gdb) p *hashtable
    $14 = {nbuckets = 16384, log2_nbuckets = 14, nbuckets_original = 16384, nbuckets_optimal = 16384, 
      log2_nbuckets_optimal = 14, buckets = {unshared = 0x2fb1260, shared = 0x2fb1260}, keepNulls = false, skewEnabled = false, 
      skewBucket = 0x0, skewBucketLen = 0, nSkewBuckets = 0, skewBucketNums = 0x0, nbatch = 1, curbatch = 0, 
      nbatch_original = 1, nbatch_outstart = 1, growEnabled = true, totalTuples = 0, partialTuples = 0, skewTuples = 0, 
      innerBatchFile = 0x0, outerBatchFile = 0x0, outer_hashfunctions = 0x3053b68, inner_hashfunctions = 0x3053bc0, 
      hashStrict = 0x3053c18, spaceUsed = 0, spaceAllowed = 16777216, spacePeak = 0, spaceUsedSkew = 0, 
      spaceAllowedSkew = 335544, hashCxt = 0x3053a50, batchCxt = 0x2f8b170, chunks = 0x0, current_chunk = 0x0, area = 0x0, 
      parallel_state = 0x0, batches = 0x0, current_chunk_shared = 9187201950435737471}
    

HJ_BUILD_HASHTABLE->使用的Hash函数

    
    
    (gdb) p *hashtable->inner_hashfunctions
    $15 = {fn_addr = 0x4c8a0a <hashtext>, fn_oid = 400, fn_nargs = 1, fn_strict = true, fn_retset = false, fn_stats = 2 '\002', 
      fn_extra = 0x0, fn_mcxt = 0x3053a50, fn_expr = 0x0}
    (gdb) p *hashtable->outer_hashfunctions
    $16 = {fn_addr = 0x4c8a0a <hashtext>, fn_oid = 400, fn_nargs = 1, fn_strict = true, fn_retset = false, fn_stats = 2 '\002', 
      fn_extra = 0x0, fn_mcxt = 0x3053a50, fn_expr = 0x0}
    

HJ_BUILD_HASHTABLE->赋值,并执行此Hash Node节点,结果总元组数为10000

    
    
    (gdb) n
    289                 hashNode->hashtable = hashtable;
    (gdb) 
    290                 (void) MultiExecProcNode((PlanState *) hashNode);
    (gdb) 
    297                 if (hashtable->totalTuples == 0 && !HJ_FILL_OUTER(node))
    (gdb) p hashtable->totalTuples 
    $18 = 10000
    

HJ_BUILD_HASHTABLE->批次数为1,只需要执行1个批次即可

    
    
    (gdb) n
    304                 hashtable->nbatch_outstart = hashtable->nbatch;
    (gdb) p hashtable->nbatch
    $19 = 1
    

HJ_BUILD_HASHTABLE->重置OuterNotEmpty为F

    
    
    (gdb) n
    311                 node->hj_OuterNotEmpty = false;
    (gdb) 
    313                 if (parallel)
    

HJ_BUILD_HASHTABLE->非并行执行,切换状态为HJ_NEED_NEW_OUTER

    
    
    (gdb) 
    313                 if (parallel)
    (gdb) n
    340                     node->hj_JoinState = HJ_NEED_NEW_OUTER;
    

HJ_NEED_NEW_OUTER->获取(执行ExecHashJoinOuterGetTuple)下一个outer relation的一个元组

    
    
    349                 if (parallel)
    (gdb) n
    354                     outerTupleSlot =
    (gdb) 
    357                 if (TupIsNull(outerTupleSlot))
    (gdb) p *outerTupleSlot
    $20 = {type = T_TupleTableSlot, tts_isempty = false, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = true, 
      tts_tuple = 0x2f88300, tts_tupleDescriptor = 0x7f0710d02bd0, tts_mcxt = 0x2ee1640, tts_buffer = 507, tts_nvalid = 1, 
      tts_values = 0x2ee22a8, tts_isnull = 0x2ee22d0, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x0}, tts_off = 2, tts_fixedTupleDescriptor = true}
    

HJ_NEED_NEW_OUTER->设置相关变量

    
    
    (gdb) n
    371                 econtext->ecxt_outertuple = outerTupleSlot;
    (gdb) 
    372                 node->hj_MatchedOuter = false;
    (gdb) 
    378                 node->hj_CurHashValue = hashvalue;
    (gdb) 
    379                 ExecHashGetBucketAndBatch(hashtable, hashvalue,
    (gdb) p hashvalue
    $21 = 2324234220
    (gdb) n
    381                 node->hj_CurSkewBucketNo = ExecHashGetSkewBucket(hashtable,
    (gdb) 
    383                 node->hj_CurTuple = NULL;
    (gdb) p *node
    $22 = {js = {ps = {type = T_HashJoinState, plan = 0x2faaff8, state = 0x2ee1758, ExecProcNode = 0x70291d <ExecHashJoin>, 
          ExecProcNodeReal = 0x70291d <ExecHashJoin>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
          qual = 0x0, lefttree = 0x2ee2070, righttree = 0x2ee2918, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
          ps_ResultTupleSlot = 0x2f20d98, ps_ExprContext = 0x2ee1fb0, ps_ProjInfo = 0x2ee3550, scandesc = 0x0}, 
        jointype = JOIN_INNER, single_match = true, joinqual = 0x0}, hashclauses = 0x2f21430, hj_OuterHashKeys = 0x2f22230, 
      hj_InnerHashKeys = 0x2f22740, hj_HashOperators = 0x2f227a0, hj_HashTable = 0x2f88ee8, hj_CurHashValue = 2324234220, 
      hj_CurBucketNo = 16364, hj_CurSkewBucketNo = -1, hj_CurTuple = 0x0, hj_OuterTupleSlot = 0x2f212f0, 
      hj_HashTupleSlot = 0x2ee3278, hj_NullOuterTupleSlot = 0x0, hj_NullInnerTupleSlot = 0x0, hj_FirstOuterTupleSlot = 0x0, 
      hj_JoinState = 2, hj_MatchedOuter = false, hj_OuterNotEmpty = true}
    (gdb) p *econtext
    $25 = {type = T_ExprContext, ecxt_scantuple = 0x0, ecxt_innertuple = 0x0, ecxt_outertuple = 0x2ee2248, 
      ecxt_per_query_memory = 0x2ee1640, ecxt_per_tuple_memory = 0x2f710c0, ecxt_param_exec_vals = 0x0, 
      ecxt_param_list_info = 0x0, ecxt_aggvalues = 0x0, ecxt_aggnulls = 0x0, caseValue_datum = 0, caseValue_isNull = true, 
      domainValue_datum = 0, domainValue_isNull = true, ecxt_estate = 0x2ee1758, ecxt_callbacks = 0x0}
    (gdb) p *node->hj_HashTupleSlot
    $26 = {type = T_TupleTableSlot, tts_isempty = true, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, 
      tts_tuple = 0x0, tts_tupleDescriptor = 0x2ee3060, tts_mcxt = 0x2ee1640, tts_buffer = 0, tts_nvalid = 0, 
      tts_values = 0x2ee32d8, tts_isnull = 0x2ee32f0, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = true}  
    

HJ_NEED_NEW_OUTER->切换状态为HJ_SCAN_BUCKET,开始扫描Hash Table

    
    
    (gdb) n
    407                 node->hj_JoinState = HJ_SCAN_BUCKET;
    (gdb) 
    

HJ_SCAN_BUCKET->不匹配,切换状态为HJ_FILL_OUTER_TUPLE

    
    
    (gdb) 
    416                 if (parallel)
    (gdb) n
    427                     if (!ExecScanHashBucket(node, econtext))
    (gdb) 
    430                         node->hj_JoinState = HJ_FILL_OUTER_TUPLE;
    (gdb) 
    431                         continue;
    (gdb) 
    

HJ_FILL_OUTER_TUPLE->切换状态为HJ_NEED_NEW_OUTER  
不管是否获得/发出一个元组，下一个状态是NEED_NEW_OUTER

    
    
    209         switch (node->hj_JoinState)
    (gdb) 
    483                 node->hj_JoinState = HJ_NEED_NEW_OUTER;
    

HJ_FILL_OUTER_TUPLE->由于不是外连接,无需FILL,回到HJ_NEED_NEW_OUTER处理逻辑

    
    
    (gdb) n
    485                 if (!node->hj_MatchedOuter &&
    (gdb) 
    486                     HJ_FILL_OUTER(node))
    (gdb) 
    485                 if (!node->hj_MatchedOuter &&
    (gdb) 
    549     }
    (gdb) 
    

HJ_SCAN_BUCKET->在SCAN_BUCKET成功扫描的位置设置断点

    
    
    (gdb) b nodeHashjoin.c:441
    Breakpoint 3 at 0x7025c3: file nodeHashjoin.c, line 441.
    (gdb) c
    Continuing.
    Breakpoint 3, ExecHashJoinImpl (pstate=0x2ee1d98, parallel=false) at nodeHashjoin.c:447
    447                 if (joinqual == NULL || ExecQual(joinqual, econtext))
    

HJ_SCAN_BUCKET->存在匹配的元组,设置相关标记

    
    
    (gdb) n
    449                     node->hj_MatchedOuter = true;
    (gdb) 
    450                     HeapTupleHeaderSetMatch(HJTUPLE_MINTUPLE(node->hj_CurTuple));
    (gdb) 
    453                     if (node->js.jointype == JOIN_ANTI)
    (gdb) n
    464                     if (node->js.single_match)
    (gdb) 
    465                         node->hj_JoinState = HJ_NEED_NEW_OUTER;
    (gdb) 
    

HJ_SCAN_BUCKET->执行投影操作并返回

    
    
    467                     if (otherqual == NULL || ExecQual(otherqual, econtext))
    (gdb) 
    468                         return ExecProject(node->js.ps.ps_ProjInfo);
    (gdb) 
    

总的来说,Hash Join的实现是创建inner relation的Hash Table,然后获取outer
relation的元组,如匹配则执行投影操作返回相应的元组,除了创建HT外,其他步骤不断的变换状态执行,直至满足Portal要求的元组数量为止.

### 四、参考资料

[Hash Joins: Past, Present and Future/PGCon
2017](https://www.pgcon.org/2017/schedule/events/1053.en.html)  
[A Look at How Postgres Executes a Tiny Join - Part
1](https://dzone.com/articles/a-look-at-how-postgres-executes-a-tiny-join-part-1)  
[A Look at How Postgres Executes a Tiny Join - Part
2](https://dzone.com/articles/a-look-at-how-postgres-executes-a-tiny-join-part-2)  
[Assignment 2 Symmetric Hash
Join](https://cs.uwaterloo.ca/~david/cs448/A2.pdf)

