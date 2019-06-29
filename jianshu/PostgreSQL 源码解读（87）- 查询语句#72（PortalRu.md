本节介绍了PortalRunSelect->ExecutorRun->ExecutePlan函数以及ExecProcNode的其中一个Real函数(ExecSeqScan)。ExecutePlan函数处理查询计划，直到检索到指定数量(参数numbertuple)的元组，并沿着指定的方向扫描。ExecSeqScan函数顺序扫描relation,返回下一个符合条件的元组。

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

**ExecutePlan**  
PortalRunSelect->ExecutorRun->ExecutePlan函数处理查询计划，直到检索到指定数量(参数numbertuple)的元组，并沿着指定的方向扫描.

    
    
    
    /* ----------------------------------------------------------------     *      ExecutePlan
     *
     *      Processes the query plan until we have retrieved 'numberTuples' tuples,
     *      moving in the specified direction.
     *      处理查询计划，直到检索到指定数量(参数numbertuple)的元组，并沿着指定的方向移动。
     *
     *      Runs to completion if numberTuples is 0
     *      如参数numbertuple为0,则运行至结束为止
     *
     * Note: the ctid attribute is a 'junk' attribute that is removed before the
     * user can see it
     * 注意:ctid属性是"junk"属性,在返回给用户前会移除
     * ----------------------------------------------------------------     */
    static void
    ExecutePlan(EState *estate,//执行状态
                PlanState *planstate,//计划状态
                bool use_parallel_mode,//是否使用并行模式
                CmdType operation,//操作类型
                bool sendTuples,//是否需要传输元组
                uint64 numberTuples,//元组数量
                ScanDirection direction,//扫描方向
                DestReceiver *dest,//接收的目标端
                bool execute_once)//是否只执行一次
    {
        TupleTableSlot *slot;//元组表Slot
        uint64      current_tuple_count;//当前的元组计数
    
        /*
         * initialize local variables
         * 初始化本地变量
         */
        current_tuple_count = 0;
    
        /*
         * Set the direction.
         * 设置扫描方向
         */
        estate->es_direction = direction;
    
        /*
         * If the plan might potentially be executed multiple times, we must force
         * it to run without parallelism, because we might exit early.
         * 如果计划可能被多次执行，那么必须强制它在非并行的情况下运行，因为可能会提前退出。
         */
        if (!execute_once)
            use_parallel_mode = false;//如需多次执行,则不允许并行执行
    
        estate->es_use_parallel_mode = use_parallel_mode;
        if (use_parallel_mode)
            EnterParallelMode();//如并行,则进入并行模式
    
        /*
         * Loop until we've processed the proper number of tuples from the plan.
         * 循环直至执行计划已处理完成相应数量的元组
         * 注意:每次循环只处理一个元组,每次都要重置元组Expr的上下文/过滤不需要的列/发送元组
         */
        for (;;)
        {
            /* Reset the per-output-tuple exprcontext */
            //重置Expr上下文
            ResetPerTupleExprContext(estate);
    
            /*
             * Execute the plan and obtain a tuple
             * 执行计划,获取一个元组
             */
            slot = ExecProcNode(planstate);
    
            /*
             * if the tuple is null, then we assume there is nothing more to
             * process so we just end the loop...
             * 如果返回的元组为空，那么可以认为没有什么要处理的了，结束循环……
             */
            if (TupIsNull(slot))
            {
                /*
                 * If we know we won't need to back up, we can release resources
                 * at this point.
                 * 如果已知不需要备份(回溯),那么可以释放资源了
                 */
                if (!(estate->es_top_eflags & EXEC_FLAG_BACKWARD))
                    (void) ExecShutdownNode(planstate);
                break;
            }
    
            /*
             * If we have a junk filter, then project a new tuple with the junk
             * removed.
             * 如有junk过滤器,使用junk执行投影操作,产生一个新的元组
             * 
             * Store this new "clean" tuple in the junkfilter's resultSlot.
             * (Formerly, we stored it back over the "dirty" tuple, which is WRONG
             * because that tuple slot has the wrong descriptor.)
             * 将这个新的“clean”元组存储在junkfilter的resultSlot中。
             * (以前，将其存储在“dirty” tuple上，这是错误的，因为该tuple slot的描述符是错误的。)
             */
            if (estate->es_junkFilter != NULL)
                slot = ExecFilterJunk(estate->es_junkFilter, slot);
    
            /*
             * If we are supposed to send the tuple somewhere, do so. (In
             * practice, this is probably always the case at this point.)
             * 如果要将元组发送到某个地方(接收器)，那么就这样做。
             * (实际上，在这一点上可能总是如此。)
             */
            if (sendTuples)
            {
                /*
                 * If we are not able to send the tuple, we assume the destination
                 * has closed and no more tuples can be sent. If that's the case,
                 * end the loop.
                 * 如果不能发送元组，有理由假设目的接收器已经关闭，不能发送更多元组，结束循环。
                 */
                if (!dest->receiveSlot(slot, dest))
                    break;//跳出循环
            }
    
            /*
             * Count tuples processed, if this is a SELECT.  (For other operation
             * types, the ModifyTable plan node must count the appropriate
             * events.)
             * 如果操作类型为CMD_SELECT，则计算已处理的元组。
             * (对于其他操作类型，ModifyTable plan节点必须统计合适的事件。)
             */
            if (operation == CMD_SELECT)
                (estate->es_processed)++;
    
            /*
             * check our tuple count.. if we've processed the proper number then
             * quit, else loop again and process more tuples.  Zero numberTuples
             * means no limit.
             * 检查处理的元组计数…
             * 如果已完成处理，那么退出，否则再次循环并处理更多元组。
             * 注意:numberTuples=0表示没有限制。
             */
            current_tuple_count++;
            if (numberTuples && numberTuples == current_tuple_count)
            {
                /*
                 * If we know we won't need to back up, we can release resources
                 * at this point.
                 * 不需要回溯，可以在此时释放资源。
                 */
                if (!(estate->es_top_eflags & EXEC_FLAG_BACKWARD))
                    (void) ExecShutdownNode(planstate);
                break;
            }
        }
    
        if (use_parallel_mode)
            ExitParallelMode();//退出并行模式
    }
    
    
    /* ----------------------------------------------------------------     *      ExecProcNode
     *
     *      Execute the given node to return a(nother) tuple.
     *      调用node->ExecProcNode函数返回元组(one or another)
     * ----------------------------------------------------------------     */
    #ifndef FRONTEND
    static inline TupleTableSlot *
    ExecProcNode(PlanState *node)
    {
        if (node->chgParam != NULL) /* 参数变化?something changed? */
            ExecReScan(node);       /* 调用ExecReScan函数;let ReScan handle this */
    
        return node->ExecProcNode(node);//执行ExecProcNode
    }
    #endif
    
    

