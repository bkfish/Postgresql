本节简单介绍了创建执行计划中的create_plan->create_plan_recurse->create_projection_plan和create_sort_plan函数实现逻辑。

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

create_plan->create_plan_recurse->create_projection_plan函数创建计划树,执行投影操作并通过递归的方式为子访问路径生成执行计划。create_sort_plan函数创建Sort计划节点。

    
    
    //---------------------------------------------------------------- create_projection_plan
    
    /*
     * create_projection_plan
     *
     *    Create a plan tree to do a projection step and (recursively) plans
     *    for its subpaths.  We may need a Result node for the projection,
     *    but sometimes we can just let the subplan do the work.
     *    创建计划树,执行投影操作并通过递归的方式为子访问路径生成执行计划.
     *    一般来说需要一个Result节点用于投影操作,但有时候可以让子计划执行此项任务.
     */
    static Plan *
    create_projection_plan(PlannerInfo *root, ProjectionPath *best_path, int flags)
    {
        Plan       *plan;
        Plan       *subplan;
        List       *tlist;
        bool        needs_result_node = false;
    
        /*
         * Convert our subpath to a Plan and determine whether we need a Result
         * node.
         * 转换subpath为Plan,并确定是否需要Result节点.
         *
         * In most cases where we don't need to project, creation_projection_path
         * will have set dummypp, but not always.  First, some createplan.c
         * routines change the tlists of their nodes.  (An example is that
         * create_merge_append_plan might add resjunk sort columns to a
         * MergeAppend.)  Second, create_projection_path has no way of knowing
         * what path node will be placed on top of the projection path and
         * therefore can't predict whether it will require an exact tlist. For
         * both of these reasons, we have to recheck here.
         * 在大多数情况下，我们不需要投影运算，creation_projection_path将设置dummypp标志，但并不总是如此。
         * 首先,一些createplan.c中的函数更改其节点的tlist。
         * (例如，create_merge_append_plan可能会向MergeAppend添加resjunk sort列)。
         * 其次，create_projection_path无法知道将在投影路径顶部放置哪些路径节点，因此无法预测它是否需要一个确切的tlist。
         * 由于这两个原因，我们不得不在这里重新检查。
         */
        if (use_physical_tlist(root, &best_path->path, flags))
        {
            /*
             * Our caller doesn't really care what tlist we return, so we don't
             * actually need to project.  However, we may still need to ensure
             * proper sortgroupref labels, if the caller cares about those.
             * 如果我们的调用者并不关心返回的结果tlist，所以实际上不需要投影运算。
             * 然而，如果调用者关心sortgroupref标签，可能仍然需要确保正确的sortgroupref数据。
             */
            subplan = create_plan_recurse(root, best_path->subpath, 0);
            tlist = subplan->targetlist;
            if (flags & CP_LABEL_TLIST)
                apply_pathtarget_labeling_to_tlist(tlist,
                                                   best_path->path.pathtarget);
        }
        else if (is_projection_capable_path(best_path->subpath))
        {
            /*
             * Our caller requires that we return the exact tlist, but no separate
             * result node is needed because the subpath is projection-capable.
             * Tell create_plan_recurse that we're going to ignore the tlist it
             * produces.
             * 调用者要求返回精确的tlist，但是不需要单独的Result节点，因为子路径支持投影。
             * 调用create_plan_recurse时忽略它生成的tlist。
             */
            subplan = create_plan_recurse(root, best_path->subpath,
                                          CP_IGNORE_TLIST);
            tlist = build_path_tlist(root, &best_path->path);
        }
        else
        {
            /*
             * It looks like we need a result node, unless by good fortune the
             * requested tlist is exactly the one the child wants to produce.
             * 看起来需要一个Result节点，除非幸运的是，请求的tlist正是子节点想要生成的。
             */
            subplan = create_plan_recurse(root, best_path->subpath, 0);
            tlist = build_path_tlist(root, &best_path->path);
            needs_result_node = !tlist_same_exprs(tlist, subplan->targetlist);
        }
    
        /*
         * If we make a different decision about whether to include a Result node
         * than create_projection_path did, we'll have made slightly wrong cost
         * estimates; but label the plan with the cost estimates we actually used,
         * not "corrected" ones.  (XXX this could be cleaned up if we moved more
         * of the sortcolumn setup logic into Path creation, but that would add
         * expense to creating Paths we might end up not using.)
         * 如果我们对是否包含一个Result节点做出与create_projection_path不同的决定，
         * 我们就会做出略微错误的成本估算;
         * 但是在这个计划上标上我们实际使用的成本估算，而不是“修正的”成本估算。
         * (如果我们将更多的sortcolumn设置逻辑移到路径创建中，这个问题就可以解决了，
         *  但是这会增加创建路径的成本，而最终可能不会使用这些路径。)
         */
        if (!needs_result_node)
        {
            /* Don't need a separate Result, just assign tlist to subplan */
            //不需要单独的Result节点,把tlist赋值给subplan
            plan = subplan;
            plan->targetlist = tlist;
    
            /* Label plan with the estimated costs we actually used */
            //标记估算成本
            plan->startup_cost = best_path->path.startup_cost;
            plan->total_cost = best_path->path.total_cost;
            plan->plan_rows = best_path->path.rows;
            plan->plan_width = best_path->path.pathtarget->width;
            plan->parallel_safe = best_path->path.parallel_safe;
            /* ... but don't change subplan's parallel_aware flag */
        }
        else
        {
            /* We need a Result node */
            //需要Result节点
            plan = (Plan *) make_result(tlist, NULL, subplan);
    
            copy_generic_path_info(plan, (Path *) best_path);
        }
    
        return plan;
    }
    
    //---------------------------------------------------------------- create_sort_plan
     /*
      * create_sort_plan
      *
      *    Create a Sort plan for 'best_path' and (recursively) plans
      *    for its subpaths.
      *    创建Sort计划节点
      */
     static Sort *
     create_sort_plan(PlannerInfo *root, SortPath *best_path, int flags)
     {
         Sort       *plan;
         Plan       *subplan;
     
         /*
          * We don't want any excess columns in the sorted tuples, so request a
          * smaller tlist.  Otherwise, since Sort doesn't project, tlist
          * requirements pass through.
          * 我们不希望在排序元组中有任何多余的列，所以希望得到一个更小的tlist。
          * 否则，由于Sort不执行投影运算，tlist就会通过完整的保留传递。
          */
         subplan = create_plan_recurse(root, best_path->subpath,
                                       flags | CP_SMALL_TLIST);
     
         /*
          * make_sort_from_pathkeys() indirectly calls find_ec_member_for_tle(),
          * which will ignore any child EC members that don't belong to the given
          * relids. Thus, if this sort path is based on a child relation, we must
          * pass its relids.
          * make_sort_from_pathkeys()间接调用find_ec_member_for_tle()，它将忽略不属于给定relid的任何子EC成员。
          * 因此，如果这个排序路径是基于子关系的，我们必须传递它的relids。
          */
         plan = make_sort_from_pathkeys(subplan, best_path->path.pathkeys,
                                        IS_OTHER_REL(best_path->subpath->parent) ?
                                        best_path->path.parent->relids : NULL);
     
         copy_generic_path_info(&plan->plan, (Path *) best_path);
     
         return plan;
     }
    
    
    //------------------------------------------------ build_path_tlist
    
    /*
     * Build a target list (ie, a list of TargetEntry) for the Path's output.
     * 构建用于输出的target链表(比如TargetEntry节点链表)
     *
     * This is almost just make_tlist_from_pathtarget(), but we also have to
     * deal with replacing nestloop params.
     * 该函数几乎就是make_tlist_from_pathtarget()的实现逻辑，但还必须处理替换nestloop参数。
     */
    static List *
    build_path_tlist(PlannerInfo *root, Path *path)
    {
        List       *tlist = NIL;
        Index      *sortgrouprefs = path->pathtarget->sortgrouprefs;
        int         resno = 1;
        ListCell   *v;
    
        foreach(v, path->pathtarget->exprs)
        {
            Node       *node = (Node *) lfirst(v);
            TargetEntry *tle;
    
            /*
             * If it's a parameterized path, there might be lateral references in
             * the tlist, which need to be replaced with Params.  There's no need
             * to remake the TargetEntry nodes, so apply this to each list item
             * separately.
             * 如果是参数化路径，那么tlist中可能有横向引用，需要用Params替换。
             * 不需要重新构建TargetEntry节点，因此可以将其单独应用于每个链表项。
             */
            if (path->param_info)
                node = replace_nestloop_params(root, node);
    
            tle = makeTargetEntry((Expr *) node,
                                  resno,
                                  NULL,
                                  false);
            if (sortgrouprefs)
                tle->ressortgroupref = sortgrouprefs[resno - 1];
    
            tlist = lappend(tlist, tle);
            resno++;
        }
        return tlist;
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
    
    

启动gdb,设置断点,进入create_projection_plan函数

    
    
    (gdb) b create_projection_plan
    Breakpoint 2 at 0x7b95a8: file createplan.c, line 1627.
    (gdb) c
    Continuing.
    
    Breakpoint 2, create_projection_plan (root=0x26c1258, best_path=0x2722d00, flags=1) at createplan.c:1627
    1627        bool        needs_result_node = false;
    

转换subpath为Plan,并确定是否需要Result节点,并且判断是否需要生成Result节点

    
    
    ...
    (gdb) n
    1642        if (use_physical_tlist(root, &best_path->path, flags))
    (gdb) n
    1655        else if (is_projection_capable_path(best_path->subpath))
    (gdb) 
    1673            subplan = create_plan_recurse(root, best_path->subpath, 0);
    

查看best_path&best_path->subpath变量

    
    
    (gdb) p *best_path
    $3 = {path = {type = T_ProjectionPath, pathtype = T_Result, parent = 0x2722998, pathtarget = 0x27226f8, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, startup_cost = 20070.931487218411, 
        total_cost = 20320.931487218411, pathkeys = 0x26cfe98}, subpath = 0x2722c68, dummypp = true}
    (gdb) p *(SortPath *)best_path->subpath
    $16 = {path = {type = T_SortPath, pathtype = T_Sort, parent = 0x2722998, pathtarget = 0x27212d8, param_info = 0x0, 
        parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, startup_cost = 20070.931487218411, 
        total_cost = 20320.931487218411, pathkeys = 0x26cfe98}, subpath = 0x2721e60}
    

创建subpath(SortPath)的执行计划

    
    
    (gdb) step
    create_plan_recurse (root=0x26c1258, best_path=0x2722c68, flags=0) at createplan.c:364
    364     check_stack_depth();
    (gdb) n
    366     switch (best_path->pathtype)
    (gdb) 
    447             plan = (Plan *) create_sort_plan(root,
    

进入create_sort_plan

    
    
    (gdb) step
    create_sort_plan (root=0x26c1258, best_path=0x2722c68, flags=0) at createplan.c:1759
    1759        subplan = create_plan_recurse(root, best_path->subpath,
    

SortPath的subpath是HashPath

    
    
    (gdb) p best_path->subpath->type
    $17 = T_HashPath
    (gdb) p *(HashPath *)best_path->subpath
    $18 = {jpath = {path = {type = T_HashPath, pathtype = T_HashJoin, parent = 0x27210c0, pathtarget = 0x27212d8, 
          param_info = 0x0, parallel_aware = false, parallel_safe = true, parallel_workers = 0, rows = 100000, 
          startup_cost = 3754, total_cost = 8689.6112499999999, pathkeys = 0x0}, jointype = JOIN_INNER, inner_unique = true, 
        outerjoinpath = 0x2720f68, innerjoinpath = 0x26d0598, joinrestrictinfo = 0x2722068}, path_hashclauses = 0x27223c0, 
      num_batches = 1, inner_rows_total = 10000}
    

完成SortPath执行计划的构建

    
    
    (gdb) 
    1774        return plan;
    (gdb) 
    1775    }
    (gdb) p *plan
    $20 = {plan = {type = T_Sort, startup_cost = 20070.931487218411, total_cost = 20320.931487218411, plan_rows = 100000, 
        plan_width = 47, parallel_aware = false, parallel_safe = true, plan_node_id = 0, targetlist = 0x2723208, qual = 0x0, 
        lefttree = 0x27243d0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}, numCols = 1, 
      sortColIdx = 0x27222a0, sortOperators = 0x2724468, collations = 0x2724488, nullsFirst = 0x27244a8}
    

回到上一层

    
    
    (gdb) n
    create_plan_recurse (root=0x26c1258, best_path=0x2722c68, flags=0) at createplan.c:450
    450             break;
    

回到create_projection_plan函数

    
    
    (gdb) n
    504     return plan;
    (gdb) 
    505 }
    (gdb) 
    create_projection_plan (root=0x26c1258, best_path=0x2722d00, flags=1) at createplan.c:1674
    1674            tlist = build_path_tlist(root, &best_path->path);
    

执行完毕,返回create_plan,结果,最外层的Plan为Sort

    
    
    (gdb) 
    1708        return plan;
    (gdb) 
    1709    }
    (gdb) p *plan
    $22 = {type = T_Sort, startup_cost = 20070.931487218411, total_cost = 20320.931487218411, plan_rows = 100000, 
      plan_width = 47, parallel_aware = false, parallel_safe = true, plan_node_id = 0, targetlist = 0x2724548, qual = 0x0, 
      lefttree = 0x27243d0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    (gdb) n
    create_plan_recurse (root=0x26c1258, best_path=0x2722d00, flags=1) at createplan.c:504
    504     return plan;
    (gdb) p *plan
    $23 = {type = T_Sort, startup_cost = 20070.931487218411, total_cost = 20320.931487218411, plan_rows = 100000, 
      plan_width = 47, parallel_aware = false, parallel_safe = true, plan_node_id = 0, targetlist = 0x2724548, qual = 0x0, 
      lefttree = 0x27243d0, righttree = 0x0, initPlan = 0x0, extParam = 0x0, allParam = 0x0}
    (gdb) n
    505 }
    (gdb) 
    create_plan (root=0x26c1258, best_path=0x2722d00) at createplan.c:329
    329     if (!IsA(plan, ModifyTable))
    

DONE!

### 四、参考资料

[createplan.c](https://doxygen.postgresql.org/createplan_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

