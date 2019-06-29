本节简单介绍了创建执行计划主函数create_plan的实现逻辑。

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

create_plan调用create_plan_recurse函数，递归遍历访问路径，相应的创建计划（Plan）节点。

    
    
    /*
     * create_plan
     *    Creates the access plan for a query by recursively processing the
     *    desired tree of pathnodes, starting at the node 'best_path'.  For
     *    every pathnode found, we create a corresponding plan node containing
     *    appropriate id, target list, and qualification information.
     *    从节点'best_path'开始,递归处理路径节点树，为查询语句创建执行计划。
     *    对于找到的每个访问路径节点，创建一个相应的计划节点，其中包含合适的id、投影列和限定信息。
     *
     *    The tlists and quals in the plan tree are still in planner format,
     *    ie, Vars still correspond to the parser's numbering.  This will be
     *    fixed later by setrefs.c.
     *    计划中的投影列和约束条件仍然以优化器的格式存储.
     *    比如Vars对应着解析树的编号等,这些处理都在setrefs.c中完成
     *
     *    best_path is the best access path
     *    best_path是最优的访问路径
     *
     *    Returns a Plan tree.
     *    返回Plan树.
     */
    Plan *
    create_plan(PlannerInfo *root, Path *best_path)
    {
        Plan       *plan;
    
        /* plan_params should not be in use in current query level */
        //plan_params在当前查询层次上不应使用(值为NULL)
        Assert(root->plan_params == NIL);
    
        /* Initialize this module's private workspace in PlannerInfo */
        //初始化该模块中优化器信息的私有工作空间
        root->curOuterRels = NULL;
        root->curOuterParams = NIL;
    
        /* Recursively process the path tree, demanding the correct tlist result */
        //递归处理计划树(tlist参数设置为CP_EXACT_TLIST)
        plan = create_plan_recurse(root, best_path, CP_EXACT_TLIST);
    
        /*
         * Make sure the topmost plan node's targetlist exposes the original
         * column names and other decorative info.  Targetlists generated within
         * the planner don't bother with that stuff, but we must have it on the
         * top-level tlist seen at execution time.  However, ModifyTable plan
         * nodes don't have a tlist matching the querytree targetlist.
         * 确保最顶层计划节点的投影列targetlist可以获知原始列名和其他调整过的信息。
         * 在计划器中生成的投影列targetlist不需要处理这些信息，但是必须在执行时看到最高层的投影列tlist。
         * 注意:ModifyTable计划节点没有一个匹配查询树targetlist的tlist。
         */
        if (!IsA(plan, ModifyTable))
            apply_tlist_labeling(plan->targetlist, root->processed_tlist);//非ModifyTable
    
        /*
         * Attach any initPlans created in this query level to the topmost plan
         * node.  (In principle the initplans could go in any plan node at or
         * above where they're referenced, but there seems no reason to put them
         * any lower than the topmost node for the query level.  Also, see
         * comments for SS_finalize_plan before you try to change this.)
         * 将此查询级别中创建的任何initplan附加到最高层的计划节点中。
         * (原则上，initplans可以在引用它们的任何计划节点或以上的节点中访问，
         * 但似乎没有理由将它们放在查询级别的最高层节点以下。
         * 另外，如需尝试更改SS_finalize_plan,请参阅注释。)
         */
        SS_attach_initplans(root, plan);
    
        /* Check we successfully assigned all NestLoopParams to plan nodes */
        //检查已经为计划节点参数NestLoopParams赋值
        if (root->curOuterParams != NIL)
            elog(ERROR, "failed to assign all NestLoopParams to plan nodes");
    
        /*
         * Reset plan_params to ensure param IDs used for nestloop params are not
         * re-used later
         * 重置plan_params参数,以确保用于nestloop参数的参数IDs不会在后续被重复使用
         */
        root->plan_params = NIL;
    
        return plan;
    }
    
    //------------------------------------------------------------------------ create_plan_recurse
    /*
     * create_plan_recurse
     *    Recursive guts of create_plan().
     *    create_plan()函数中的递归实现过程.
     */
    static Plan *
    create_plan_recurse(PlannerInfo *root, Path *best_path, int flags)
    {
        Plan       *plan;
    
        /* Guard against stack overflow due to overly complex plans */
        //确保堆栈不会溢出
        check_stack_depth();
    
        switch (best_path->pathtype)//根据路径类型,执行相应的处理
        {
            case T_SeqScan://顺序扫描
            case T_SampleScan://采样扫描
            case T_IndexScan://索引扫描
            case T_IndexOnlyScan://索引快速扫描
            case T_BitmapHeapScan://位图堆扫描
            case T_TidScan://TID扫描
            case T_SubqueryScan://子查询扫描
            case T_FunctionScan://函数扫描
            case T_TableFuncScan://表函数扫描
            case T_ValuesScan://Values扫描
            case T_CteScan://CTE扫描
            case T_WorkTableScan://WorkTable扫描
            case T_NamedTuplestoreScan://NamedTuplestore扫描
            case T_ForeignScan://外表扫描
            case T_CustomScan://自定义扫描
                plan = create_scan_plan(root, best_path, flags);//扫描计划
                break;
            case T_HashJoin://Hash连接
            case T_MergeJoin://合并连接
            case T_NestLoop://内嵌循环连接
                plan = create_join_plan(root,
                                        (JoinPath *) best_path);//连接结合
                break;
            case T_Append://追加(集合)
                plan = create_append_plan(root,
                                          (AppendPath *) best_path);//追加(集合并)计划
                break;
            case T_MergeAppend://合并
                plan = create_merge_append_plan(root,
                                                (MergeAppendPath *) best_path);
                break;
            case T_Result://投影操作
                if (IsA(best_path, ProjectionPath))
                {
                    plan = create_projection_plan(root,
                                                  (ProjectionPath *) best_path,
                                                  flags);
                }
                else if (IsA(best_path, MinMaxAggPath))
                {
                    plan = (Plan *) create_minmaxagg_plan(root,
                                                          (MinMaxAggPath *) best_path);
                }
                else
                {
                    Assert(IsA(best_path, ResultPath));
                    plan = (Plan *) create_result_plan(root,
                                                       (ResultPath *) best_path);
                }
                break;
            case T_ProjectSet://投影集合操作
                plan = (Plan *) create_project_set_plan(root,
                                                        (ProjectSetPath *) best_path);
                break;
            case T_Material://物化
                plan = (Plan *) create_material_plan(root,
                                                     (MaterialPath *) best_path,
                                                     flags);
                break;
            case T_Unique://唯一处理
                if (IsA(best_path, UpperUniquePath))
                {
                    plan = (Plan *) create_upper_unique_plan(root,
                                                             (UpperUniquePath *) best_path,
                                                             flags);
                }
                else
                {
                    Assert(IsA(best_path, UniquePath));
                    plan = create_unique_plan(root,
                                              (UniquePath *) best_path,
                                              flags);
                }
                break;
            case T_Gather://汇总收集
                plan = (Plan *) create_gather_plan(root,
                                                   (GatherPath *) best_path);
                break;
            case T_Sort://排序
                plan = (Plan *) create_sort_plan(root,
                                                 (SortPath *) best_path,
                                                 flags);
                break;
            case T_Group://分组
                plan = (Plan *) create_group_plan(root,
                                                  (GroupPath *) best_path);
                break;
            case T_Agg://聚集计算
                if (IsA(best_path, GroupingSetsPath))
                    plan = create_groupingsets_plan(root,
                                                    (GroupingSetsPath *) best_path);
                else
                {
                    Assert(IsA(best_path, AggPath));
                    plan = (Plan *) create_agg_plan(root,
                                                    (AggPath *) best_path);
                }
                break;
            case T_WindowAgg://窗口函数
                plan = (Plan *) create_windowagg_plan(root,
                                                      (WindowAggPath *) best_path);
                break;
            case T_SetOp://集合操作
                plan = (Plan *) create_setop_plan(root,
                                                  (SetOpPath *) best_path,
                                                  flags);
                break;
            case T_RecursiveUnion://递归UNION
                plan = (Plan *) create_recursiveunion_plan(root,
                                                           (RecursiveUnionPath *) best_path);
                break;
            case T_LockRows://锁定(for update)
                plan = (Plan *) create_lockrows_plan(root,
                                                     (LockRowsPath *) best_path,
                                                     flags);
                break;
            case T_ModifyTable://更新
                plan = (Plan *) create_modifytable_plan(root,
                                                        (ModifyTablePath *) best_path);
                break;
            case T_Limit://限制操作
                plan = (Plan *) create_limit_plan(root,
                                                  (LimitPath *) best_path,
                                                  flags);
                break;
            case T_GatherMerge://收集合并
                plan = (Plan *) create_gather_merge_plan(root,
                                                         (GatherMergePath *) best_path);
                break;
            default://其他非法类型
                elog(ERROR, "unrecognized node type: %d",
                     (int) best_path->pathtype);
                plan = NULL;        /* keep compiler quiet */
                break;
        }
    
        return plan;
    }
    
    
    //------------------------------------------------------------------------ apply_tlist_labeling
    
     /*
      * apply_tlist_labeling
      *      Apply the TargetEntry labeling attributes of src_tlist to dest_tlist
      *      将src_tlist的TargetEntry标记属性应用到dest_tlist
      *
      * This is useful for reattaching column names etc to a plan's final output
      * targetlist.
      */
     void
     apply_tlist_labeling(List *dest_tlist, List *src_tlist)
     {
         ListCell   *ld,
                    *ls;
     
         Assert(list_length(dest_tlist) == list_length(src_tlist));
         forboth(ld, dest_tlist, ls, src_tlist)
         {
             TargetEntry *dest_tle = (TargetEntry *) lfirst(ld);
             TargetEntry *src_tle = (TargetEntry *) lfirst(ls);
     
             Assert(dest_tle->resno == src_tle->resno);
             dest_tle->resname = src_tle->resname;
             dest_tle->ressortgroupref = src_tle->ressortgroupref;
             dest_tle->resorigtbl = src_tle->resorigtbl;
             dest_tle->resorigcol = src_tle->resorigcol;
             dest_tle->resjunk = src_tle->resjunk;
         }
     }
     
    
    //------------------------------------------------------------------------ apply_tlist_labeling
     /*
      * SS_attach_initplans - attach initplans to topmost plan node
      * 将initplans附加到最顶层的计划节点 
      *
      * Attach any initplans created in the current query level to the specified
      * plan node, which should normally be the topmost node for the query level.
      * (In principle the initPlans could go in any node at or above where they're
      * referenced; but there seems no reason to put them any lower than the
      * topmost node, so we don't bother to track exactly where they came from.)
      * We do not touch the plan node's cost; the initplans should have been
      * accounted for in path costing.
      */
     void
     SS_attach_initplans(PlannerInfo *root, Plan *plan)
     {
         plan->initPlan = root->init_plans;
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
    
    

启动gdb,设置断点,进入

    
    
    (gdb) info break
    Num     Type           Disp Enb Address            What
    2       breakpoint     keep y   0x00000000007b76c1 in create_plan at createplan.c:313
    (gdb) c
    Continuing.
    
    Breakpoint 2, create_plan (root=0x26c1258, best_path=0x2722d00) at createplan.c:313
    313     Assert(root->plan_params == NIL);
    

进入create_plan_recurse函数

    
    
    313     Assert(root->plan_params == NIL);
    (gdb) n
    316     root->curOuterRels = NULL;
    (gdb) 
    317     root->curOuterParams = NIL;
    (gdb) 
    320     plan = create_plan_recurse(root, best_path, CP_EXACT_TLIST);
    (gdb) step
    create_plan_recurse (root=0x26c1258, best_path=0x2722d00, flags=1) at createplan.c:364
    364     check_stack_depth();
    

根据访问路径类型(T_ProjectionPath)选择处理分支

    
    
    (gdb) p *best_path
    $1 = {type = T_ProjectionPath, pathtype = T_Result, parent = 0x2722998, pathtarget = 0x27226f8, param_info = 0x0, 
      parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, startup_cost = 20070.931487218411, 
      total_cost = 20320.931487218411, pathkeys = 0x26cfe98}
    

调用create_projection_plan函数

    
    
    (gdb) n
    400             if (IsA(best_path, ProjectionPath))
    (gdb) 
    402                 plan = create_projection_plan(root,
    

创建相应的Plan(T_Sort,存在左右子树),下一节将详细解释create_projection_plan函数

    
    
    (gdb) 
    504     return plan;
    (gdb) p *plan
    $1 = {type = T_Sort, startup_cost = 20070.931487218411, total_cost = 20320.931487218411, plan_rows = 100000, 
      plan_width = 47, parallel_aware = false, parallel_safe = true, plan_node_id = 0, targetlist = 0x2724548, qual = 0x0, 
      lefttree = 0x27243d0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    

执行返回

    
    
    (gdb) 
    create_plan (root=0x270f9c8, best_path=0x2722d00) at createplan.c:329
    329     if (!IsA(plan, ModifyTable))
    (gdb) 
    330         apply_tlist_labeling(plan->targetlist, root->processed_tlist);
    (gdb) 
    339     SS_attach_initplans(root, plan);
    (gdb) 
    342     if (root->curOuterParams != NIL)
    (gdb) 
    349     root->plan_params = NIL;
    (gdb) 
    351     return plan;
    

DONE!

### 四、参考资料

[createplan.c](https://doxygen.postgresql.org/createplan_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

