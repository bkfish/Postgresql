本节介绍了APPEND Plan Node节点的初始化和执行逻辑，主要的实现函数包括ExecInitAppend和ExecAppend。

### 一、数据结构

**AppendState**  
用于Append Node执行的数据结构

    
    
    /* ----------------     *   AppendState information
     *   AppendState结构体
     *
     *      nplans              how many plans are in the array
     *                          数组大小
     *      whichplan           which plan is being executed (0 .. n-1), or a
     *                          special negative value. See nodeAppend.c.
     *                          那个计划将要被/正在被执行(0 .. n-1),或者一个特别的负数,详见nodeAppend.c
     *      pruningstate        details required to allow partitions to be
     *                          eliminated from the scan, or NULL if not possible.
     *                          在扫描过程中允许忽略分区的细节信息,入不可能则为NULL
     *      valid_subplans      for runtime pruning, valid appendplans indexes to
     *                          scan.
     *                          用于运行期裁剪分区,将要扫描的有效appendplans索引
     * ----------------     */
    
    struct AppendState;
    typedef struct AppendState AppendState;
    struct ParallelAppendState;
    typedef struct ParallelAppendState ParallelAppendState;
    struct PartitionPruneState;
    
    struct AppendState
    {
        //第一个字段为NodeTag
        PlanState   ps;             /* its first field is NodeTag */
        //PlanStates数组
        PlanState **appendplans;    /* array of PlanStates for my inputs */
        //数组大小
        int         as_nplans;
        //那个计划将要被/正在被执行
        int         as_whichplan;
        //包含第一个部分计划的appendplans数组编号
        int         as_first_partial_plan;  /* Index of 'appendplans' containing
                                             * the first partial plan */
        //并行执行的协调信息
        ParallelAppendState *as_pstate; /* parallel coordination info */
        //协调信息大小
        Size        pstate_len;     /* size of parallel coordination info */
        //分区裁剪信息
        struct PartitionPruneState *as_prune_state;
        //有效的子计划位图集
        Bitmapset  *as_valid_subplans;
        //选择下个子计划的实现函数
        bool        (*choose_next_subplan) (AppendState *);
    };
    
    

### 二、源码解读

