本节介绍了PortalStart->ExecutorStart(standard_ExecutorStart)->InitPlan函数的实现逻辑，该函数用于初始化查询执行计划。

### 一、数据结构

**EState**  
执行器在调用时的主要工作状态

    
    
    /* ----------------     *    EState information
     *    EState信息
     * Master working state for an Executor invocation
     * 执行器在调用时的主要工作状态
     * ----------------     */
    typedef struct EState
    {
        NodeTag     type;//标记
    
        /* Basic state for all query types: */
        //所有查询类型的基础状态
        ScanDirection es_direction; /* 扫描方向,如向前/后等;current scan direction */
        Snapshot    es_snapshot;    /* 时间条件(通过快照实现);time qual to use */
        Snapshot    es_crosscheck_snapshot; /* RI的交叉检查时间条件;crosscheck time qual for RI */
        List       *es_range_table; /* RTE链表;List of RangeTblEntry */
        struct RangeTblEntry **es_range_table_array;    /* 与链表等价的数组;equivalent array */
        Index       es_range_table_size;    /* 数组大小;size of the range table arrays */
        Relation   *es_relations;   /* RTE中的Relation指针,如未Open则为NULL;
                                     * Array of per-range-table-entry Relation
                                     * pointers, or NULL if not yet opened */
        struct ExecRowMark **es_rowmarks;   /* ExecRowMarks指针数组;
                                             * Array of per-range-table-entry
                                             * ExecRowMarks, or NULL if none */
        PlannedStmt *es_plannedstmt;    /* 计划树的最顶层PlannedStmt;link to top of plan tree */
        const char *es_sourceText;  /* QueryDesc中的源文本;Source text from QueryDesc */
    
        JunkFilter *es_junkFilter;  /* 最顶层的JunkFilter;top-level junk filter, if any */
    
        /* If query can insert/delete tuples, the command ID to mark them with */
        //如查询可以插入/删除元组,这里记录了命令ID
        CommandId   es_output_cid;
    
        /* Info about target table(s) for insert/update/delete queries: */
        //insert/update/delete 目标表信息
        ResultRelInfo *es_result_relations; /* ResultRelInfos数组;array of ResultRelInfos */
        int         es_num_result_relations;    /* 数组大小;length of array */
        ResultRelInfo *es_result_relation_info; /* 当前活动的数组;currently active array elt */
    
        /*
         * Info about the partition root table(s) for insert/update/delete queries
         * targeting partitioned tables.  Only leaf partitions are mentioned in
         * es_result_relations, but we need access to the roots for firing
         * triggers and for runtime tuple routing.
         * 关于针对分区表的插入/更新/删除查询的分区根表的信息。
         * 在es_result_relations中只提到了叶子分区，但是需要访问根表来触发触发器和运行时元组路由。
         */
        ResultRelInfo *es_root_result_relations;    /* ResultRelInfos数组;array of ResultRelInfos */
        int         es_num_root_result_relations;   /* 数组大小;length of the array */
    
        /*
         * The following list contains ResultRelInfos created by the tuple routing
         * code for partitions that don't already have one.
         * 下面的链表包含元组路由代码为没有分区的分区创建的ResultRelInfos。
         */
        List       *es_tuple_routing_result_relations;
    
        /* Stuff used for firing triggers: */
        //用于触发触发器的信息
        List       *es_trig_target_relations;   /* 与触发器相关的ResultRelInfos数组;trigger-only ResultRelInfos */
        TupleTableSlot *es_trig_tuple_slot; /* 用于触发器输出的元组; for trigger output tuples */
        TupleTableSlot *es_trig_oldtup_slot;    /* 用于TriggerEnabled;for TriggerEnabled */
        TupleTableSlot *es_trig_newtup_slot;    /* 用于TriggerEnabled;for TriggerEnabled */
    
        /* Parameter info: */
        //参数信息
        ParamListInfo es_param_list_info;   /* 外部参数值; values of external params */
        ParamExecData *es_param_exec_vals;  /* 内部参数值; values of internal params */
    
        QueryEnvironment *es_queryEnv;  /* 查询环境; query environment */
    
        /* Other working state: */
        //其他工作状态
        MemoryContext es_query_cxt; /* EState所在的内存上下文;per-query context in which EState lives */
    
        List       *es_tupleTable;  /* TupleTableSlots链表;List of TupleTableSlots */
    
        uint64      es_processed;   /* 处理的元组数;# of tuples processed */
        Oid         es_lastoid;     /* 最后处理的oid(用于INSERT命令);last oid processed (by INSERT) */
    
        int         es_top_eflags;  /* 传递给ExecutorStart函数的eflags参数;eflags passed to ExecutorStart */
        int         es_instrument;  /* InstrumentOption标记;OR of InstrumentOption flags */
        bool        es_finished;    /* ExecutorFinish函数已完成则为T;true when ExecutorFinish is done */
    
        List       *es_exprcontexts;    /* EState中的ExprContexts链表;List of ExprContexts within EState */
    
        List       *es_subplanstates;   /* SubPlans的PlanState链表;List of PlanState for SubPlans */
    
        List       *es_auxmodifytables; /* 第二个ModifyTableStates链表;List of secondary ModifyTableStates */
    
        /*
         * this ExprContext is for per-output-tuple operations, such as constraint
         * checks and index-value computations.  It will be reset for each output
         * tuple.  Note that it will be created only if needed.
         * 这个ExprContext用于元组输出操作，例如约束检查和索引值计算。它在每个输出元组时重置。
         * 注意，只有在需要时才会创建它。
         */
        ExprContext *es_per_tuple_exprcontext;
    
        /*
         * These fields are for re-evaluating plan quals when an updated tuple is
         * substituted in READ COMMITTED mode.  es_epqTuple[] contains tuples that
         * scan plan nodes should return instead of whatever they'd normally
         * return, or NULL if nothing to return; es_epqTupleSet[] is true if a
         * particular array entry is valid; and es_epqScanDone[] is state to
         * remember if the tuple has been returned already.  Arrays are of size
         * es_range_table_size and are indexed by scan node scanrelid - 1.
         * 这些字段用于在读取提交模式中替换更新后的元组时重新评估计划的条件quals。
         * es_epqTuple[]数组包含扫描计划节点应该返回的元组，而不是它们通常返回的任何值，如果没有返回值，则为NULL;
         * 如果特定数组条目有效，则es_epqTupleSet[]为真;es_epqScanDone[]是状态，用来记住元组是否已经返回。
         * 数组大小为es_range_table_size，通过扫描节点scanrelid - 1进行索引。
         */
        HeapTuple  *es_epqTuple;    /* EPQ替换元组的数组;array of EPQ substitute tuples */
        bool       *es_epqTupleSet; /* 如EPQ元组已提供,则为T;true if EPQ tuple is provided */
        bool       *es_epqScanDone; /* 如EPQ元组已获取,则为T;true if EPQ tuple has been fetched */
    
        bool        es_use_parallel_mode;   /* 能否使用并行worker?can we use parallel workers? */
    
        /* The per-query shared memory area to use for parallel execution. */
        //用于并行执行的每个查询共享内存区域。
        struct dsa_area *es_query_dsa;
    
        /*
         * JIT information. es_jit_flags indicates whether JIT should be performed
         * and with which options.  es_jit is created on-demand when JITing is
         * performed.
         * JIT的信息。es_jit_flags指示是否应该执行JIT以及使用哪些选项。在执行JITing时，按需创建es_jit。
         *
         * es_jit_combined_instr is the the combined, on demand allocated,
         * instrumentation from all workers. The leader's instrumentation is kept
         * separate, and is combined on demand by ExplainPrintJITSummary().
         * es_jit_combined_instr是所有workers根据需要分配的组合工具。
         * leader的插装是独立的，根据需要由ExplainPrintJITSummary()组合。
         */
        int         es_jit_flags;
        struct JitContext *es_jit;
        struct JitInstrumentation *es_jit_worker_instr;
    } EState;
    
    