**ExecSeqScan**  
ExecSeqScan函数顺序扫描relation,返回下一个符合条件的元组。

    
    
    /* ----------------------------------------------------------------     *      ExecSeqScan(node)
     *      
     *      Scans the relation sequentially and returns the next qualifying
     *      tuple.
     *      We call the ExecScan() routine and pass it the appropriate
     *      access method functions.
     *      顺序扫描relation,返回下一个符合条件的元组。
     *      调用ExecScan函数,传入相应的访问方法函数
     * ----------------------------------------------------------------     */
    static TupleTableSlot *
    ExecSeqScan(PlanState *pstate)
    {
        SeqScanState *node = castNode(SeqScanState, pstate);//获取SeqScanState
    
        return ExecScan(&node->ss,
                        (ExecScanAccessMtd) SeqNext,
                        (ExecScanRecheckMtd) SeqRecheck);//执行Scan
    }
    
    /* ----------------------------------------------------------------     *      ExecScan
     *
     *      Scans the relation using the 'access method' indicated and
     *      returns the next qualifying tuple in the direction specified
     *      in the global variable ExecDirection.
     *      The access method returns the next tuple and ExecScan() is
     *      responsible for checking the tuple returned against the qual-clause.
     *      使用指定的“访问方法”扫描关系，并按照全局变量ExecDirection中指定的方向返回下一个符合条件的元组。
     *      访问方法返回下一个元组，ExecScan()负责根据qual-clause条件子句检查返回的元组是否符合条件。
     *
     *      A 'recheck method' must also be provided that can check an
     *      arbitrary tuple of the relation against any qual conditions
     *      that are implemented internal to the access method.
     *      调用者还必须提供“recheck method”，根据访问方法内部实现的条件检查关系的所有元组。
     *
     *      Conditions:
     *        -- the "cursor" maintained by the AMI is positioned at the tuple
     *           returned previously.
     *      前提条件:
     *        由AMI负责维护的游标已由先前的处理过程定位.
     *
     *      Initial States:
     *        -- the relation indicated is opened for scanning so that the
     *           "cursor" is positioned before the first qualifying tuple.
     *      初始状态:
     *        在游标可定位返回第一个符合条件的元组前,relation已打开可进行扫描
     * ----------------------------------------------------------------     */
    TupleTableSlot *
    ExecScan(ScanState *node,
             ExecScanAccessMtd accessMtd,   /* 返回元组的访问方法;function returning a tuple */
             ExecScanRecheckMtd recheckMtd) //recheck方法
    {
        ExprContext *econtext;//表达式上下文
        ExprState  *qual;//表达式状态
        ProjectionInfo *projInfo;//投影信息
    
        /*
         * Fetch data from node
         * 从node中提取数据
         */
        qual = node->ps.qual;
        projInfo = node->ps.ps_ProjInfo;
        econtext = node->ps.ps_ExprContext;
    
        /* interrupt checks are in ExecScanFetch */
        //在ExecScanFetch中有中断检查
    
        /*
         * If we have neither a qual to check nor a projection to do, just skip
         * all the overhead and return the raw scan tuple.
         * 如果既没有要检查的条件qual,也没有要做的投影操作，那么就跳过所有的操作并返回raw scan元组。
         */
        if (!qual && !projInfo)
        {
            ResetExprContext(econtext);
            return ExecScanFetch(node, accessMtd, recheckMtd);
        }
    
        /*
         * Reset per-tuple memory context to free any expression evaluation
         * storage allocated in the previous tuple cycle.
         * 重置每个元组内存上下文，以释放用于在前一个元组循环中分配的表达式求值内存空间。
         */
        ResetExprContext(econtext);
    
        /*
         * get a tuple from the access method.  Loop until we obtain a tuple that
         * passes the qualification.
         * 从访问方法中获取一个元组。循环，直到获得通过限定条件的元组。
         */
        for (;;)
        {
            TupleTableSlot *slot;//slot变量
    
            slot = ExecScanFetch(node, accessMtd, recheckMtd);//获取slot
    
            /*
             * if the slot returned by the accessMtd contains NULL, then it means
             * there is nothing more to scan so we just return an empty slot,
             * being careful to use the projection result slot so it has correct
             * tupleDesc.
             * 如果accessMtd方法返回的slot中包含NULL，那么这意味着不再需要扫描了，
             * 这时候只需要返回一个空slot，小心使用投影结果slot，这样可以有正确的tupleDesc了。
             */
            if (TupIsNull(slot))
            {
                if (projInfo)
                    return ExecClearTuple(projInfo->pi_state.resultslot);
                else
                    return slot;
            }
    
            /*
             * place the current tuple into the expr context
             * 把当前tuple放入到expr上下文中
             */
            econtext->ecxt_scantuple = slot;
    
            /*
             * check that the current tuple satisfies the qual-clause
             * 检查当前的tuple是否符合qual-clause条件
             * 
             * check for non-null qual here to avoid a function call to ExecQual()
             * when the qual is null ... saves only a few cycles, but they add up
             * ...
             * 在这里检查qual是否非空，以避免在qual为空时调用ExecQual()函数…
             * 只节省了几个调用周期，但它们加起来……的成本还是蛮可观的
             */
            if (qual == NULL || ExecQual(qual, econtext))
            {
                /*
                 * Found a satisfactory scan tuple.
                 * 发现一个满足条件的元组
                 */
                if (projInfo)
                {
                    /*
                     * Form a projection tuple, store it in the result tuple slot
                     * and return it.
                     * 构造一个投影元组,存储在结果元组slot中并返回
                     */
                    return ExecProject(projInfo);//执行投影操作并返回
                }
                else
                {
                    /*
                     * Here, we aren't projecting, so just return scan tuple.
                     * 不需要执行投影操作,返回元组
                     */
                    return slot;//直接返回
                }
            }
            else
                InstrCountFiltered1(node, 1);//instrument计数
    
            /*
             * Tuple fails qual, so free per-tuple memory and try again.
             * 元组不满足条件,释放资源,重试
             */
            ResetExprContext(econtext);
        }
    }
    
    
    /*
     * ExecScanFetch -- check interrupts & fetch next potential tuple
     * ExecScanFetch -- 检查中断&提前下一个备选元组
     *
     * This routine is concerned with substituting a test tuple if we are
     * inside an EvalPlanQual recheck.  If we aren't, just execute
     * the access method's next-tuple routine.
     * 这个例程是处理测试元组的替换(如果在EvalPlanQual重新检查中)。
     * 如果不是在EvalPlanQual中，则执行access方法的next-tuple例程。
     */
    static inline TupleTableSlot *
    ExecScanFetch(ScanState *node,
                  ExecScanAccessMtd accessMtd,
                  ExecScanRecheckMtd recheckMtd)
    {
        EState     *estate = node->ps.state;
    
        CHECK_FOR_INTERRUPTS();//检查中断
    
        if (estate->es_epqTuple != NULL)//如es_epqTuple不为NULL()
        {
            //es_epqTuple字段用于在READ COMMITTED模式中替换更新后的元组后,重新评估是否满足执行计划的条件quals
            /*
             * We are inside an EvalPlanQual recheck.  Return the test tuple if
             * one is available, after rechecking any access-method-specific
             * conditions.
             * 我们正在EvalPlanQual复查。
             * 如果test tuple可用，则在重新检查所有特定于访问方法的条件后返回该元组。
             */
            Index       scanrelid = ((Scan *) node->ps.plan)->scanrelid;//访问的relid
    
            if (scanrelid == 0)//relid==0
            {
                TupleTableSlot *slot = node->ss_ScanTupleSlot;
    
                /*
                 * This is a ForeignScan or CustomScan which has pushed down a
                 * join to the remote side.  The recheck method is responsible not
                 * only for rechecking the scan/join quals but also for storing
                 * the correct tuple in the slot.
                 * 这是一个ForeignScan或CustomScan，它将下推到远程端。
                 * recheck方法不仅负责重新检查扫描/连接quals，还负责在slot中存储正确的元组。
                 */
                if (!(*recheckMtd) (node, slot))
                    ExecClearTuple(slot);   /* 验证不通过,释放资源,不返回元组;would not be returned by scan */
                return slot;
            }
            else if (estate->es_epqTupleSet[scanrelid - 1])//从estate->es_epqTupleSet数组中获取标志
            {
                TupleTableSlot *slot = node->ss_ScanTupleSlot;//获取slot
    
                /* Return empty slot if we already returned a tuple */
                //如已返回元组,则清空slot
                if (estate->es_epqScanDone[scanrelid - 1])
                    return ExecClearTuple(slot);
                /* Else mark to remember that we shouldn't return more */
                //否则,标记没有返回
                estate->es_epqScanDone[scanrelid - 1] = true;
    
                /* Return empty slot if we haven't got a test tuple */
                //如test tuple为NULL,则清空slot
                if (estate->es_epqTuple[scanrelid - 1] == NULL)
                    return ExecClearTuple(slot);
    
                /* Store test tuple in the plan node's scan slot */
                //在计划节点的scan slot中存储test tuple
                ExecStoreHeapTuple(estate->es_epqTuple[scanrelid - 1],
                                   slot, false);
    
                /* Check if it meets the access-method conditions */
                //检查是否满足访问方法条件
                if (!(*recheckMtd) (node, slot))
                    ExecClearTuple(slot);   /* 不满足,清空slot;would not be returned by scan */
    
                return slot;
            }
        }
    
        /*
         * Run the node-type-specific access method function to get the next tuple
         * 运行node-type-specific方法函数,获取下一个tuple
         */
        return (*accessMtd) (node);
    }
    
    
    /*
     * ExecProject
     *
     * Projects a tuple based on projection info and stores it in the slot passed
     * to ExecBuildProjectInfo().
     * 根据投影信息投影一个元组，并将其存储在传递给ExecBuildProjectInfo()的slot中。
     *
     * Note: the result is always a virtual tuple; therefore it may reference
     * the contents of the exprContext's scan tuples and/or temporary results
     * constructed in the exprContext.  If the caller wishes the result to be
     * valid longer than that data will be valid, he must call ExecMaterializeSlot
     * on the result slot.
     * 注意:结果总是一个虚拟元组;
     * 因此，它可以引用exprContext的扫描元组和/或exprContext中构造的临时结果的内容。
     * 如果调用者希望结果有效的时间长于数据有效的时间，必须在结果slot上调用ExecMaterializeSlot。
     */
    #ifndef FRONTEND
    static inline TupleTableSlot *
    ExecProject(ProjectionInfo *projInfo)
    {
        ExprContext *econtext = projInfo->pi_exprContext;
        ExprState  *state = &projInfo->pi_state;
        TupleTableSlot *slot = state->resultslot;
        bool        isnull;
    
        /*
         * Clear any former contents of the result slot.  This makes it safe for
         * us to use the slot's Datum/isnull arrays as workspace.
         * 清除以前的结果slot内容。
         * 这使得我们可以安全地使用slot的Datum/isnull数组作为工作区。
         */
        ExecClearTuple(slot);
    
        /* Run the expression, discarding scalar result from the last column. */
        //运行表达式，从最后一列丢弃scalar结果。
        (void) ExecEvalExprSwitchContext(state, econtext, &isnull);
    
        /*
         * Successfully formed a result row.  Mark the result slot as containing a
         * valid virtual tuple (inlined version of ExecStoreVirtualTuple()).
         * 成功形成了一个结果行。
         * 将结果slot标记为包含一个有效的虚拟元组(ExecStoreVirtualTuple()的内联版本)。
         */
        slot->tts_flags &= ~TTS_FLAG_EMPTY;
        slot->tts_nvalid = slot->tts_tupleDescriptor->natts;
    
        return slot;
    }
    #endif
    
    /*
     * ExecQual - evaluate a qual prepared with ExecInitQual (possibly via
     * ExecPrepareQual).  Returns true if qual is satisfied, else false.
     * 解析用ExecInitQual准备的条件qual(可能通过ExecPrepareQual)。
     * 如果满足条件qual，返回true，否则为false。
     * 
     * Note: ExecQual used to have a third argument "resultForNull".  The
     * behavior of this function now corresponds to resultForNull == false.
     * If you want the resultForNull == true behavior, see ExecCheck.
     * 注意:ExecQual曾经有第三个参数“resultForNull”。
     * 这个函数的行为现在对应于resultForNull == false。
     * 如果希望resultForNull == true行为，请参阅ExecCheck。
     */
    #ifndef FRONTEND
    static inline bool
    ExecQual(ExprState *state, ExprContext *econtext)
    {
        Datum       ret;
        bool        isnull;
    
        /* short-circuit (here and in ExecInitQual) for empty restriction list */
        //如state为NULL,直接返回
        if (state == NULL)
            return true;
    
        /* verify that expression was compiled using ExecInitQual */
        //使用函数ExecInitQual验证表达式是否可以编译
        Assert(state->flags & EEO_FLAG_IS_QUAL);
    
        ret = ExecEvalExprSwitchContext(state, econtext, &isnull);
    
        /* EEOP_QUAL不应返回NULL;EEOP_QUAL should never return NULL */
        Assert(!isnull);
    
        return DatumGetBool(ret);
    }
    #endif
    
    
    
    /* --------------------------------     *      ExecClearTuple
     *
     *      This function is used to clear out a slot in the tuple table.
     *      该函数清空tuple table中的slot
     *      NB: only the tuple is cleared, not the tuple descriptor (if any).
     *      注意:只有tuple被清除,而不是tuple描述符
     * --------------------------------     */
    TupleTableSlot *                /* 返回验证通过的slot;return: slot passed */
    ExecClearTuple(TupleTableSlot *slot)    /* 存储tuple的slot;slot in which to store tuple */
    {
        /*
         * sanity checks
         * 安全检查
         */
        Assert(slot != NULL);
    
        /*
         * Free the old physical tuple if necessary.
         * 如需要,释放原有的物理元组
         */
        if (TTS_SHOULDFREE(slot))
        {
            heap_freetuple(slot->tts_tuple);//释放元组
            slot->tts_flags &= ~TTS_FLAG_SHOULDFREE;
        }
        if (TTS_SHOULDFREEMIN(slot))
        {
            heap_free_minimal_tuple(slot->tts_mintuple);
            slot->tts_flags &= ~TTS_FLAG_SHOULDFREEMIN;
        }
    
        slot->tts_tuple = NULL;//设置NULL值
        slot->tts_mintuple = NULL;
    
        /*
         * Drop the pin on the referenced buffer, if there is one.
         * 如果有的话，将pin放在已引用的缓冲区上。
         */
        if (BufferIsValid(slot->tts_buffer))
            ReleaseBuffer(slot->tts_buffer);//释放缓冲区
    
        slot->tts_buffer = InvalidBuffer;
    
        /*
         * Mark it empty.
         * 标记为空
         */
        slot->tts_flags |= TTS_FLAG_EMPTY;
        slot->tts_nvalid = 0;
    
        return slot;
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
    
    

启动gdb,设置断点,进入ExecutePlan

    
    
    (gdb) b ExecutePlan
    Breakpoint 1 at 0x6db79d: file execMain.c, line 1694.
    (gdb) c
    Continuing.
    
    Breakpoint 1, ExecutePlan (estate=0x14daf48, planstate=0x14db160, use_parallel_mode=false, operation=CMD_SELECT, 
        sendTuples=true, numberTuples=0, direction=ForwardScanDirection, dest=0x14d9ed0, execute_once=true) at execMain.c:1694
    warning: Source file is more recent than executable.
    1694        current_tuple_count = 0;
    

查看输入参数  
planstate->type:T_SortState->排序Plan  
planstate->ExecProcNode:ExecProcNodeFirst,封装器  
planstate->ExecProcNodeReal:ExecSort,实际的函数  
use_parallel_mode:false,非并行模式  
operation:CMD_SELECT,查询操作  
sendTuples:T,需要发送元组给客户端  
numberTuples:0,所有元组  
direction:ForwardScanDirection  
dest:printtup(console客户端)  
execute_once:T,只执行一次

    
    
    (gdb) p *estate
    $1 = {type = T_EState, es_direction = ForwardScanDirection, es_snapshot = 0x1493e10, es_crosscheck_snapshot = 0x0, 
      es_range_table = 0x14d7c00, es_plannedstmt = 0x14d9d58, 
      es_sourceText = 0x13eeeb8 "select dw.*,grjf.grbh,grjf.xm,grjf.ny,grjf.je \nfrom t_dwxx dw,lateral (select gr.grbh,gr.xm,jf.ny,jf.je \n", ' ' <repeats 24 times>, "from t_grxx gr inner join t_jfxx jf \n", ' ' <repeats 34 times>..., 
      es_junkFilter = 0x0, es_output_cid = 0, es_result_relations = 0x0, es_num_result_relations = 0, 
      es_result_relation_info = 0x0, es_root_result_relations = 0x0, es_num_root_result_relations = 0, 
      es_tuple_routing_result_relations = 0x0, es_trig_target_relations = 0x0, es_trig_tuple_slot = 0x0, 
      es_trig_oldtup_slot = 0x0, es_trig_newtup_slot = 0x0, es_param_list_info = 0x0, es_param_exec_vals = 0x0, 
      es_queryEnv = 0x0, es_query_cxt = 0x14dae30, es_tupleTable = 0x14dbaf8, es_rowMarks = 0x0, es_processed = 0, 
      es_lastoid = 0, es_top_eflags = 16, es_instrument = 0, es_finished = false, es_exprcontexts = 0x14db550, 
      es_subplanstates = 0x0, es_auxmodifytables = 0x0, es_per_tuple_exprcontext = 0x0, es_epqTuple = 0x0, 
      es_epqTupleSet = 0x0, es_epqScanDone = 0x0, es_use_parallel_mode = false, es_query_dsa = 0x0, es_jit_flags = 0, 
      es_jit = 0x0, es_jit_worker_instr = 0x0}
    (gdb) p *planstate
    $2 = {type = T_SortState, plan = 0x14d3f90, state = 0x14daf48, ExecProcNode = 0x6e41bb <ExecProcNodeFirst>, 
      ExecProcNodeReal = 0x716144 <ExecSort>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
      qual = 0x0, lefttree = 0x14db278, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
      ps_ResultTupleSlot = 0x14ec470, ps_ExprContext = 0x0, ps_ProjInfo = 0x0, scandesc = 0x14e9fd0}
    (gdb) p *dest
    $4 = {receiveSlot = 0x48cc00 <printtup>, rStartup = 0x48c5c1 <printtup_startup>, rShutdown = 0x48d02e <printtup_shutdown>, 
      rDestroy = 0x48d0a7 <printtup_destroy>, mydest = DestRemote}  
    

赋值,准备执行ExecProcNode(ExecSort)

    
    
    (gdb) n
    1699        estate->es_direction = direction;
    (gdb) 
    1705        if (!execute_once)
    (gdb) 
    1708        estate->es_use_parallel_mode = use_parallel_mode;
    (gdb) 
    1709        if (use_parallel_mode)
    (gdb) 
    1718            ResetPerTupleExprContext(estate);
    (gdb) 
    1723            slot = ExecProcNode(planstate);
    (gdb) 
    

执行ExecProcNode(ExecSort),返回slot

    
    
    (gdb) 
    1729            if (TupIsNull(slot))
    (gdb) p *slot
    $5 = {type = T_TupleTableSlot, tts_isempty = false, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, 
      tts_tuple = 0x14ec4b0, tts_tupleDescriptor = 0x14ec058, tts_mcxt = 0x14dae30, tts_buffer = 0, tts_nvalid = 0, 
      tts_values = 0x14ec4d0, tts_isnull = 0x14ec508, tts_mintuple = 0x1a4b078, tts_minhdr = {t_len = 64, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x1a4b070}, tts_off = 0, 
      tts_fixedTupleDescriptor = true}
    

查看slot中的数据  
注意:slot中的t_data不是实际的tuple data,而是缓冲区信息,在返回时根据这些信息从缓冲区获取数据返回

    
    
    (gdb) p *slot
    $5 = {type = T_TupleTableSlot, tts_isempty = false, tts_shouldFree = false, tts_shouldFreeMin = false, tts_slow = false, 
      tts_tuple = 0x14ec4b0, tts_tupleDescriptor = 0x14ec058, tts_mcxt = 0x14dae30, tts_buffer = 0, tts_nvalid = 0, 
      tts_values = 0x14ec4d0, tts_isnull = 0x14ec508, tts_mintuple = 0x1a4b078, tts_minhdr = {t_len = 64, t_self = {ip_blkid = {
            bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x1a4b070}, tts_off = 0, 
      tts_fixedTupleDescriptor = true}
    (gdb) p *slot->tts_tuple
    $6 = {t_len = 64, t_self = {ip_blkid = {bi_hi = 0, bi_lo = 0}, ip_posid = 0}, t_tableOid = 0, t_data = 0x1a4b070}
    (gdb) p *slot->tts_tuple->t_data
    $7 = {t_choice = {t_heap = {t_xmin = 21967600, t_xmax = 0, t_field3 = {t_cid = 56, t_xvac = 56}}, t_datum = {
          datum_len_ = 21967600, datum_typmod = 0, datum_typeid = 56}}, t_ctid = {ip_blkid = {bi_hi = 0, bi_lo = 0}, 
        ip_posid = 32639}, t_infomask2 = 7, t_infomask = 2, t_hoff = 24 '\030', t_bits = 0x1a4b087 ""}
    

判断是否需要过滤属性(不需要)

    
    
    (gdb) n
    1748            if (estate->es_junkFilter != NULL)
    (gdb) 
    (gdb) p estate->es_junkFilter
    $12 = (JunkFilter *) 0x0
    

修改计数器等信息

    
    
    (gdb) 
    1755            if (sendTuples)
    (gdb) 
    1762                if (!dest->receiveSlot(slot, dest))
    (gdb) 
    1771            if (operation == CMD_SELECT)
    (gdb) 
    1772                (estate->es_processed)++;
    (gdb) p estate->es_processed
    $9 = 0
    (gdb) n
    1779            current_tuple_count++;
    (gdb) p current_tuple_count
    $10 = 0
    (gdb) n
    1780            if (numberTuples && numberTuples == current_tuple_count)
    (gdb) p numberTuples
    $11 = 0
    (gdb) n
    1790        }
    

继续循环,直接满足条件(全部扫描完毕)未知

    
    
    (gdb) n
    1718            ResetPerTupleExprContext(estate);
    (gdb) 
    1723            slot = ExecProcNode(planstate);
    (gdb) 
    1729            if (TupIsNull(slot))
    ...
    

**ExecutePlan的主体逻辑已介绍完毕,下面简单跟踪分析ExecSeqScan函数**  
设置断点,进入ExecSeqScan

    
    
    (gdb) del 1
    (gdb) c
    Continuing.
    
    Breakpoint 2, ExecSeqScan (pstate=0x14e99a0) at nodeSeqscan.c:127
    warning: Source file is more recent than executable.
    127     SeqScanState *node = castNode(SeqScanState, pstate);
    

查看输入参数  
plan为SeqScan  
ExecProcNode=ExecProcNodeReal,均为函数ExecSeqScan  
targetlist为投影列信息

    
    
    (gdb) p *pstate
    $13 = {type = T_SeqScanState, plan = 0x14d5570, state = 0x14daf48, ExecProcNode = 0x714d59 <ExecSeqScan>, 
      ExecProcNodeReal = 0x714d59 <ExecSeqScan>, instrument = 0x0, worker_instrument = 0x0, worker_jit_instrument = 0x0, 
      qual = 0x0, lefttree = 0x0, righttree = 0x0, initPlan = 0x0, subPlan = 0x0, chgParam = 0x0, 
      ps_ResultTupleSlot = 0x14e9c38, ps_ExprContext = 0x14e9ab8, ps_ProjInfo = 0x0, scandesc = 0x7fa45b442ab8}
    (gdb) p *pstate->plan
    $14 = {type = T_SeqScan, startup_cost = 0, total_cost = 164, plan_rows = 10000, plan_width = 20, parallel_aware = false, 
      parallel_safe = true, plan_node_id = 7, targetlist = 0x14d5438, qual = 0x0, lefttree = 0x0, righttree = 0x0, 
      initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    

进入ExecScan函数  
accessMtd方法为SeqNext  
recheckMtd方法为SeqRecheck

    
    
    (gdb) n
    129     return ExecScan(&node->ss,
    (gdb) step
    ExecScan (node=0x14e99a0, accessMtd=0x714c6d <SeqNext>, recheckMtd=0x714d3d <SeqRecheck>) at execScan.c:132
    warning: Source file is more recent than executable.
    132     qual = node->ps.qual;
    

ExecScan->投影信息,为NULL

    
    
    (gdb) p *projInfo
    Cannot access memory at address 0x0
    

ExecScan->约束条件为NULL

    
    
    (gdb) p *qual
    Cannot access memory at address 0x0
    

ExecScan->如果既没有要检查的条件qual,也没有要做的投影操作，那么就跳过所有的操作并返回raw scan元组

    
    
    (gdb) n
    142     if (!qual && !projInfo)
    (gdb) 
    144         ResetExprContext(econtext);
    (gdb) n
    145         return ExecScanFetch(node, accessMtd, recheckMtd);
    

ExecScan->进入ExecScanFetch

    
    
    (gdb) step
    ExecScanFetch (node=0x14e99a0, accessMtd=0x714c6d <SeqNext>, recheckMtd=0x714d3d <SeqRecheck>) at execScan.c:39
    39      EState     *estate = node->ps.state;
    

ExecScan->检查中断,判断是否处于EvalPlanQual recheck状态(为NULL,实际不是)

    
    
    39      EState     *estate = node->ps.state;
    (gdb) n
    41      CHECK_FOR_INTERRUPTS();
    (gdb) 
    43      if (estate->es_epqTuple != NULL)
    (gdb) p *estate->es_epqTuple
    Cannot access memory at address 0x0
    

ExecScan->调用访问方法SeqNext,返回slot

    
    
    (gdb) n
    95      return (*accessMtd) (node);
    (gdb) n
    96  }
    

ExecScan->回到ExecScan&ExecSeqScan,结束调用

    
    
    (gdb) n
    ExecScan (node=0x14e99a0, accessMtd=0x714c6d <SeqNext>, recheckMtd=0x714d3d <SeqRecheck>) at execScan.c:219
    219 }
    (gdb) 
    ExecSeqScan (pstate=0x14e99a0) at nodeSeqscan.c:132
    132 }
    (gdb) 
    

DONE!

### 四、参考资料

[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