**ExecInitAppend**  
ExecInitAppend函数为开始append node的所有子关系扫描执行相关的初始化.

    
    
    /* ----------------------------------------------------------------     *      ExecInitAppend
     *
     *      Begin all of the subscans of the append node.
     *      开始append node的所有子关系扫描
     * 
     *     (This is potentially wasteful, since the entire result of the
     *      append node may not be scanned, but this way all of the
     *      structures get allocated in the executor's top level memory
     *      block instead of that of the call to ExecAppend.)
     *     (这可能是一种浪费，因为可能不会扫描append节点的整个结果，
     *      但是通过这种方式，所有结构都将在执行程序的最顶级内存块中分配，
     *      而不是在ExecAppend调用的内存块中分配。)
     * ----------------------------------------------------------------     */
    AppendState *
    ExecInitAppend(Append *node, EState *estate, int eflags)
    {
        AppendState *appendstate = makeNode(AppendState);
        PlanState **appendplanstates;
        Bitmapset  *validsubplans;
        int         nplans;
        int         firstvalid;
        int         i,
                    j;
        ListCell   *lc;
    
        /* check for unsupported flags */
        //检查不支持的标记
        Assert(!(eflags & EXEC_FLAG_MARK));
    
        /*
         * create new AppendState for our append node
         * 为append节点创建AppendState
         */
        appendstate->ps.plan = (Plan *) node;
        appendstate->ps.state = estate;
        appendstate->ps.ExecProcNode = ExecAppend;
    
        /* Let choose_next_subplan_* function handle setting the first subplan */
        //让choose_next_subplan_* 函数处理第一个子计划的设置
        appendstate->as_whichplan = INVALID_SUBPLAN_INDEX;
    
        /* If run-time partition pruning is enabled, then set that up now */
        //如允许运行时分区裁剪,则配置分区裁剪
        if (node->part_prune_info != NULL)
        {
            PartitionPruneState *prunestate;
    
            /* We may need an expression context to evaluate partition exprs */
            //需要独立的表达式上下文来解析分区表达式
            ExecAssignExprContext(estate, &appendstate->ps);
    
            /* Create the working data structure for pruning. */
            //为裁剪创建工作数据结构信息
            prunestate = ExecCreatePartitionPruneState(&appendstate->ps,
                                                       node->part_prune_info);
            appendstate->as_prune_state = prunestate;
    
            /* Perform an initial partition prune, if required. */
            //如需要,执行初始的分区裁剪
            if (prunestate->do_initial_prune)
            {
                /* Determine which subplans survive initial pruning */
                //确定那个子计划在初始裁剪中仍"存活"
                validsubplans = ExecFindInitialMatchingSubPlans(prunestate,
                                                                list_length(node->appendplans));
    
                /*
                 * The case where no subplans survive pruning must be handled
                 * specially.  The problem here is that code in explain.c requires
                 * an Append to have at least one subplan in order for it to
                 * properly determine the Vars in that subplan's targetlist.  We
                 * sidestep this issue by just initializing the first subplan and
                 * setting as_whichplan to NO_MATCHING_SUBPLANS to indicate that
                 * we don't really need to scan any subnodes.
                 * 没有子计划"存活"的情况需要特别处理.
                 * 这里的问题是explain.c的执行逻辑至少需要一个Append节点和一个子计划,
                 *   用于确定子计划投影列链表中的Vars.
                 * 通过初始化第一个子计划以及设置as_whichplan为NO_MATCHING_SUBPLANS,
                 *   用以指示不需要扫描任何的子节点.
                 */
                if (bms_is_empty(validsubplans))
                {
                    appendstate->as_whichplan = NO_MATCHING_SUBPLANS;
    
                    /* Mark the first as valid so that it's initialized below */
                    //标记第一个子计划为有效的,以便在接下来可以初始化
                    validsubplans = bms_make_singleton(0);
                }
                //子计划数目
                nplans = bms_num_members(validsubplans);
            }
            else
            {
                /* We'll need to initialize all subplans */
                //需要初始化所有的的子计划
                nplans = list_length(node->appendplans);
                Assert(nplans > 0);
                validsubplans = bms_add_range(NULL, 0, nplans - 1);
            }
    
            /*
             * If no runtime pruning is required, we can fill as_valid_subplans
             * immediately, preventing later calls to ExecFindMatchingSubPlans.
             * 如不需要运行期裁剪,那么我们可以马上设置as_valid_subplans,
             *   以防止后续逻辑调用ExecFindMatchingSubPlans
             */
            if (!prunestate->do_exec_prune)
            {
                Assert(nplans > 0);
                appendstate->as_valid_subplans = bms_add_range(NULL, 0, nplans - 1);
            }
        }
        else
        {
            //不需要运行期裁剪
            nplans = list_length(node->appendplans);
    
            /*
             * When run-time partition pruning is not enabled we can just mark all
             * subplans as valid; they must also all be initialized.
             * 运行期裁剪不允许,设置所有的子计划为有效,同时这些子计划必须全部初始化
             */
            Assert(nplans > 0);
            appendstate->as_valid_subplans = validsubplans =
                bms_add_range(NULL, 0, nplans - 1);
            appendstate->as_prune_state = NULL;
        }
    
        /*
         * Initialize result tuple type and slot.
         * 初始化结果元组类型和slot
         */
        ExecInitResultTupleSlotTL(&appendstate->ps, &TTSOpsVirtual);
    
        /* node returns slots from each of its subnodes, therefore not fixed */
        //从每一个子节点中返回slot,但并不固定
        appendstate->ps.resultopsset = true;
        appendstate->ps.resultopsfixed = false;
    
        appendplanstates = (PlanState **) palloc(nplans *
                                                 sizeof(PlanState *));
    
        /*
         * call ExecInitNode on each of the valid plans to be executed and save
         * the results into the appendplanstates array.
         * 在每一个有效的计划上执行ExecInitNode,
         *   同时保存结果到appendplanstates数组中.
         *
         * While at it, find out the first valid partial plan.
         * 在此期间,找到第一个有效的部分计划(并行使用)
         */
        j = i = 0;
        firstvalid = nplans;
        foreach(lc, node->appendplans)
        {
            if (bms_is_member(i, validsubplans))
            {
                Plan       *initNode = (Plan *) lfirst(lc);
    
                /*
                 * Record the lowest appendplans index which is a valid partial
                 * plan.
                 * 记录有效部分计划的最小appendplans索引号
                 */
                if (i >= node->first_partial_plan && j < firstvalid)
                    firstvalid = j;
    
                appendplanstates[j++] = ExecInitNode(initNode, estate, eflags);//初始化节点
            }
            i++;
        }
        //配置Append State
        appendstate->as_first_partial_plan = firstvalid;
        appendstate->appendplans = appendplanstates;
        appendstate->as_nplans = nplans;
    
        /*
         * Miscellaneous initialization
         * 初始化
         */
    
        appendstate->ps.ps_ProjInfo = NULL;
    
        /* For parallel query, this will be overridden later. */
        //对于并行查询,该值后面会被覆盖
        appendstate->choose_next_subplan = choose_next_subplan_locally;
    
        return appendstate;
    }
    
    

