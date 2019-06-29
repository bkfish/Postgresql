本节介绍了PortalStart->ExecutorStart(standard_ExecutorStart)->InitPlan->ExecInitNode函数的实现逻辑，该函数通过递归调用初始化计划树中的所有Plan节点。

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
    

### 二、源码解读

ExecInitNode函数递归初始化计划树中的所有Plan节点,返回对应给定的Plan Node节点的PlanState节点.

    
    
    /* ------------------------------------------------------------------------     *      ExecInitNode
     *
     *      Recursively initializes all the nodes in the plan tree rooted
     *      at 'node'.
     *      递归初始化计划树中的所有Plan节点.
     *
     *      Inputs:
     *        'node' is the current node of the plan produced by the query planner
     *        'estate' is the shared execution state for the plan tree
     *        'eflags' is a bitwise OR of flag bits described in executor.h
     *        node-查询计划器产生的当前节点
     *        estate-Plan树共享的执行状态信息
     *        eflags-一个位或标记位,在executor.h中描述
     *
     *      Returns a PlanState node corresponding to the given Plan node.
     *      返回对应Plan Node节点的PlanState节点
     * ------------------------------------------------------------------------     */
    PlanState *
    ExecInitNode(Plan *node, EState *estate, int eflags)
    {
        PlanState  *result;//结果
        List       *subps;//子PlanState链表
        ListCell   *l;//临时变量
    
        /*
         * do nothing when we get to the end of a leaf on tree.
         * 如node为NULL则返回NULL
         */
        if (node == NULL)
            return NULL;
    
        /*
         * Make sure there's enough stack available. Need to check here, in
         * addition to ExecProcNode() (via ExecProcNodeFirst()), to ensure the
         * stack isn't overrun while initializing the node tree.
         * 确保有足够的堆栈可用。
         * 除了ExecProcNode()(通过ExecProcNodeFirst()调用)，还需要在这里进行检查，以确保在初始化节点树时堆栈没有溢出。
         */
        check_stack_depth();
    
        switch (nodeTag(node))//根据节点类型进入相应的逻辑
        {
                /*
                 * 控制节点;control nodes
                 */
            case T_Result:
                result = (PlanState *) ExecInitResult((Result *) node,
                                                      estate, eflags);
                break;
    
            case T_ProjectSet:
                result = (PlanState *) ExecInitProjectSet((ProjectSet *) node,
                                                          estate, eflags);
                break;
    
            case T_ModifyTable:
                result = (PlanState *) ExecInitModifyTable((ModifyTable *) node,
                                                           estate, eflags);
                break;
    
            case T_Append:
                result = (PlanState *) ExecInitAppend((Append *) node,
                                                      estate, eflags);
                break;
    
            case T_MergeAppend:
                result = (PlanState *) ExecInitMergeAppend((MergeAppend *) node,
                                                           estate, eflags);
                break;
    
            case T_RecursiveUnion:
                result = (PlanState *) ExecInitRecursiveUnion((RecursiveUnion *) node,
                                                              estate, eflags);
                break;
    
            case T_BitmapAnd:
                result = (PlanState *) ExecInitBitmapAnd((BitmapAnd *) node,
                                                         estate, eflags);
                break;
    
            case T_BitmapOr:
                result = (PlanState *) ExecInitBitmapOr((BitmapOr *) node,
                                                        estate, eflags);
                break;
    
                /*
                 * 扫描节点;scan nodes
                 */
            case T_SeqScan:
                result = (PlanState *) ExecInitSeqScan((SeqScan *) node,
                                                       estate, eflags);
                break;
    
            case T_SampleScan:
                result = (PlanState *) ExecInitSampleScan((SampleScan *) node,
                                                          estate, eflags);
                break;
    
            case T_IndexScan:
                result = (PlanState *) ExecInitIndexScan((IndexScan *) node,
                                                         estate, eflags);
                break;
    
            case T_IndexOnlyScan:
                result = (PlanState *) ExecInitIndexOnlyScan((IndexOnlyScan *) node,
                                                             estate, eflags);
                break;
    
            case T_BitmapIndexScan:
                result = (PlanState *) ExecInitBitmapIndexScan((BitmapIndexScan *) node,
                                                               estate, eflags);
                break;
    
            case T_BitmapHeapScan:
                result = (PlanState *) ExecInitBitmapHeapScan((BitmapHeapScan *) node,
                                                              estate, eflags);
                break;
    
            case T_TidScan:
                result = (PlanState *) ExecInitTidScan((TidScan *) node,
                                                       estate, eflags);
                break;
    
            case T_SubqueryScan:
                result = (PlanState *) ExecInitSubqueryScan((SubqueryScan *) node,
                                                            estate, eflags);
                break;
    
            case T_FunctionScan:
                result = (PlanState *) ExecInitFunctionScan((FunctionScan *) node,
                                                            estate, eflags);
                break;
    
            case T_TableFuncScan:
                result = (PlanState *) ExecInitTableFuncScan((TableFuncScan *) node,
                                                             estate, eflags);
                break;
    
            case T_ValuesScan:
                result = (PlanState *) ExecInitValuesScan((ValuesScan *) node,
                                                          estate, eflags);
                break;
    
            case T_CteScan:
                result = (PlanState *) ExecInitCteScan((CteScan *) node,
                                                       estate, eflags);
                break;
    
            case T_NamedTuplestoreScan:
                result = (PlanState *) ExecInitNamedTuplestoreScan((NamedTuplestoreScan *) node,
                                                                   estate, eflags);
                break;
    
            case T_WorkTableScan:
                result = (PlanState *) ExecInitWorkTableScan((WorkTableScan *) node,
                                                             estate, eflags);
                break;
    
            case T_ForeignScan:
                result = (PlanState *) ExecInitForeignScan((ForeignScan *) node,
                                                           estate, eflags);
                break;
    
            case T_CustomScan:
                result = (PlanState *) ExecInitCustomScan((CustomScan *) node,
                                                          estate, eflags);
                break;
    
                /*
                 * 连接节点/join nodes
                 */
            case T_NestLoop:
                result = (PlanState *) ExecInitNestLoop((NestLoop *) node,
                                                        estate, eflags);
                break;
    
            case T_MergeJoin:
                result = (PlanState *) ExecInitMergeJoin((MergeJoin *) node,
                                                         estate, eflags);
                break;
    
            case T_HashJoin:
                result = (PlanState *) ExecInitHashJoin((HashJoin *) node,
                                                        estate, eflags);
                break;
    
                /*
                 * 物化节点/materialization nodes
                 */
            case T_Material:
                result = (PlanState *) ExecInitMaterial((Material *) node,
                                                        estate, eflags);
                break;
    
            case T_Sort:
                result = (PlanState *) ExecInitSort((Sort *) node,
                                                    estate, eflags);
                break;
    
            case T_Group:
                result = (PlanState *) ExecInitGroup((Group *) node,
                                                     estate, eflags);
                break;
    
            case T_Agg:
                result = (PlanState *) ExecInitAgg((Agg *) node,
                                                   estate, eflags);
                break;
    
            case T_WindowAgg:
                result = (PlanState *) ExecInitWindowAgg((WindowAgg *) node,
                                                         estate, eflags);
                break;
    
            case T_Unique:
                result = (PlanState *) ExecInitUnique((Unique *) node,
                                                      estate, eflags);
                break;
    
            case T_Gather:
                result = (PlanState *) ExecInitGather((Gather *) node,
                                                      estate, eflags);
                break;
    
            case T_GatherMerge:
                result = (PlanState *) ExecInitGatherMerge((GatherMerge *) node,
                                                           estate, eflags);
                break;
    
            case T_Hash:
                result = (PlanState *) ExecInitHash((Hash *) node,
                                                    estate, eflags);
                break;
    
            case T_SetOp:
                result = (PlanState *) ExecInitSetOp((SetOp *) node,
                                                     estate, eflags);
                break;
    
            case T_LockRows:
                result = (PlanState *) ExecInitLockRows((LockRows *) node,
                                                        estate, eflags);
                break;
    
            case T_Limit:
                result = (PlanState *) ExecInitLimit((Limit *) node,
                                                     estate, eflags);
                break;
    
            default:
                elog(ERROR, "unrecognized node type: %d", (int) nodeTag(node));
                result = NULL;      /* 避免优化器提示警告信息;keep compiler quiet */
                break;
        }
        //设置节点的ExecProcNode函数
        ExecSetExecProcNode(result, result->ExecProcNode);
    
        /*
         * Initialize any initPlans present in this node.  The planner put them in
         * a separate list for us.
         * 初始化该Plan节点中的所有initPlans.
         * 计划器把这些信息放到一个单独的链表中
         */
        subps = NIL;//初始化
        foreach(l, node->initPlan)//遍历initPlan
        {
            SubPlan    *subplan = (SubPlan *) lfirst(l);//子计划
            SubPlanState *sstate;//子计划状态
    
            Assert(IsA(subplan, SubPlan));
            sstate = ExecInitSubPlan(subplan, result);//初始化SubPlan
            subps = lappend(subps, sstate);//添加到链表中
        }
        result->initPlan = subps;//赋值
    
        /* Set up instrumentation for this node if requested */
        //如需要,配置instrumentation
        if (estate->es_instrument)
            result->instrument = InstrAlloc(1, estate->es_instrument);
    
        return result;
    }
    
    /*
     * If a node wants to change its ExecProcNode function after ExecInitNode()
     * has finished, it should do so with this function.  That way any wrapper
     * functions can be reinstalled, without the node having to know how that
     * works.
     * 如果一个节点想要在ExecInitNode()完成之后更改它的ExecProcNode函数，那么它应该使用这个函数。
     * 这样就可以重新安装任何包装器函数，而不必让节点知道它是如何工作的。
     */
    void
    ExecSetExecProcNode(PlanState *node, ExecProcNodeMtd function)
    {
        /*
         * Add a wrapper around the ExecProcNode callback that checks stack depth
         * during the first execution and maybe adds an instrumentation wrapper.
         * When the callback is changed after execution has already begun that
         * means we'll superfluously execute ExecProcNodeFirst, but that seems ok.
         * 在ExecProcNode回调函数添加一个包装器，在第一次执行时检查堆栈深度，可能还会添加一个检测包装器。
         * 在执行已经开始之后，当回调函数被更改时，这意味着再次执行ExecProcNodeFirst是多余的，但这似乎是可以的。
         */
        node->ExecProcNodeReal = function;
        node->ExecProcNode = ExecProcNodeFirst;
    }
    
    
    /* ----------------------------------------------------------------     *      ExecInitSeqScan
     *      初始化顺序扫描节点
     * ----------------------------------------------------------------     */
    SeqScanState *
    ExecInitSeqScan(SeqScan *node, EState *estate, int eflags)
    {
        SeqScanState *scanstate;
    
        /*
         * Once upon a time it was possible to have an outerPlan of a SeqScan, but
         * not any more.
         * 先前有可能存在外部的SeqScan计划,但现在该做法已废弃,这里进行校验
         */
        Assert(outerPlan(node) == NULL);
        Assert(innerPlan(node) == NULL);
    
        /*
         * create state structure
         * 创建SeqScanState数据结构体
         */
        scanstate = makeNode(SeqScanState);
        scanstate->ss.ps.plan = (Plan *) node;
        scanstate->ss.ps.state = estate;
        scanstate->ss.ps.ExecProcNode = ExecSeqScan;
    
        /*
         * Miscellaneous initialization
         * 初始化
         * create expression context for node
         * 创建表达式上下文
         */
        ExecAssignExprContext(estate, &scanstate->ss.ps);
    
        /*
         * open the scan relation
         * 打开扫描的Relation
         */
        scanstate->ss.ss_currentRelation =
            ExecOpenScanRelation(estate,
                                 node->scanrelid,
                                 eflags);
    
        /* and create slot with the appropriate rowtype */
        //使用合适的rowtype打开slot
        ExecInitScanTupleSlot(estate, &scanstate->ss,
                              RelationGetDescr(scanstate->ss.ss_currentRelation));
    
        /*
         * Initialize result type and projection.
         * 初始化结果类型和投影
         */
        ExecInitResultTypeTL(&scanstate->ss.ps);
        ExecAssignScanProjectionInfo(&scanstate->ss);
    
        /*
         * initialize child expressions
         * 初始化子表达式
         */
        scanstate->ss.ps.qual =
            ExecInitQual(node->plan.qual, (PlanState *) scanstate);
    
        return scanstate;
    }
    
    /* ----------------------------------------------------------------     *      ExecOpenScanRelation
     *
     *      Open the heap relation to be scanned by a base-level scan plan node.
     *      This should be called during the node's ExecInit routine.
     *       打开Heap Relation，由一个基表扫描计划节点扫描。这应该在节点的ExecInit例程中调用。
     * ----------------------------------------------------------------     */
    Relation
    ExecOpenScanRelation(EState *estate, Index scanrelid, int eflags)
    {
        Relation    rel;
    
        /* Open the relation. */
         //打开关系
        rel = ExecGetRangeTableRelation(estate, scanrelid);
    
        /*
         * Complain if we're attempting a scan of an unscannable relation, except
         * when the query won't actually be run.  This is a slightly klugy place
         * to do this, perhaps, but there is no better place.
         * 给出提示信息:试图扫描一个不存在的关系。这可能是一个有点笨拙的地方，但没有更好的提示了。
         */
        if ((eflags & (EXEC_FLAG_EXPLAIN_ONLY | EXEC_FLAG_WITH_NO_DATA)) == 0 &&
            !RelationIsScannable(rel))
            ereport(ERROR,
                    (errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
                     errmsg("materialized view \"%s\" has not been populated",
                            RelationGetRelationName(rel)),
                     errhint("Use the REFRESH MATERIALIZED VIEW command.")));
    
        return rel;
    }
    
    /* ----------------------------------------------------------------     *      ExecInitHashJoin
     *      初始化Hash连接节点
     *      Init routine for HashJoin node.
     *      Hash连接通过递归调用ExecInitNode函数初始化参与连接的Relation
     * ----------------------------------------------------------------     */
    HashJoinState *
    ExecInitHashJoin(HashJoin *node, EState *estate, int eflags)
    {
        HashJoinState *hjstate;
        Plan       *outerNode;
        Hash       *hashNode;
        List       *lclauses;
        List       *rclauses;
        List       *rhclauses;
        List       *hoperators;
        TupleDesc   outerDesc,
                    innerDesc;
        ListCell   *l;
    
        /* check for unsupported flags */
        //校验不支持的flags
        Assert(!(eflags & (EXEC_FLAG_BACKWARD | EXEC_FLAG_MARK)));
    
        /*
         * create state structure
         * 创建state数据结构体
         */
        hjstate = makeNode(HashJoinState);
        hjstate->js.ps.plan = (Plan *) node;
        hjstate->js.ps.state = estate;
    
        /*
         * See ExecHashJoinInitializeDSM() and ExecHashJoinInitializeWorker()
         * where this function may be replaced with a parallel version, if we
         * managed to launch a parallel query.
         * 请参阅ExecHashJoinInitializeWorker()和ExecHashJoinInitializeWorker()，
         * 如果成功启动了并行查询，这个函数可以用一个并行版本替换。
         */
        hjstate->js.ps.ExecProcNode = ExecHashJoin;
        hjstate->js.jointype = node->join.jointype;
    
        /*
         * Miscellaneous initialization
         * 初始化
         * create expression context for node
         */
        ExecAssignExprContext(estate, &hjstate->js.ps);
    
        /*
         * initialize child nodes
         * 初始化子节点
         * 
         * Note: we could suppress the REWIND flag for the inner input, which
         * would amount to betting that the hash will be a single batch.  Not
         * clear if this would be a win or not.
         * 注意:可以禁止内部输入的REWIND标志，这相当于打赌散列将是单个批处理。
         * 不清楚这是否会是一场胜利。
         */
        outerNode = outerPlan(node);
        hashNode = (Hash *) innerPlan(node);
    
        outerPlanState(hjstate) = ExecInitNode(outerNode, estate, eflags);//递归处理外表节点
        outerDesc = ExecGetResultType(outerPlanState(hjstate));//
        innerPlanState(hjstate) = ExecInitNode((Plan *) hashNode, estate, eflags);//递归处理内表节点
        innerDesc = ExecGetResultType(innerPlanState(hjstate));
    
        /*
         * Initialize result slot, type and projection.
         * 初始化节点slot/类型和投影
         */
        ExecInitResultTupleSlotTL(&hjstate->js.ps);
        ExecAssignProjectionInfo(&hjstate->js.ps, NULL);
    
        /*
         * tuple table initialization
         * 元组表初始化
         */
        hjstate->hj_OuterTupleSlot = ExecInitExtraTupleSlot(estate, outerDesc);
    
        /*
         * detect whether we need only consider the first matching inner tuple
         * 检测是否只需要考虑内表元组的首次匹配
         */
        hjstate->js.single_match = (node->join.inner_unique ||
                                    node->join.jointype == JOIN_SEMI);
    
        /* set up null tuples for outer joins, if needed */
        //配置外连接的NULL元组
        switch (node->join.jointype)
        {
            case JOIN_INNER:
            case JOIN_SEMI:
                break;
            case JOIN_LEFT:
            case JOIN_ANTI://左连接&半连接
                hjstate->hj_NullInnerTupleSlot =
                    ExecInitNullTupleSlot(estate, innerDesc);
                break;
            case JOIN_RIGHT:
                hjstate->hj_NullOuterTupleSlot =
                    ExecInitNullTupleSlot(estate, outerDesc);
                break;
            case JOIN_FULL:
                hjstate->hj_NullOuterTupleSlot =
                    ExecInitNullTupleSlot(estate, outerDesc);
                hjstate->hj_NullInnerTupleSlot =
                    ExecInitNullTupleSlot(estate, innerDesc);
                break;
            default:
                elog(ERROR, "unrecognized join type: %d",
                     (int) node->join.jointype);
        }
    
        /*
         * now for some voodoo.  our temporary tuple slot is actually the result
         * tuple slot of the Hash node (which is our inner plan).  we can do this
         * because Hash nodes don't return tuples via ExecProcNode() -- instead
         * the hash join node uses ExecScanHashBucket() to get at the contents of
         * the hash table.  -cim 6/9/91
         * 现在来点巫术。
         * 临时tuple槽实际上是散列节点的结果tuple槽(这是我们的内部计划)。
         * 之可以这样做，是因为哈希节点不会通过ExecProcNode()返回元组——
         * 相反，哈希连接节点使用ExecScanHashBucket()来获取哈希表的内容。
         *                by cim 6/9/91
         */
        {
            HashState  *hashstate = (HashState *) innerPlanState(hjstate);
            TupleTableSlot *slot = hashstate->ps.ps_ResultTupleSlot;
    
            hjstate->hj_HashTupleSlot = slot;
        }
    
        /*
         * initialize child expressions
         * 初始化子表达式
         */
        hjstate->js.ps.qual =
            ExecInitQual(node->join.plan.qual, (PlanState *) hjstate);
        hjstate->js.joinqual =
            ExecInitQual(node->join.joinqual, (PlanState *) hjstate);
        hjstate->hashclauses =
            ExecInitQual(node->hashclauses, (PlanState *) hjstate);
    
        /*
         * initialize hash-specific info
         * 初始化hash相关的信息
         */
        hjstate->hj_HashTable = NULL;
        hjstate->hj_FirstOuterTupleSlot = NULL;
    
        hjstate->hj_CurHashValue = 0;
        hjstate->hj_CurBucketNo = 0;
        hjstate->hj_CurSkewBucketNo = INVALID_SKEW_BUCKET_NO;
        hjstate->hj_CurTuple = NULL;
    
        /*
         * Deconstruct the hash clauses into outer and inner argument values, so
         * that we can evaluate those subexpressions separately.  Also make a list
         * of the hash operator OIDs, in preparation for looking up the hash
         * functions to use.
         * 将哈希子句解构为外部和内部参数值，以便能够分别计算这些子表达式。
         * 还可以列出哈希运算符oid，以便查找要使用的哈希函数。
         */
        lclauses = NIL;
        rclauses = NIL;
        rhclauses = NIL;
        hoperators = NIL;
        foreach(l, node->hashclauses)
        {
            OpExpr     *hclause = lfirst_node(OpExpr, l);
    
            lclauses = lappend(lclauses, ExecInitExpr(linitial(hclause->args),
                                                      (PlanState *) hjstate));
            rclauses = lappend(rclauses, ExecInitExpr(lsecond(hclause->args),
                                                      (PlanState *) hjstate));
            rhclauses = lappend(rhclauses, ExecInitExpr(lsecond(hclause->args),
                                                       innerPlanState(hjstate)));
            hoperators = lappend_oid(hoperators, hclause->opno);
        }
        hjstate->hj_OuterHashKeys = lclauses;
        hjstate->hj_InnerHashKeys = rclauses;
        hjstate->hj_HashOperators = hoperators;
        /* child Hash node needs to evaluate inner hash keys, too */
        ((HashState *) innerPlanState(hjstate))->hashkeys = rhclauses;
    
        hjstate->hj_JoinState = HJ_BUILD_HASHTABLE;
        hjstate->hj_MatchedOuter = false;
        hjstate->hj_OuterNotEmpty = false;
    
        return hjstate;
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
    
    

启动gdb,设置断点,进入ExecInitNode

    
    
    (gdb) b ExecInitNode
    Breakpoint 1 at 0x6e3b90: file execProcnode.c, line 148.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecInitNode (node=0x1b71f90, estate=0x1b78f48, eflags=16) at execProcnode.c:148
    warning: Source file is more recent than executable.
    148     if (node == NULL)
    

输入参数,node为T_Sort

    
    
    (gdb) p *node
    $1 = {type = T_Sort, startup_cost = 20070.931487218411, total_cost = 20320.931487218411, plan_rows = 100000, 
      plan_width = 47, parallel_aware = false, parallel_safe = true, plan_node_id = 0, targetlist = 0x1b762c0, qual = 0x0, 
      lefttree = 0x1b75728, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    (gdb) p *estate
    $2 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x1b31e10, es_crosscheck_snapshot = 0x0, 
      es_range_table = 0x1b75c00, es_plannedstmt = 0x1b77d58, 
      es_sourceText = 0x1a8ceb8 "select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je \nfrom t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je \n", ' ' <repeats 24 times>, "from t_grxx gr inner join t_jfxx jf \n", ' ' <repeats 34 times>..., 
      es_junkFilter = 0x0, es_output_cid = 0, es_result_relations = 0x0, es_num_result_relations = 0, 
      es_result_relation_info = 0x0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, 
      es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, es_trig_tuple_slot = 0x0, 
      es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, es_param_exec_vals = 0x0, 
      es_queryEnv = 0x0, es_query_cxt = 0x1b78e30, es_tupleTable = 0x0, es_rowMarks = 0x0, es_processed = 0, es_lastoid = 0, 
      es_top_eflags = 16, es_instrument = 0, es_finished = false, es_exprcontexts = 0x0, es_subplanstates = 0x0, 
      es_auxmodifytables = 0x0, es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, es_epqTupleSet = 0x0, es_epqScanDone = 0x0, 
      es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, es_jit = 0x0, es_jit_worker_instr = 0x0}
    

检查堆栈

    
    
    (gdb) n
    156     check_stack_depth();
    

进入相应的处理逻辑

    
    
    158     switch (nodeTag(node))
    (gdb) 
    313             result = (PlanState *) ExecInitSort((Sort *) node,
    

返回结果

    
    
    (gdb) p *result
    $3 = {type = 11084746, plan = 0x69900000699, state = 0x0, ExecProcNode = 0x0, ExecProcNodeReal = 0x0, 
      instrument = 0x1000000000000, worker_instrument = 0x0, worker_jit_instrument = 0x200000001, qual = 0x0, lefttree = 0x0, 
      righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x7f7f7f7f7f7f7f7e, ps_ResultTupleSlot = 0x7f7f7f7f7f7f7f7f, 
      ps_ExprContext = 0x7f7f7f7f7f7f7f7f, ps_ProjInfo = 0x80, scandesc = 0x0}
    

设置断点,进入ExecSetExecProcNode

    
    
    (gdb) b ExecSetExecProcNode
    Breakpoint 2 at 0x6e41a1: file execProcnode.c, line 414.
    (gdb) c
    Continuing.
    

ExecSetExecProcNode->输入参数,function为ExecSeqScan,在实际执行时调用此函数

    
    
    (gdb) p *function
    $5 = {TupleTableSlot *(struct PlanState *)} 0x714d59 <ExecSeqScan>
    (gdb) p *node
    $6 = {type = T_SeqScanState, plan = 0x1b74f58, state = 0x1b78f48, ExecProcNode = 0x714d59 <ExecSeqScan>, 
      ExecProcNodeReal = 0x0, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, qual = 0x0, 
      lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x1b7a9c0, 
      ps_ExprContext = 0x1b7a5a8, ps_ProjInfo = 0x1b7aa80, scandesc = 0x7f07174f5308}
    

回到最上层的ExecInitNode,initPlan为NULL

    
    
    (gdb) 
    ExecInitNode (node=0x1b74f58, estate=0x1b78f48, eflags=16) at execProcnode.c:379
    379     subps = NIL;
    (gdb) 
    380     foreach(l, node->initPlan)
    (gdb) p *node
    $7 = {type = T_SeqScan, startup_cost = 0, total_cost = 1726, plan_rows = 100000, plan_width = 16, parallel_aware = false, 
      parallel_safe = true, plan_node_id = 5, targetlist = 0x1b74e20, qual = 0x0, lefttree = 0x0, righttree = 0x0, 
      initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    

完成调用

    
    
    (gdb) n
    389     result->initPlan = subps;
    (gdb) 
    392     if (estate->es_instrument)
    (gdb) 
    395     return result;
    (gdb) 
    396 }
    

下面重点考察ExecInitSeqScan和ExecInitHashJoin,首先是ExecInitHashJoin  
ExecInitHashJoin->

    
    
    (gdb) b ExecInitSeqScan
    Breakpoint 3 at 0x714daf: file nodeSeqscan.c, line 148.
    (gdb) b ExecInitHashJoin
    Breakpoint 4 at 0x701f60: file nodeHashjoin.c, line 604.
    (gdb) c
    Continuing.
    
    Breakpoint 4, ExecInitHashJoin (node=0x1b737c0, estate=0x1b78f48, eflags=16) at nodeHashjoin.c:604
    warning: Source file is more recent than executable.
    604     Assert(!(eflags & (EXEC_FLAG_BACKWARD | EXEC_FLAG_MARK)));
    

ExecInitHashJoin->校验并初始化

    
    
    604     Assert(!(eflags & (EXEC_FLAG_BACKWARD | EXEC_FLAG_MARK)));
    (gdb) n
    609     hjstate = makeNode(HashJoinState);
    (gdb) 
    610     hjstate->js.ps.plan = (Plan *) node;
    (gdb) 
    611     hjstate->js.ps.state = estate;
    (gdb) 
    618     hjstate->js.ps.ExecProcNode = ExecHashJoin;
    (gdb) 
    619     hjstate->js.jointype = node->join.jointype;
    (gdb) 
    626     ExecAssignExprContext(estate, &hjstate->js.ps);
    

ExecInitHashJoin->初步的数据结构体

    
    
    (gdb) p *hjstate
    $8 = {js = {ps = {type = T_HashJoinState, plan = 0x1b737c0, state = 0x1b78f48, ExecProcNode = 0x701efa <ExecHashJoin>, 
          ExecProcNodeReal = 0x0, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, qual = 0x0, 
          lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x0, 
          ps_ExprContext = 0x0, ps_ProjInfo = 0x0, scandesc = 0x0}, jointype = JOIN_INNER, single_match = false, 
        joinqual = 0x0}, hashclauses = 0x0, hj_OuterHashKeys = 0x0, hj_InnerHashKeys = 0x0, hj_HashOperators = 0x0, 
      hj_HashTable = 0x0, hj_CurHashValue = 0, hj_CurBucketNo = 0, hj_CurSkewBucketNo = 0, hj_CurTuple = 0x0, 
      hj_OuterTupleSlot = 0x0, hj_HashTupleSlot = 0x0, hj_NullOuterTupleSlot = 0x0, hj_NullInnerTupleSlot = 0x0, 
      hj_FirstOuterTupleSlot = 0x0, hj_JoinState = 0, hj_MatchedOuter = false, hj_OuterNotEmpty = false}
    

ExecInitHashJoin->获取HashJoin的outer&inner(PG视为Hash节点)  
outerNode为HashJoin,innerNode为Hash

    
    
    (gdb) n
    635     outerNode = outerPlan(node);
    gdb) n
    636     hashNode = (Hash *) innerPlan(node);
    (gdb) 
    638     outerPlanState(hjstate) = ExecInitNode(outerNode, estate, eflags);
    (gdb) p *node
    $9 = {join = {plan = {type = T_HashJoin, startup_cost = 3754, total_cost = 8689.6112499999999, plan_rows = 100000, 
          plan_width = 47, parallel_aware = false, parallel_safe = true, plan_node_id = 1, targetlist = 0x1b74cc8, qual = 0x0, 
          lefttree = 0x1b73320, righttree = 0x1b73728, initPlan = 0x0, extParam = 0x0, allParam = 0x0}, jointype = JOIN_INNER, 
        inner_unique = true, joinqual = 0x0}, hashclauses = 0x1b74bb8}
    (gdb) p *outerNode
    $12 = {type = T_HashJoin, startup_cost = 3465, total_cost = 8138, plan_rows = 100000, plan_width = 31, 
      parallel_aware = false, parallel_safe = true, plan_node_id = 2, targetlist = 0x1b75588, qual = 0x0, lefttree = 0x1b72da0, 
      righttree = 0x1b73288, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    (gdb) p *hashNode
    $11 = {plan = {type = T_Hash, startup_cost = 164, total_cost = 164, plan_rows = 10000, plan_width = 20, 
        parallel_aware = false, parallel_safe = true, plan_node_id = 6, targetlist = 0x1b75c08, qual = 0x0, 
        lefttree = 0x1b73570, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}, skewTable = 16742, 
      skewColumn = 1, skewInherit = false, rows_total = 0}  
    

ExecInitHashJoin->进入outerNode的HashJoin,直接跳过

    
    
    (gdb) n
    
    Breakpoint 4, ExecInitHashJoin (node=0x1b73320, estate=0x1b78f48, eflags=16) at nodeHashjoin.c:604
    604     Assert(!(eflags & (EXEC_FLAG_BACKWARD | EXEC_FLAG_MARK)));
    (gdb) finish
    Run till exit from #0  ExecInitHashJoin (node=0x1b73320, estate=0x1b78f48, eflags=16) at nodeHashjoin.c:604
    

ExecInitHashJoin->进入innerNode的调用(ExecInitSeqScan)

    
    
    Breakpoint 3, ExecInitSeqScan (node=0x1b72da0, estate=0x1b78f48, eflags=16) at nodeSeqscan.c:148
    warning: Source file is more recent than executable.
    148     Assert(outerPlan(node) == NULL);
    (gdb) 
    

**ExecInitSeqScan**  
ExecInitSeqScan- >执行校验,并创建Node.  
注意:ExecProcNode=ExecSeqScan

    
    
    148     Assert(outerPlan(node) == NULL);
    (gdb) n
    149     Assert(innerPlan(node) == NULL);
    (gdb) 
    154     scanstate = makeNode(SeqScanState);
    (gdb) 
    155     scanstate->ss.ps.plan = (Plan *) node;
    (gdb) 
    156     scanstate->ss.ps.state = estate;
    (gdb) 
    157     scanstate->ss.ps.ExecProcNode = ExecSeqScan;
    (gdb) 
    164     ExecAssignExprContext(estate, &scanstate->ss.ps);
    (gdb) p *scanstate
    $1 = {ss = {ps = {type = T_SeqScanState, plan = 0x1b72da0, state = 0x1b78f48, ExecProcNode = 0x714d59 <ExecSeqScan>, 
          ExecProcNodeReal = 0x0, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, qual = 0x0, 
          lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x0, 
          ps_ExprContext = 0x0, ps_ProjInfo = 0x0, scandesc = 0x0}, ss_currentRelation = 0x0, ss_currentScanDesc = 0x0, 
        ss_ScanTupleSlot = 0x0}, pscan_len = 0}
    

ExecInitSeqScan->打开Relation

    
    
    gdb) n
    173         ExecOpenScanRelation(estate,
    (gdb) 
    172     scanstate->ss.ss_currentRelation =
    (gdb) 
    179                           RelationGetDescr(scanstate->ss.ss_currentRelation));
    (gdb) p *scanstate->ss.ss_currentRelation
    $3 = {rd_node = {spcNode = 1663, dbNode = 16402, relNode = 16747}, rd_smgr = 0x1b64650, rd_refcnt = 1, rd_backend = -1, 
      rd_islocaltemp = false, rd_isnailed = false, rd_isvalid = true, rd_indexvalid = 1 '\001', rd_statvalid = true, 
      rd_createSubid = 0, rd_newRelfilenodeSubid = 0, rd_rel = 0x7f07174f5c78, rd_att = 0x7f07174f5d90, rd_id = 16747, 
      rd_lockInfo = {lockRelId = {relId = 16747, dbId = 16402}}, rd_rules = 0x0, rd_rulescxt = 0x0, trigdesc = 0x0, 
      rd_rsdesc = 0x0, rd_fkeylist = 0x0, rd_fkeyvalid = false, rd_partkeycxt = 0x0, rd_partkey = 0x0, rd_pdcxt = 0x0, 
      rd_partdesc = 0x0, rd_partcheck = 0x0, rd_indexlist = 0x7f0717447328, rd_oidindex = 0, rd_pkindex = 0, 
      rd_replidindex = 0, rd_statlist = 0x0, rd_indexattr = 0x0, rd_projindexattr = 0x0, rd_keyattr = 0x0, rd_pkattr = 0x0, 
      rd_idattr = 0x0, rd_projidx = 0x0, rd_pubactions = 0x0, rd_options = 0x0, rd_index = 0x0, rd_indextuple = 0x0, 
      rd_amhandler = 0, rd_indexcxt = 0x0, rd_amroutine = 0x0, rd_opfamily = 0x0, rd_opcintype = 0x0, rd_support = 0x0, 
      rd_supportinfo = 0x0, rd_indoption = 0x0, rd_indexprs = 0x0, rd_indpred = 0x0, rd_exclops = 0x0, rd_exclprocs = 0x0, 
      rd_exclstrats = 0x0, rd_amcache = 0x0, rd_indcollation = 0x0, rd_fdwroutine = 0x0, rd_toastoid = 0, 
      pgstat_info = 0x1b0b6c0}
    

ExecInitSeqScan->使用合适的rowtype打开slot(初始化ScanTupleSlot)

    
    
    (gdb) 
    178     ExecInitScanTupleSlot(estate, &scanstate->ss,
    (gdb) 
    184     ExecInitResultTupleSlotTL(estate, &scanstate->ss.ps);
    (gdb) p *scanstate->ss.ss_ScanTupleSlot
    $4 = {type = T_TupleTableSlot, tts_isempty = true, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, 
      tts_tuple = 0x0, tts_tupleDescriptor = 0x7f07174f5d90, tts_mcxt = 0x1b78e30, tts_buffer = 0, tts_nvalid = 0, 
      tts_values = 0x1b79a98, tts_isnull = 0x1b79ab0, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = true}
    (gdb) 
    

ExecInitSeqScan->初始化结果类型和投影

    
    
    (gdb) n
    185     ExecAssignScanProjectionInfo(&scanstate->ss);
    (gdb) 
    191         ExecInitQual(node->plan.qual, (PlanState *) scanstate);
    (gdb) p *scanstate
    $5 = {ss = {ps = {type = T_SeqScanState, plan = 0x1b72da0, state = 0x1b78f48, ExecProcNode = 0x714d59 <ExecSeqScan>, 
          ExecProcNodeReal = 0x0, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, qual = 0x0, 
          lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, ps_ResultTupleSlot = 0x1b79d48, 
          ps_ExprContext = 0x1b79978, ps_ProjInfo = 0x1b79e08, scandesc = 0x7f07174f5d90}, ss_currentRelation = 0x7f07174f5a60, 
        ss_currentScanDesc = 0x0, ss_ScanTupleSlot = 0x1b79a38}, pscan_len = 0}
    (gdb) p *scanstate->ss.ps.ps_ResultTupleSlot
    $6 = {type = T_TupleTableSlot, tts_isempty = true, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, 
      tts_tuple = 0x0, tts_tupleDescriptor = 0x1b79b30, tts_mcxt = 0x1b78e30, tts_buffer = 0, tts_nvalid = 0, 
      tts_values = 0x1b79da8, tts_isnull = 0x1b79dc0, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = true}
    (gdb) p *scanstate->ss.ps.ps_ProjInfo
    $7 = {type = T_ProjectionInfo, pi_state = {tag = {type = T_ExprState}, flags = 6 '\006', resnull = false, resvalue = 0, 
        resultslot = 0x1b79d48, steps = 0x1b79ea0, evalfunc = 0x6d104b <ExecInterpExprStillValid>, expr = 0x1b72c68, 
        evalfunc_private = 0x6cec02 <ExecInterpExpr>, steps_len = 5, steps_alloc = 16, parent = 0x1b79860, ext_params = 0x0, 
        innermost_caseval = 0x0, innermost_casenull = 0x0, innermost_domainval = 0x0, innermost_domainnull = 0x0}, 
      pi_exprContext = 0x1b79978}
    

ExecInitSeqScan->初始化子条件表达式(为NULL),返回结果

    
    
    (gdb) n
    190     scanstate->ss.ps.qual =
    (gdb) 
    193     return scanstate;
    (gdb) p *scanstate->ss.ps.qual
    Cannot access memory at address 0x0
    

ExecInitSeqScan->回到ExecInitNode for Node SeqScan

    
    
    (gdb) n
    194 }
    (gdb) 
    ExecInitNode (node=0x1b72da0, estate=0x1b78f48, eflags=16) at execProcnode.c:209
    warning: Source file is more recent than executable.
    209             break;
    

ExecInitSeqScan->回到ExecInitNode,结束调用

    
    
    (gdb) n
    379     subps = NIL;
    (gdb) 
    380     foreach(l, node->initPlan)
    (gdb) 
    389     result->initPlan = subps;
    (gdb) 
    392     if (estate->es_instrument)
    (gdb) 
    395     return result;
    (gdb) 
    396 }
    (gdb) 
    