**PlanState**  
PlanState是所有PlanState-type节点的虚父类(请参照面向对象的虚类)

    
    
    /* ----------------     *      PlanState node
     *      PlanState节点
     * We never actually instantiate any PlanState nodes; this is just the common
     * abstract superclass for all PlanState-type nodes.
     * 实际上并没有实例化PlanState节点;
     * 这是所有PlanState-type节点的虚父类(请参照面向对象的虚类).
     * ----------------     */
    typedef struct PlanState
    {
        NodeTag     type;//节点类型
    
        Plan       *plan;           /* 绑定的计划节点;associated Plan node */
    
        EState     *state;          /* 在执行期,指向top-level计划的EState指针
                                     * at execution time, states of individual
                                     * nodes point to one EState for the whole
                                     * top-level plan */
    
        ExecProcNodeMtd ExecProcNode;   /* 执行返回下一个元组的函数;function to return next tuple */
        ExecProcNodeMtd ExecProcNodeReal;   /* 如果ExecProcNode是封装器,则这个变量是实际的函数;
                                             * actual function, if above is a
                                             * wrapper */
    
        Instrumentation *instrument;    /* 该节点可选的运行期统计信息;Optional runtime stats for this node */
        WorkerInstrumentation *worker_instrument;   /* 每个worker的instrumentation;per-worker instrumentation */
    
        /* Per-worker JIT instrumentation */
        //worker相应的JIT instrumentation
        struct SharedJitInstrumentation *worker_jit_instrument;
    
        /*
         * Common structural data for all Plan types.  These links to subsidiary
         * state trees parallel links in the associated plan tree (except for the
         * subPlan list, which does not exist in the plan tree).
         * 所有Plan类型均有的数据结构;
         * 这些链接到附属状态通过并行的方式链接到关联的计划树中(除了计划树中不存在的子计划链表)。
         */
        ExprState  *qual;           /* 布尔条件;boolean qual condition */
        struct PlanState *lefttree; /* 输入的计划树,包括左树&右树;input plan tree(s) */
        struct PlanState *righttree;
    
        List       *initPlan;       /* 初始的SubPlanState节点链表;
                                     * Init SubPlanState nodes (un-correlated expr
                                     * subselects) */
        List       *subPlan;        /* 在表达式中的subPlan链表;SubPlanState nodes in my expressions */
    
        /*
         * State for management of parameter-change-driven rescanning
         * 参数改变驱动的重新扫描管理状态
         */
        Bitmapset  *chgParam;       /* changed Params的IDs集合(用于NestLoop连接);set of IDs of changed Params */
    
        /*
         * Other run-time state needed by most if not all node types.
         * 大多数(如果不是所有)节点类型都需要的其他运行时状态。
         */
        TupleDesc ps_ResultTupleDesc;   /* 节点的返回类型;node's return type */
        TupleTableSlot *ps_ResultTupleSlot; /* 结果元组的slot;slot for my result tuples */
        ExprContext *ps_ExprContext;    /* 节点的表达式解析上下文;node's expression-evaluation context */
        ProjectionInfo *ps_ProjInfo;    /* 元组投影信息;info for doing tuple projection */
    
        /*
         * Scanslot's descriptor if known. This is a bit of a hack, but otherwise
         * it's hard for expression compilation to optimize based on the
         * descriptor, without encoding knowledge about all executor nodes.
         * Scanslot描述符。
         * 这有点笨拙，但是如果不编码关于所有执行器节点的知识，那么表达式编译就很难基于描述符进行优化。
         */
        TupleDesc   scandesc;
    } PlanState;
    
    