**ExecAppend**  
ExecAppend在多个子计划中进行迭代处理

    
    
    /* ----------------------------------------------------------------     *     ExecAppend
     *
     *      Handles iteration over multiple subplans.
     *      在多个子计划中处理迭代
     * ----------------------------------------------------------------     */
    static TupleTableSlot *
    ExecAppend(PlanState *pstate)
    {
        AppendState *node = castNode(AppendState, pstate);
    
        if (node->as_whichplan < 0)
        {
            /*
             * If no subplan has been chosen, we must choose one before
             * proceeding.
             * 如果没有子计划选中,必须在开始前选择一个
             */
            if (node->as_whichplan == INVALID_SUBPLAN_INDEX &&
                !node->choose_next_subplan(node))
                return ExecClearTuple(node->ps.ps_ResultTupleSlot);//出错,则清除tuple,返回
    
            /* Nothing to do if there are no matching subplans */
            //如无匹配的子计划,返回
            else if (node->as_whichplan == NO_MATCHING_SUBPLANS)
                return ExecClearTuple(node->ps.ps_ResultTupleSlot);
        }
    
        for (;;)//循环获取tuple slot
        {
            PlanState  *subnode;
            TupleTableSlot *result;
    
            CHECK_FOR_INTERRUPTS();
    
            /*
             * figure out which subplan we are currently processing
             * 选择哪个子计划需要现在执行
             */
            Assert(node->as_whichplan >= 0 && node->as_whichplan < node->as_nplans);
            subnode = node->appendplans[node->as_whichplan];
    
            /*
             * get a tuple from the subplan
             * 从子计划中获取tuple
             */
            result = ExecProcNode(subnode);
    
            if (!TupIsNull(result))
            {
                /*
                 * If the subplan gave us something then return it as-is. We do
                 * NOT make use of the result slot that was set up in
                 * ExecInitAppend; there's no need for it.
                 * 如果子计划有返回,则直接返回.
                 * 在这里,不需要在ExecInitAppend中构造结果slot,因为不需要这样做.
                 */
                return result;
            }
    
            /* choose new subplan; if none, we're done */
            //选择新的子计划,如无,则完成调用
            if (!node->choose_next_subplan(node))
                return ExecClearTuple(node->ps.ps_ResultTupleSlot);
        }
    }
    

### 三、跟踪分析

测试脚本如下

    
    
    testdb=# explain verbose select * from t_hash_partition where c1 = 1 OR c1 = 2;
                                         QUERY PLAN                                      
    -------------------------------------------------------------------------------------     Append  (cost=0.00..30.53 rows=6 width=200)
       ->  Seq Scan on public.t_hash_partition_1  (cost=0.00..15.25 rows=3 width=200)
             Output: t_hash_partition_1.c1, t_hash_partition_1.c2, t_hash_partition_1.c3
             Filter: ((t_hash_partition_1.c1 = 1) OR (t_hash_partition_1.c1 = 2))
       ->  Seq Scan on public.t_hash_partition_3  (cost=0.00..15.25 rows=3 width=200)
             Output: t_hash_partition_3.c1, t_hash_partition_3.c2, t_hash_partition_3.c3
             Filter: ((t_hash_partition_3.c1 = 1) OR (t_hash_partition_3.c1 = 2))
    (7 rows)
    

**ExecInitAppend**  
启动gdb,设置断点

    
    
    (gdb) b ExecInitAppend
    Breakpoint 1 at 0x6efa8a: file nodeAppend.c, line 103.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecInitAppend (node=0x27af638, estate=0x27be058, eflags=16) at nodeAppend.c:103
    103     AppendState *appendstate = makeNode(AppendState);
    

初始化AppendState

    
    
    (gdb) n
    113     Assert(!(eflags & EXEC_FLAG_MARK));
    (gdb) 
    119     ExecLockNonLeafAppendTables(node->partitioned_rels, estate);
    (gdb) 
    124     appendstate->ps.plan = (Plan *) node;
    (gdb) 
    125     appendstate->ps.state = estate;
    (gdb) 
    126     appendstate->ps.ExecProcNode = ExecAppend;
    (gdb) 
    129     appendstate->as_whichplan = INVALID_SUBPLAN_INDEX;
    (gdb) 
    