ExecInitSeqScan->回到ExecInitHashJoin

    
    
    (gdb) 
    ExecInitHashJoin (node=0x1b73320, estate=0x1b78f48, eflags=16) at nodeHashjoin.c:639
    639     outerDesc = ExecGetResultType(outerPlanState(hjstate));
    

**ExecInitHashJoin**  
ExecInitHashJoin- >完成outer relation的处理,开始处理inner relation(递归调用ExecInitNode)

    
    
    639     outerDesc = ExecGetResultType(outerPlanState(hjstate));
    (gdb) n
    640     innerPlanState(hjstate) = ExecInitNode((Plan *) hashNode, estate, eflags);
    (gdb) 
    
    Breakpoint 2, ExecInitSeqScan (node=0x1b72ff0, estate=0x1b78f48, eflags=16) at nodeSeqscan.c:148
    148     Assert(outerPlan(node) == NULL);
    (gdb) del 2
    (gdb) finish
    Run till exit from #0  ExecInitSeqScan (node=0x1b72ff0, estate=0x1b78f48, eflags=16) at nodeSeqscan.c:148
    0x00000000006e3cd2 in ExecInitNode (node=0x1b72ff0, estate=0x1b78f48, eflags=16) at execProcnode.c:207
    207             result = (PlanState *) ExecInitSeqScan((SeqScan *) node,
    Value returned is $10 = (SeqScanState *) 0x1b7a490
    (gdb) 
    ...
    (gdb) n
    641     innerDesc = ExecGetResultType(innerPlanState(hjstate));
    

ExecInitHashJoin->查看outerDesc和innerDesc

    
    
    (gdb) p *outerDesc
    $14 = {natts = 3, tdtypeid = 2249, tdtypmod = -1, tdhasoid = false, tdrefcount = -1, constr = 0x0, attrs = 0x1b79b50}
    (gdb) p *innerDesc
    $15 = {natts = 3, tdtypeid = 2249, tdtypmod = -1, tdhasoid = false, tdrefcount = -1, constr = 0x0, attrs = 0x1b7ab38}
    (gdb) 
    

ExecInitHashJoin->初始化节点slot/类型/投影/元组表等

    
    
    (gdb) n
    647     ExecAssignProjectionInfo(&hjstate->js.ps, NULL);
    (gdb) 
    652     hjstate->hj_OuterTupleSlot = ExecInitExtraTupleSlot(estate, outerDesc);
    (gdb) 
    657     hjstate->js.single_match = (node->join.inner_unique ||
    (gdb) 
    658                                 node->join.jointype == JOIN_SEMI);
    (gdb) p *hjstate
    $16 = {js = {ps = {type = T_HashJoinState, plan = 0x1b73320, state = 0x1b78f48, ExecProcNode = 0x701efa <ExecHashJoin>, 
          ExecProcNodeReal = 0x0, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, qual = 0x0, 
          lefttree = 0x1b79860, righttree = 0x1b7a2b8, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
          ps_ResultTupleSlot = 0x1b7ad30, ps_ExprContext = 0x1b797a0, ps_ProjInfo = 0x1b85bf8, scandesc = 0x0}, 
        jointype = JOIN_INNER, single_match = false, joinqual = 0x0}, hashclauses = 0x0, hj_OuterHashKeys = 0x0, 
      hj_InnerHashKeys = 0x0, hj_HashOperators = 0x0, hj_HashTable = 0x0, hj_CurHashValue = 0, hj_CurBucketNo = 0, 
      hj_CurSkewBucketNo = 0, hj_CurTuple = 0x0, hj_OuterTupleSlot = 0x1b860a8, hj_HashTupleSlot = 0x0, 
      hj_NullOuterTupleSlot = 0x0, hj_NullInnerTupleSlot = 0x0, hj_FirstOuterTupleSlot = 0x0, hj_JoinState = 0, 
      hj_MatchedOuter = false, hj_OuterNotEmpty = false}
    (gdb) p *hjstate->js.ps.ps_ResultTupleSlot #结果TupleSlot
    $20 = {type = T_TupleTableSlot, tts_isempty = true, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, 
      tts_tuple = 0x0, tts_tupleDescriptor = 0x1b857b8, tts_mcxt = 0x1b78e30, tts_buffer = 0, tts_nvalid = 0, 
      tts_values = 0x1b7ad90, tts_isnull = 0x1b7adb8, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = true}  
    (gdb) p *hjstate->js.ps.ps_ProjInfo #投影
    $18 = {type = T_ProjectionInfo, pi_state = {tag = {type = T_ExprState}, flags = 6 '\006', resnull = false, resvalue = 0, 
        resultslot = 0x1b7ad30, steps = 0x1b85c90, evalfunc = 0x6d104b <ExecInterpExprStillValid>, expr = 0x1b75588, 
        evalfunc_private = 0x6cec02 <ExecInterpExpr>, steps_len = 8, steps_alloc = 16, parent = 0x1b79588, ext_params = 0x0, 
        innermost_caseval = 0x0, innermost_casenull = 0x0, innermost_domainval = 0x0, innermost_domainnull = 0x0}, 
      pi_exprContext = 0x1b797a0}
    (gdb) p *hjstate->hj_OuterTupleSlot #元组slot
    $19 = {type = T_TupleTableSlot, tts_isempty = true, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, 
      tts_tuple = 0x0, tts_tupleDescriptor = 0x1b79b30, tts_mcxt = 0x1b78e30, tts_buffer = 0, tts_nvalid = 0, 
      tts_values = 0x1b86108, tts_isnull = 0x1b86120, tts_mintuple = 0x0, tts_minhdr = {t_len = 0, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x0}, tts_off = 0, tts_fixedTupleDescriptor = true}
    (gdb) n
    657     hjstate->js.single_match = (node->join.inner_unique ||
    (gdb) 
    661     switch (node->join.jointype)
    (gdb) p hjstate->js.single_match
    $21 = false        
    

ExecInitHashJoin->配置外连接的NULL元组(不需要)

    
    
    (gdb) n
    665             break;
    

ExecInitHashJoin->获取Hash操作的State,注意ExecProcNode是一个包装器(ExecProcNodeFirst),实际的函数是ExecHash

    
    
    (gdb) 
    694         HashState  *hashstate = (HashState *) innerPlanState(hjstate);
    (gdb) 
    695         TupleTableSlot *slot = hashstate->ps.ps_ResultTupleSlot;
    (gdb) n
    697         hjstate->hj_HashTupleSlot = slot;
    (gdb) 
    (gdb) p *hashstate
    $22 = {ps = {type = T_HashState, plan = 0x1b73288, state = 0x1b78f48, ExecProcNode = 0x6e41bb <ExecProcNodeFirst>, 
        ExecProcNodeReal = 0x6fb5f2 <ExecHash>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
        qual = 0x0, lefttree = 0x1b7a490, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
        ps_ResultTupleSlot = 0x1b856f8, ps_ExprContext = 0x1b7a3d0, ps_ProjInfo = 0x0, scandesc = 0x0}, hashtable = 0x0, 
      hashkeys = 0x0, shared_info = 0x0, hinstrument = 0x0, parallel_state = 0x0}
    

ExecInitHashJoin->初始化(子)表达式,均为NULL

    
    
    (gdb) n
    697         hjstate->hj_HashTupleSlot = slot;
    (gdb) 
    704         ExecInitQual(node->join.plan.qual, (PlanState *) hjstate);
    (gdb) n
    703     hjstate->js.ps.qual =
    (gdb) 
    706         ExecInitQual(node->join.joinqual, (PlanState *) hjstate);
    (gdb) 
    705     hjstate->js.joinqual =
    (gdb) 
    708         ExecInitQual(node->hashclauses, (PlanState *) hjstate);
    (gdb) 
    707     hjstate->hashclauses =
    (gdb) 
    713     hjstate->hj_HashTable = NULL;
    (gdb) 
    714     hjstate->hj_FirstOuterTupleSlot = NULL;
    (gdb) 
    716     hjstate->hj_CurHashValue = 0;
    (gdb) 
    717     hjstate->hj_CurBucketNo = 0;
    (gdb) 
    718     hjstate->hj_CurSkewBucketNo = INVALID_SKEW_BUCKET_NO;
    (gdb) 
    719     hjstate->hj_CurTuple = NULL;
    (gdb) 
    727     lclauses = NIL;
    (gdb) p *hjstate->js.ps.qual
    Cannot access memory at address 0x0
    (gdb) p *hjstate->js.joinqual
    Cannot access memory at address 0x0
    (gdb) 
    

ExecInitHashJoin->将哈希子句解构为外部和内部参数值，以便能够分别计算这些子表达式;还可以列出哈希运算符oid，以便查找要使用的哈希函数.

    
    
    ...
    (gdb) p *hjstate->hj_OuterHashKeys
    $25 = {type = T_List, length = 1, head = 0x1b86f98, tail = 0x1b86f98}
    (gdb) p *hjstate->hj_InnerHashKeys
    $26 = {type = T_List, length = 1, head = 0x1b87708, tail = 0x1b87708}
    (gdb) p *hjstate->hj_HashOperators
    $27 = {type = T_OidList, length = 1, head = 0x1b87768, tail = 0x1b87768}
    (gdb) 
    

ExecInitHashJoin->完成调用

    
    
    (gdb) n
    746     hjstate->hj_JoinState = HJ_BUILD_HASHTABLE;
    (gdb) 
    747     hjstate->hj_MatchedOuter = false;
    (gdb) 
    748     hjstate->hj_OuterNotEmpty = false;
    (gdb) 
    750     return hjstate;
    (gdb) 
    751 }
    

ExecInitHashJoin->最终结果(注意:这是最上层的HashJoin)

    
    
    (gdb) p *hjstate
    $28 = {js = {ps = {type = T_HashJoinState, plan = 0x1b73320, state = 0x1b78f48, ExecProcNode = 0x701efa <ExecHashJoin>, 
          ExecProcNodeReal = 0x0, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, qual = 0x0, 
          lefttree = 0x1b79860, righttree = 0x1b7a2b8, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
          ps_ResultTupleSlot = 0x1b7ad30, ps_ExprContext = 0x1b797a0, ps_ProjInfo = 0x1b85bf8, scandesc = 0x0}, 
        jointype = JOIN_INNER, single_match = false, joinqual = 0x0}, hashclauses = 0x1b86168, hj_OuterHashKeys = 0x1b86fc0, 
      hj_InnerHashKeys = 0x1b87730, hj_HashOperators = 0x1b87790, hj_HashTable = 0x0, hj_CurHashValue = 0, hj_CurBucketNo = 0, 
      hj_CurSkewBucketNo = -1, hj_CurTuple = 0x0, hj_OuterTupleSlot = 0x1b860a8, hj_HashTupleSlot = 0x1b856f8, 
      hj_NullOuterTupleSlot = 0x0, hj_NullInnerTupleSlot = 0x0, hj_FirstOuterTupleSlot = 0x0, hj_JoinState = 1, 
      hj_MatchedOuter = false, hj_OuterNotEmpty = false}
    

DONE!

### 四、参考资料

[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