### 二、源码解读

InitPlan函数初始化查询执行计划:打开文件/分配存储空间并启动规则管理器.

    
    
    /* ----------------------------------------------------------------     *      InitPlan
     *
     *      Initializes the query plan: open files, allocate storage
     *      and start up the rule manager
     *      初始化查询执行计划:打开文件/分配存储空间并启动规则管理器
     * ----------------------------------------------------------------     */
    static void
    InitPlan(QueryDesc *queryDesc, int eflags)
    {
        CmdType     operation = queryDesc->operation;//命令类型
        PlannedStmt *plannedstmt = queryDesc->plannedstmt;//已规划的语句指针
        Plan       *plan = plannedstmt->planTree;//计划树
        List       *rangeTable = plannedstmt->rtable;//RTE链表
        EState     *estate = queryDesc->estate;//参见数据结构
        PlanState  *planstate;//参见数据结构
        TupleDesc   tupType;//参见数据结构
        ListCell   *l;
        int         i;
    
        /*
         * Do permissions checks
         * 权限检查
         */
        ExecCheckRTPerms(rangeTable, true);
    
        /*
         * initialize the node's execution state
         * 初始化节点执行状态
         */
        ExecInitRangeTable(estate, rangeTable);
    
        estate->es_plannedstmt = plannedstmt;
    
        /*
         * Initialize ResultRelInfo data structures, and open the result rels.
         * 初始化ResultRelInfo数据结构,打开结果rels
         */
        if (plannedstmt->resultRelations)//存在resultRelations
        {
            List       *resultRelations = plannedstmt->resultRelations;//结果Relation链表
            int         numResultRelations = list_length(resultRelations);//链表大小
            ResultRelInfo *resultRelInfos;//ResultRelInfo数组
            ResultRelInfo *resultRelInfo;//ResultRelInfo指针
    
            resultRelInfos = (ResultRelInfo *)
                palloc(numResultRelations * sizeof(ResultRelInfo));//分配空间
            resultRelInfo = resultRelInfos;//指针赋值
            foreach(l, resultRelations)//遍历链表
            {
                Index       resultRelationIndex = lfirst_int(l);
                Relation    resultRelation;
    
                resultRelation = ExecGetRangeTableRelation(estate,
                                                           resultRelationIndex);//获取结果Relation
                InitResultRelInfo(resultRelInfo,
                                  resultRelation,
                                  resultRelationIndex,
                                  NULL,
                                  estate->es_instrument);//初始化ResultRelInfo
                resultRelInfo++;//处理下一个ResultRelInfo
            }
            estate->es_result_relations = resultRelInfos;//赋值
            estate->es_num_result_relations = numResultRelations;
    
            /* es_result_relation_info is NULL except when within ModifyTable */
            //设置es_result_relation_info为NULL,除了ModifyTable
            estate->es_result_relation_info = NULL;
    
            /*
             * In the partitioned result relation case, also build ResultRelInfos
             * for all the partitioned table roots, because we will need them to
             * fire statement-level triggers, if any.
             * 在分区结果关系这种情况中，还为所有分区表根构建ResultRelInfos，
             * 因为我们需要它们触发语句级别的触发器(如果有的话)。
             */
            if (plannedstmt->rootResultRelations)//存在rootResultRelations(并行处理)
            {
                int         num_roots = list_length(plannedstmt->rootResultRelations);
    
                resultRelInfos = (ResultRelInfo *)
                    palloc(num_roots * sizeof(ResultRelInfo));
                resultRelInfo = resultRelInfos;
                foreach(l, plannedstmt->rootResultRelations)
                {
                    Index       resultRelIndex = lfirst_int(l);
                    Relation    resultRelDesc;
    
                    resultRelDesc = ExecGetRangeTableRelation(estate,
                                                              resultRelIndex);
                    InitResultRelInfo(resultRelInfo,
                                      resultRelDesc,
                                      resultRelIndex,
                                      NULL,
                                      estate->es_instrument);
                    resultRelInfo++;
                }
    
                estate->es_root_result_relations = resultRelInfos;
                estate->es_num_root_result_relations = num_roots;
            }
            else
            {
                estate->es_root_result_relations = NULL;
                estate->es_num_root_result_relations = 0;
            }
        }
        else//不存在resultRelations
        {
            /*
             * if no result relation, then set state appropriately
             * 无resultRelations,设置相应的信息为NULL或为0
             */
            estate->es_result_relations = NULL;
            estate->es_num_result_relations = 0;
            estate->es_result_relation_info = NULL;
            estate->es_root_result_relations = NULL;
            estate->es_num_root_result_relations = 0;
        }
    
        /*
         * Next, build the ExecRowMark array from the PlanRowMark(s), if any.
         * 下一步,利用PlanRowMark(s)创建ExecRowMark数组
         */
        if (plannedstmt->rowMarks)//如存在rowMarks
        {
            estate->es_rowmarks = (ExecRowMark **)
                palloc0(estate->es_range_table_size * sizeof(ExecRowMark *));
            foreach(l, plannedstmt->rowMarks)
            {
                PlanRowMark *rc = (PlanRowMark *) lfirst(l);
                Oid         relid;
                Relation    relation;
                ExecRowMark *erm;
    
                /* ignore "parent" rowmarks; they are irrelevant at runtime */
                if (rc->isParent)
                    continue;
    
                /* get relation's OID (will produce InvalidOid if subquery) */
                relid = exec_rt_fetch(rc->rti, estate)->relid;
    
                /* open relation, if we need to access it for this mark type */
                switch (rc->markType)
                {
                    case ROW_MARK_EXCLUSIVE:
                    case ROW_MARK_NOKEYEXCLUSIVE:
                    case ROW_MARK_SHARE:
                    case ROW_MARK_KEYSHARE:
                    case ROW_MARK_REFERENCE:
                        relation = ExecGetRangeTableRelation(estate, rc->rti);
                        break;
                    case ROW_MARK_COPY:
                        /* no physical table access is required */
                        relation = NULL;
                        break;
                    default:
                        elog(ERROR, "unrecognized markType: %d", rc->markType);
                        relation = NULL;    /* keep compiler quiet */
                        break;
                }
    
                /* Check that relation is a legal target for marking */
                if (relation)
                    CheckValidRowMarkRel(relation, rc->markType);
    
                erm = (ExecRowMark *) palloc(sizeof(ExecRowMark));
                erm->relation = relation;
                erm->relid = relid;
                erm->rti = rc->rti;
                erm->prti = rc->prti;
                erm->rowmarkId = rc->rowmarkId;
                erm->markType = rc->markType;
                erm->strength = rc->strength;
                erm->waitPolicy = rc->waitPolicy;
                erm->ermActive = false;
                ItemPointerSetInvalid(&(erm->curCtid));
                erm->ermExtra = NULL;
    
                Assert(erm->rti > 0 && erm->rti <= estate->es_range_table_size &&
                       estate->es_rowmarks[erm->rti - 1] == NULL);
    
                estate->es_rowmarks[erm->rti - 1] = erm;
            }
        }
    
        /*
         * Initialize the executor's tuple table to empty.
         * 初始化执行器的元组表为NULL
         */
        estate->es_tupleTable = NIL;
        estate->es_trig_tuple_slot = NULL;
        estate->es_trig_oldtup_slot = NULL;
        estate->es_trig_newtup_slot = NULL;
    
        /* mark EvalPlanQual not active */
        //标记EvalPlanQual为非活动模式
        estate->es_epqTuple = NULL;
        estate->es_epqTupleSet = NULL;
        estate->es_epqScanDone = NULL;
    
        /*
         * Initialize private state information for each SubPlan.  We must do this
         * before running ExecInitNode on the main query tree, since
         * ExecInitSubPlan expects to be able to find these entries.
         * 为每个子计划初始化私有状态信息。
         * 在主查询树上运行ExecInitNode之前，必须这样做，因为ExecInitSubPlan希望能够找到这些条目。
         */
        Assert(estate->es_subplanstates == NIL);
        i = 1;                      /* subplan索引计数从1开始;subplan indices count from 1 */
        foreach(l, plannedstmt->subplans)//遍历subplans
        {
            Plan       *subplan = (Plan *) lfirst(l);
            PlanState  *subplanstate;
            int         sp_eflags;
    
            /*
             * A subplan will never need to do BACKWARD scan nor MARK/RESTORE. If
             * it is a parameterless subplan (not initplan), we suggest that it be
             * prepared to handle REWIND efficiently; otherwise there is no need.
             * 子计划永远不需要执行向后扫描或标记/恢复。
             * 如果它是一个无参数的子计划(不是initplan)，建议它准备好有效地处理向后扫描;否则就没有必要了。
             */
            sp_eflags = eflags
                & (EXEC_FLAG_EXPLAIN_ONLY | EXEC_FLAG_WITH_NO_DATA);//设置sp_eflags
            if (bms_is_member(i, plannedstmt->rewindPlanIDs))
                sp_eflags |= EXEC_FLAG_REWIND;
    
            subplanstate = ExecInitNode(subplan, estate, sp_eflags);//执行Plan节点初始化过程
    
            estate->es_subplanstates = lappend(estate->es_subplanstates,
                                               subplanstate);
    
            i++;
        }
    
        /*
         * Initialize the private state information for all the nodes in the query
         * tree.  This opens files, allocates storage and leaves us ready to start
         * processing tuples.
         * 为查询树中的所有节点初始化私有状态信息。
         * 这将打开文件、分配存储并让我们准备好开始处理元组。
         */
        planstate = ExecInitNode(plan, estate, eflags);//执行Plan节点初始化过程
    
        /*
         * Get the tuple descriptor describing the type of tuples to return.
         * 获取元组描述(返回的元组类型等)
         */
        tupType = ExecGetResultType(planstate);
    
        /*
         * Initialize the junk filter if needed.  SELECT queries need a filter if
         * there are any junk attrs in the top-level tlist.
         * 如需要，初始化垃圾过滤器。如果顶层的tlist中有任何垃圾标识，SELECT查询需要一个过滤器。
         */
        if (operation == CMD_SELECT)//SELECT命令
        {
            bool        junk_filter_needed = false;
            ListCell   *tlist;
    
            foreach(tlist, plan->targetlist)//遍历tlist
            {
                TargetEntry *tle = (TargetEntry *) lfirst(tlist);
    
                if (tle->resjunk)//如需要垃圾过滤器
                {
                    junk_filter_needed = true;//设置为T
                    break;
                }
            }
    
            if (junk_filter_needed)
            {
                JunkFilter *j;
    
                j = ExecInitJunkFilter(planstate->plan->targetlist,
                                       tupType->tdhasoid,
                                       ExecInitExtraTupleSlot(estate, NULL));//初始化
                estate->es_junkFilter = j;
    
                /* Want to return the cleaned tuple type */
                //期望返回已清理的元组类型
                tupType = j->jf_cleanTupType;
            }
        }
        //赋值
        queryDesc->tupDesc = tupType;
        queryDesc->planstate = planstate;
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
    
    

启动gdb,设置断点,进入InitPlan

    
    
    (gdb) b InitPlan
    Breakpoint 1 at 0x6d9df2: file execMain.c, line 808.
    (gdb) c
    Continuing.
    
    Breakpoint 1, InitPlan (queryDesc=0x20838b8, eflags=16) at execMain.c:808
    warning: Source file is more recent than executable.
    808     CmdType     operation = queryDesc->operation;
    

输入参数

    
    
    (gdb) p *queryDesc
    $1 = {operation = CMD_SELECT, plannedstmt = 0x207e6c0, 
      sourceText = 0x1f96eb8 "select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je \nfrom t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je \n", ' ' <repeats 24 times>, "from t_grxx gr inner join t_jfxx jf \n", ' ' <repeats 34 times>..., 
      snapshot = 0x1fba8e0, crosscheck_snapshot = 0x0, dest = 0xf8f280 <donothingDR>, params = 0x0, queryEnv = 0x0, 
      instrument_options = 0, tupDesc = 0x0, estate = 0x207f898, planstate = 0x0, already_executed = false, totaltime = 0x0}
    (gdb) p eflags
    $2 = 16
    

变量赋值,PlannedStmt的详细数据结构先前章节已有解释,请参见相关章节.

    
    
    (gdb) n
    809     PlannedStmt *plannedstmt = queryDesc->plannedstmt;
    (gdb) 
    810     Plan       *plan = plannedstmt->planTree;
    (gdb) 
    811     List       *rangeTable = plannedstmt->rtable;
    (gdb) n
    812     EState     *estate = queryDesc->estate;
    (gdb) 
    821     ExecCheckRTPerms(rangeTable, true);
    (gdb) p *plannedstmt
    $3 = {type = T_PlannedStmt, commandType = CMD_SELECT, queryId = 0, hasReturning = false, hasModifyingCTE = false, 
      canSetTag = true, transientPlan = false, dependsOnRole = false, parallelModeNeeded = false, jitFlags = 0, 
      planTree = 0x20788f8, rtable = 0x207c568, resultRelations = 0x0, nonleafResultRelations = 0x0, rootResultRelations = 0x0, 
      subplans = 0x0, rewindPlanIDs = 0x0, rowMarks = 0x0, relationOids = 0x207c5c8, invalItems = 0x0, paramExecTypes = 0x0, 
      utilityStmt = 0x0, stmt_location = 0, stmt_len = 313}
    (gdb) p *plan
    $4 = {type = T_Sort, startup_cost = 20070.931487218411, total_cost = 20320.931487218411, plan_rows = 100000, 
      plan_width = 47, parallel_aware = false, parallel_safe = true, plan_node_id = 0, targetlist = 0x207cc28, qual = 0x0, 
      lefttree = 0x207c090, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    (gdb) p *estate
    $5 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x1fba8e0, es_crosscheck_snapshot = 0x0, 
      es_range_table = 0x0, es_plannedstmt = 0x0, 
      es_sourceText = 0x1f96eb8 "select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je \nfrom t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je \n", ' ' <repeats 24 times>, "from t_grxx gr inner join t_jfxx jf \n", ' ' <repeats 34 times>..., 
      es_junkFilter = 0x0, es_output_cid = 0, es_result_relations = 0x0, es_num_result_relations = 0, 
      es_result_relation_info = 0x0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, 
      es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, es_trig_tuple_slot = 0x0, 
      es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, es_param_exec_vals = 0x0, 
      es_queryEnv = 0x0, es_query_cxt = 0x207f780, es_tupleTable = 0x0, es_rowMarks = 0x0, es_processed = 0, es_lastoid = 0, 
      es_top_eflags = 16, es_instrument = 0, es_finished = false, es_exprcontexts = 0x0, es_subplanstates = 0x0, 
      es_auxmodifytables = 0x0, es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, es_epqTupleSet = 0x0, es_epqScanDone = 0x0, 
      es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, es_jit = 0x0, es_jit_worker_instr = 0x0}
    

RTE链表,一共有5个Item,3个基础关系RTE_RELATION(1/3/4),1个RTE_SUBQUERY(2号),1个RTE_JOIN(5号)

    
    
    (gdb) n
    826     estate->es_range_table = rangeTable;
    (gdb) p *rangeTable
    $6 = {type = T_List, length = 5, head = 0x207c540, tail = 0x207cb28}
    (gdb) p *(RangeTblEntry *)rangeTable->head->data.ptr_value
    $9 = {type = T_RangeTblEntry, rtekind = RTE_RELATION, relid = 16734, relkind = 114 'r', tablesample = 0x0, subquery = 0x0, 
      security_barrier = false, jointype = JOIN_INNER, joinaliasvars = 0x0, functions = 0x0, funcordinality = false, 
      tablefunc = 0x0, values_lists = 0x0, ctename = 0x0, ctelevelsup = 0, self_reference = false, coltypes = 0x0, 
      coltypmods = 0x0, colcollations = 0x0, enrname = 0x0, enrtuples = 0, alias = 0x1f98448, eref = 0x1fbdd20, 
      lateral = false, inh = false, inFromCl = true, requiredPerms = 2, checkAsUser = 0, selectedCols = 0x2054de8, 
      insertedCols = 0x0, updatedCols = 0x0, securityQuals = 0x0}
    ...
    

没有结果Relation,执行相应的处理逻辑

    
    
    (gdb) 
    835     if (plannedstmt->resultRelations)
    (gdb) p plannedstmt->resultRelations
    $14 = (List *) 0x0
    (gdb) n
    922         estate->es_result_relations = NULL;
    (gdb) 
    923         estate->es_num_result_relations = 0;
    (gdb) 
    924         estate->es_result_relation_info = NULL;
    (gdb) 
    925         estate->es_root_result_relations = NULL;
    (gdb) 
    926         estate->es_num_root_result_relations = 0;
    

不存在rowMarks(for update)

    
    
    (gdb) n
    939     foreach(l, plannedstmt->rowMarks)
    (gdb) p plannedstmt->rowMarks
    $15 = (List *) 0x0
    (gdb) p plannedstmt->rowMarks
    $16 = (List *) 0x0
    (gdb) n
    1000        estate->es_tupleTable = NIL;
    

执行赋值

    
    
    (gdb) n
    1001        estate->es_trig_tuple_slot = NULL;
    (gdb) 
    1002        estate->es_trig_oldtup_slot = NULL;
    ...
    

没有subplans

    
    
    (gdb) n
    1017        foreach(l, plannedstmt->subplans)
    (gdb) p *plannedstmt->subplans
    Cannot access memory at address 0x0
    (gdb) n
    

初始化节点(执行ExecInitNode)和获取TupleType

    
    
    (gdb) n
    1046        planstate = ExecInitNode(plan, estate, eflags);
    (gdb) n
    1051        tupType = ExecGetResultType(planstate);
    (gdb) 
    1057        if (operation == CMD_SELECT)
    (gdb) p *planstate
    $17 = {type = T_SortState, plan = 0x20788f8, state = 0x207f898, ExecProcNode = 0x6e41bb <ExecProcNodeFirst>, 
      ExecProcNodeReal = 0x716144 <ExecSort>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
      qual = 0x0, lefttree = 0x207fbc8, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
      ps_ResultTupleSlot = 0x2090dc0, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, scandesc = 0x208e920}
    (gdb) p tupType
    $18 = (TupleDesc) 0x20909a8
    (gdb) p *tupType
    $19 = {natts = 7, tdtypeid = 2249, tdtypmod = -1, tdhasoid = false, tdrefcount = -1, constr = 0x0, attrs = 0x20909c8}
    

判断是否需要垃圾过滤器

    
    
    (gdb) n
    1059            bool        junk_filter_needed = false;
    (gdb) 
    1062            foreach(tlist, plan->targetlist)
    

不需要junk filter

    
    
    (gdb) 
    1073            if (junk_filter_needed)
    

赋值,结束调用

    
    
    (gdb) 
    1087        queryDesc->tupDesc = tupType;
    (gdb) 
    1088        queryDesc->planstate = planstate;
    (gdb) 
    1089    }
    (gdb) 
    standard_ExecutorStart (queryDesc=0x20838b8, eflags=16) at execMain.c:267
    267     MemoryContextSwitchTo(oldcontext);
    (gdb) p *queryDesc
    $21 = {operation = CMD_SELECT, plannedstmt = 0x207e6c0, 
      sourceText = 0x1f96eb8 "select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je \nfrom t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je \n", ' ' <repeats 24 times>, "from t_grxx gr inner join t_jfxx jf \n", ' ' <repeats 34 times>..., 
      snapshot = 0x1fba8e0, crosscheck_snapshot = 0x0, dest = 0xf8f280 <donothingDR>, params = 0x0, queryEnv = 0x0, 
      instrument_options = 0, tupDesc = 0x20909a8, estate = 0x207f898, planstate = 0x207fab0, already_executed = false, 
      totaltime = 0x0}
    

DONE!

### 四、参考资料

[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