不需要运行期分区裁剪

    
    
    (gdb) 
    132     if (node->part_prune_info != NULL)
    (gdb) p node->part_prune_info
    $1 = (struct PartitionPruneInfo *) 0x0
    (gdb) 
    

需要初始化所有的的子计划

    
    
    (gdb) n
    190         nplans = list_length(node->appendplans);
    (gdb) 
    196         Assert(nplans > 0);
    (gdb) 
    198             bms_add_range(NULL, 0, nplans - 1);
    (gdb) 
    197         appendstate->as_valid_subplans = validsubplans =
    (gdb) n
    199         appendstate->as_prune_state = NULL;
    (gdb) p *validsubplans
    $4 = {nwords = 1, words = 0x27be38c}
    (gdb) p *validsubplans->words
    $5 = 3 --> 即No.0 + No.1
    (gdb) 
    

初始化结果元组类型和slot

    
    
    (gdb) n
    205     ExecInitResultTupleSlotTL(estate, &appendstate->ps);
    (gdb) 
    207     appendplanstates = (PlanState **) palloc(nplans *
    (gdb) 
    

在每一个有效的计划上执行ExecInitNode,同时保持结果到appendplanstates数组中.

    
    
    (gdb) 
    216     j = i = 0;
    (gdb) n
    217     firstvalid = nplans;
    (gdb) 
    218     foreach(lc, node->appendplans)
    (gdb) p nplans
    $6 = 2
    (gdb) p node->appendplans
    $7 = (List *) 0x27b30f0
    (gdb) p *node->appendplans
    $8 = {type = T_List, length = 2, head = 0x27b30c8, tail = 0x27b33d8}
    (gdb) 
    

遍历appendplans,初始化appendplans中的节点(SeqScan)

    
    
    (gdb) n
    220         if (bms_is_member(i, validsubplans))
    (gdb) n
    222             Plan       *initNode = (Plan *) lfirst(lc);
    (gdb) 
    228             if (i >= node->first_partial_plan && j < firstvalid)
    (gdb) 
    231             appendplanstates[j++] = ExecInitNode(initNode, estate, eflags);
    (gdb) p j
    $9 = 0
    (gdb) p i
    $10 = 0
    (gdb) n
    233         i++;
    (gdb) 
    218     foreach(lc, node->appendplans)
    (gdb) 
    220         if (bms_is_member(i, validsubplans))
    (gdb) 
    222             Plan       *initNode = (Plan *) lfirst(lc);
    (gdb) 
    228             if (i >= node->first_partial_plan && j < firstvalid)
    (gdb) p *initNode
    $11 = {type = T_SeqScan, startup_cost = 0, total_cost = 15.25, plan_rows = 3, plan_width = 200, parallel_aware = false, 
      parallel_safe = true, plan_node_id = 2, targetlist = 0x27b31a8, qual = 0x27b3308, lefttree = 0x0, righttree = 0x0, 
      initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    (gdb) n
    231             appendplanstates[j++] = ExecInitNode(initNode, estate, eflags);
    (gdb) 
    233         i++;
    (gdb) 
    218     foreach(lc, node->appendplans)
    (gdb) 
    236     appendstate->as_first_partial_plan = firstvalid;
    (gdb) 
    
    

完成初始化,其中choose_next_subplan函数为choose_next_subplan_locally函数

    
    
    (gdb) p firstvalid
    $12 = 2
    (gdb) n
    237     appendstate->appendplans = appendplanstates;
    (gdb) 
    238     appendstate->as_nplans = nplans;
    (gdb) 
    244     appendstate->ps.ps_ProjInfo = NULL;
    (gdb) 
    247     appendstate->choose_next_subplan = choose_next_subplan_locally;
    (gdb) 
    249     return appendstate;
    (gdb) p choose_next_subplan_locally
    $13 = {_Bool (AppendState *)} 0x6f02d8 <choose_next_subplan_locally>
    (gdb) p *appendstate
    $15 = {ps = {type = T_AppendState, plan = 0x27af638, state = 0x27be058, ExecProcNode = 0x6efe19 <ExecAppend>, 
        ExecProcNodeReal = 0x0, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, qual = 0x0, 
        lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x27be5c0, 
        ps_ExprContext = 0x0, ps_ProjInfo = 0x0, scandesc = 0x0}, appendplans = 0x27be6b8, as_nplans = 2, as_whichplan = -1, 
      as_first_partial_plan = 2, as_pstate = 0x0, pstate_len = 0, as_prune_state = 0x0, as_valid_subplans = 0x27be388, 
      choose_next_subplan = 0x6f02d8 <choose_next_subplan_locally>}
    

**ExecAppend**  
设置断点,进入ExecAppend

    
    
    (gdb) del 
    Delete all breakpoints? (y or n) y
    (gdb) b ExecAppend
    Breakpoint 2 at 0x6efe2a: file nodeAppend.c, line 261.
    (gdb) c
    Continuing.
    
    Breakpoint 2, ExecAppend (pstate=0x27be270) at nodeAppend.c:261
    261     AppendState *node = castNode(AppendState, pstate);
    

输入参数为先前已完成初始化的AppendState

    
    
    (gdb) p *(AppendState *)pstate
    $19 = {ps = {type = T_AppendState, plan = 0x27af638, state = 0x27be058, ExecProcNode = 0x6efe19 <ExecAppend>, 
        ExecProcNodeReal = 0x6efe19 <ExecAppend>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
        qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
        ps_ResultTupleSlot = 0x27be5c0, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, scandesc = 0x0}, appendplans = 0x27be6b8, 
      as_nplans = 2, as_whichplan = -1, as_first_partial_plan = 2, as_pstate = 0x0, pstate_len = 0, as_prune_state = 0x0, 
      as_valid_subplans = 0x27be388, choose_next_subplan = 0x6f02d8 <choose_next_subplan_locally>}
    

如果没有子计划选中,必须在开始前选择一个(选择子计划:No.0)

    
    
    (gdb) n
    263     if (node->as_whichplan < 0)
    (gdb) 
    269         if (node->as_whichplan == INVALID_SUBPLAN_INDEX &&
    (gdb) 
    270             !node->choose_next_subplan(node))
    (gdb) 
    269         if (node->as_whichplan == INVALID_SUBPLAN_INDEX &&
    (gdb) 
    274         else if (node->as_whichplan == NO_MATCHING_SUBPLANS)
    (gdb) p node->as_whichplan
    $20 = 0
    (gdb) n
    283         CHECK_FOR_INTERRUPTS();
    

获取子计划并执行

    
    
    (gdb) 
    288         Assert(node->as_whichplan >= 0 && node->as_whichplan < node->as_nplans);
    (gdb) 
    289         subnode = node->appendplans[node->as_whichplan];
    (gdb) 
    294         result = ExecProcNode(subnode);
    (gdb) p *subnode
    $21 = {type = T_SeqScanState, plan = 0x27b2728, state = 0x27be058, ExecProcNode = 0x6e4bde <ExecProcNodeFirst>, 
      ExecProcNodeReal = 0x71578d <ExecSeqScan>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
      qual = 0x27bec88, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
      ps_ResultTupleSlot = 0x27bebc8, ps_ExprContext = 0x27be7f8, ps_ProjInfo = 0x0, scandesc = 0x7f7d049417e0}
    

返回slot

    
    
    (gdb) n
    296         if (!TupIsNull(result))
    (gdb) p *result
    $22 = {type = T_TupleTableSlot, tts_isempty = false, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, 
      tts_tuple = 0x27c5500, tts_tupleDescriptor = 0x7f7d049417e0, tts_mcxt = 0x27bdf40, tts_buffer = 120, tts_nvalid = 1, 
      tts_values = 0x27be950, tts_isnull = 0x27be968, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x0}, tts_off = 4, tts_fixedTupleDescriptor = true}
    

如第一个子计划执行完毕,则选择下一个子计划(调用choose_next_subplan函数切换到下一个子计划)

    
    
    (gdb) c
    Continuing.
    
    Breakpoint 2, ExecAppend (pstate=0x27be270) at nodeAppend.c:261
    261     AppendState *node = castNode(AppendState, pstate);
    (gdb) n
    263     if (node->as_whichplan < 0)
    (gdb) 
    283         CHECK_FOR_INTERRUPTS();
    (gdb) 
    288         Assert(node->as_whichplan >= 0 && node->as_whichplan < node->as_nplans);
    (gdb) 
    289         subnode = node->appendplans[node->as_whichplan];
    (gdb) 
    294         result = ExecProcNode(subnode);
    (gdb) 
    296         if (!TupIsNull(result))
    (gdb) 
    307         if (!node->choose_next_subplan(node))
    (gdb) 
    309     }
    

DONE!

### 四、参考资料

[Parallel Append implementation](https://www.postgresql.org/message-id/CAJ3gD9dy0K_E8r727heqXoBmWZ83HwLFwdcaSSmBQ1+S+vRuUQ@mail.gmail.com)  
[Partition Elimination in PostgreSQL
11](https://blog.2ndquadrant.com/partition-elimination-postgresql-11/)

