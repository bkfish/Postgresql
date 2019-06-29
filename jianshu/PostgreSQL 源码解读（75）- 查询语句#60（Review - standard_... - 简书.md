本节Review standard_planner函数的实现逻辑，该函数查询优化器的主入口。

### 一、源码解读

standard_planner函数由exec_simple_query->pg_plan_queries->pg_plan_query->planner函数调用,其调用栈如下:

    
    
    (gdb) b planner
    Breakpoint 1 at 0x7c7442: file planner.c, line 259.
    (gdb) c
    Continuing.
    
    Breakpoint 1, planner (parse=0x2eee0b0, cursorOptions=256, boundParams=0x0) at planner.c:259
    259     if (planner_hook)
    (gdb) bt
    #0  planner (parse=0x2eee0b0, cursorOptions=256, boundParams=0x0) at planner.c:259
    #1  0x00000000008c5824 in pg_plan_query (querytree=0x2eee0b0, cursorOptions=256, boundParams=0x0) at postgres.c:809
    #2  0x00000000008c595d in pg_plan_queries (querytrees=0x2f10ac8, cursorOptions=256, boundParams=0x0) at postgres.c:875
    #3  0x00000000008c5c35 in exec_simple_query (query_string=0x2eed1a8 "select * from t1;") at postgres.c:1050
    #4  0x00000000008ca0f8 in PostgresMain (argc=1, argv=0x2f16cb8, dbname=0x2f16b20 "testdb", username=0x2ee9d48 "xdb")
        at postgres.c:4159
    #5  0x0000000000825880 in BackendRun (port=0x2f0eb00) at postmaster.c:4361
    #6  0x0000000000824fe4 in BackendStartup (port=0x2f0eb00) at postmaster.c:4033
    #7  0x0000000000821371 in ServerLoop () at postmaster.c:1706
    #8  0x0000000000820c09 in PostmasterMain (argc=1, argv=0x2ee7d00) at postmaster.c:1379
    #9  0x0000000000747ea7 in main (argc=1, argv=0x2ee7d00) at main.c:228
    

该函数根据输入的查询树(Query)生成已规划的语句,可用于执行器实际执行.

    
    
    /*****************************************************************************
     *
     *     Query optimizer entry point
     *     查询优化器入口
     *
     * To support loadable plugins that monitor or modify planner behavior,
     * we provide a hook variable that lets a plugin get control before and
     * after the standard planning process.  The plugin would normally call
     * standard_planner().
     * 在调用标准的规划过程前后提供了钩子函数插件,可以用于监控或者改变优化器的行为.
     * 
     * Note to plugin authors: standard_planner() scribbles on its Query input,
     * so you'd better copy that data structure if you want to plan more than once.
     * 开发插件时需要注意:standard_planner()函数会修改输入参数,如果想要多次规划,最好复制输入的数据结构
     * 
     *****************************************************************************/
    PlannedStmt *
    planner(Query *parse, int cursorOptions, ParamListInfo boundParams)
    {
        PlannedStmt *result;
    
        if (planner_hook)//钩子函数
            result = (*planner_hook) (parse, cursorOptions, boundParams);
        else
            result = standard_planner(parse, cursorOptions, boundParams);
        return result;
    }
    
    PlannedStmt *
    standard_planner(Query *parse, int cursorOptions, ParamListInfo boundParams)//标准优化过程
    {
        PlannedStmt *result;//返回结果
        PlannerGlobal *glob;//全局的Plan信息-Global information for planning/optimization
        double      tuple_fraction;//
        PlannerInfo *root;//每个Query的Plan信息-Per-query information for planning/optimization
        RelOptInfo *final_rel;//Plan中的每个Relation信息-Per-relation information for planning/optimization
        Path       *best_path;//最优路径
        Plan       *top_plan;//最上层的Plan
        ListCell   *lp,//临时变量
                   *lr;
    
        /*
         * Set up global state for this planner invocation.  This data is needed
         * across all levels of sub-Query that might exist in the given command,
         * so we keep it in a separate struct that's linked to by each per-Query
         * PlannerInfo.
         * 为这个规划器的调用设置全局状态。
         * 由于在给定的命令中可能存在的所有级别的子查询中都需要此数据结构，
         * 因此我们将其保存在一个单独的结构中，每个查询PlannerInfo都链接到这个结构中。
         */
        glob = makeNode(PlannerGlobal);//构建PlannerGlobal
         //初始化参数
        glob->boundParams = boundParams;
        glob->subplans = NIL;
        glob->subroots = NIL;
        glob->rewindPlanIDs = NULL;
        glob->finalrtable = NIL;
        glob->finalrowmarks = NIL;
        glob->resultRelations = NIL;
        glob->nonleafResultRelations = NIL;
        glob->rootResultRelations = NIL;
        glob->relationOids = NIL;
        glob->invalItems = NIL;
        glob->paramExecTypes = NIL;
        glob->lastPHId = 0;
        glob->lastRowMarkId = 0;
        glob->lastPlanNodeId = 0;
        glob->transientPlan = false;
        glob->dependsOnRole = false;
    
        /*
         * Assess whether it's feasible to use parallel mode for this query. We
         * can't do this in a standalone backend, or if the command will try to
         * modify any data, or if this is a cursor operation, or if GUCs are set
         * to values that don't permit parallelism, or if parallel-unsafe
         * functions are present in the query tree.
         * 评估对这个查询使用并行模式是否可行。
         * 我们不能在独立的后台过程中进行此操作，或者在可以修改数据的命令种执行此操作，
         * 或者说如果是涉及游标的操作，或者GUCs设置为不允许并行的值，或者查询树中存在不安全的并行函数,均不允许。
         *
         * (Note that we do allow CREATE TABLE AS, SELECT INTO, and CREATE
         * MATERIALIZED VIEW to use parallel plans, but this is safe only because
         * the command is writing into a completely new table which workers won't
         * be able to see.  If the workers could see the table, the fact that
         * group locking would cause them to ignore the leader's heavyweight
         * relation extension lock and GIN page locks would make this unsafe.
         * We'll have to fix that somehow if we want to allow parallel inserts in
         * general; updates and deletes have additional problems especially around
         * combo CIDs.)
         * (请注意，我们确实允许CREATE TABLE AS、SELECT INTO和CREATE MATERIALIZED VIEW来使用并行计划，
         * 但是这是安全的，因为命令写到了一个全新的表中，并行处理过程worker看不到。
         * 如果工workers能看到这个表，那么group locking会导致它们忽略leader的重量级关系扩展锁和GIN page锁，这会变得不安全。
         * 如果希望做到并行插入，那么就必须解决这个问题;另外,更新和删除有存在额外的问题，特别是关于组合CIDs。
         *
         * For now, we don't try to use parallel mode if we're running inside a
         * parallel worker.  We might eventually be able to relax this
         * restriction, but for now it seems best not to have parallel workers
         * trying to create their own parallel workers.
         * 目前，如果在一个并行worker内部运行，则不会尝试使用并行模式。
         * 可能最终能够放宽这一限制，但就目前而言，最好不要让worker试图创建它们自己的worker。
         *
         * We can't use parallelism in serializable mode because the predicate
         * locking code is not parallel-aware.  It's not catastrophic if someone
         * tries to run a parallel plan in serializable mode; it just won't get
         * any workers and will run serially.  But it seems like a good heuristic
         * to assume that the same serialization level will be in effect at plan
         * time and execution time, so don't generate a parallel plan if we're in
         * serializable mode.
         * 在serializable事务模式下，不能使用并行，因为谓词锁定代码不支持并行。
         * 如果有人试图在serializable模式下运行并行计划，虽然这不是灾难性的,但它不会得到worker，而是会串行运行。
         * 但是，假设相同的serializable级别将在计划时和执行时生效，这似乎是一个很好的启发，
         * 因此，如果我们处于serializable模式，则不要生成并行计划。
         */
        if ((cursorOptions & CURSOR_OPT_PARALLEL_OK) != 0 &&
            IsUnderPostmaster &&    
            parse->commandType == CMD_SELECT &&
            !parse->hasModifyingCTE &&
            max_parallel_workers_per_gather > 0 &&
            !IsParallelWorker() &&
            !IsolationIsSerializable())//并行模式的判断
        {
            /* all the cheap tests pass, so scan the query tree */
            glob->maxParallelHazard = max_parallel_hazard(parse);
            glob->parallelModeOK = (glob->maxParallelHazard != PROPARALLEL_UNSAFE);
        }
        else
        {
            /* skip the query tree scan, just assume it's unsafe */
            glob->maxParallelHazard = PROPARALLEL_UNSAFE;
            glob->parallelModeOK = false;
        }
    
        /*
         * glob->parallelModeNeeded is normally set to false here and changed to
         * true during plan creation if a Gather or Gather Merge plan is actually
         * created (cf. create_gather_plan, create_gather_merge_plan).
         * 如果确实创建了一个聚合或聚合合并计划，则通常将glob->parallelModeNeeded设置为false，
         * 并在计划创建期间更改为true (cf. create_gather_plan, create_gather_merge_plan)。
         * 
         * However, if force_parallel_mode = on or force_parallel_mode = regress,
         * then we impose parallel mode whenever it's safe to do so, even if the
         * final plan doesn't use parallelism.  It's not safe to do so if the
         * query contains anything parallel-unsafe; parallelModeOK will be false
         * in that case.  Note that parallelModeOK can't change after this point.
         * Otherwise, everything in the query is either parallel-safe or
         * parallel-restricted, and in either case it should be OK to impose
         * parallel-mode restrictions.  If that ends up breaking something, then
         * either some function the user included in the query is incorrectly
         * labelled as parallel-safe or parallel-restricted when in reality it's
         * parallel-unsafe, or else the query planner itself has a bug.
         * 但是，如果force_parallel_mode = on或force_parallel_mode = regress，
         * 那么只要安全，我们就强制执行并行模式，即使最终计划不使用并行。
         * 如果查询包含任何不安全的内容，那么这样做是不安全的;在这种情况下，parallelModeOK将为false。
         * 注意，在这一点之后，parallelModeOK无法更改。
         * 否则，查询中的所有内容要么是并行安全的，要么是并行限制的，
         * 在任何一种情况下，都应该可以施加并行模式限制。
         * 如果这最终出现了问题，要么查询中包含的某个函数被错误地标记为并行安全或并行限制(实际它是并行不安全的)，
         * 要么查询规划器本身有错误。
         */
        glob->parallelModeNeeded = glob->parallelModeOK &&
            (force_parallel_mode != FORCE_PARALLEL_OFF);
    
        /* Determine what fraction of the plan is likely to be scanned */
        //确定扫描比例
        if (cursorOptions & CURSOR_OPT_FAST_PLAN)
        {
            /*
             * We have no real idea how many tuples the user will ultimately FETCH
             * from a cursor, but it is often the case that he doesn't want 'em
             * all, or would prefer a fast-start plan anyway so that he can
             * process some of the tuples sooner.  Use a GUC parameter to decide
             * what fraction to optimize for.
             * 我们不知道用户最终会从游标中获取多少元组，但通常情况下，用户不需要所有元组，
             * 或者更喜欢快速启动计划，以便更快地处理一些元组。
             * 使用GUC参数来决定优化哪个分数。
             */
            tuple_fraction = cursor_tuple_fraction;//使用GUC 参数
    
            /*
             * We document cursor_tuple_fraction as simply being a fraction, which
             * means the edge cases 0 and 1 have to be treated specially here.  We
             * convert 1 to 0 ("all the tuples") and 0 to a very small fraction.
             * 我们将cursor_tuple_fraction记录为一个简单的分数，这意味着边界情况0和1必须在这里特别处理。
             * 我们将1转换为0(“所有元组”)，将0转换为非常小的分数。
             */
            if (tuple_fraction >= 1.0)
                tuple_fraction = 0.0;
            else if (tuple_fraction <= 0.0)
                tuple_fraction = 1e-10;
        }
        else
        {
            /* Default assumption is we need all the tuples */
            //默认假设:需要所有元组
            tuple_fraction = 0.0;
        }
    
        /* primary planning entry point (may recurse for subqueries) */
        //主规划过程入口(可能会递归执行)
        root = subquery_planner(glob, parse, NULL,
                                false, tuple_fraction);//获取PlannerInfo根节点
    
        /* Select best Path and turn it into a Plan */
        //选择最优路径并把该路径转换为执行计划
        final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);//获取顶层的RelOptInfo
        best_path = get_cheapest_fractional_path(final_rel, tuple_fraction);//选择最佳路径
    
        top_plan = create_plan(root, best_path);//生成执行计划
    
        /*
         * If creating a plan for a scrollable cursor, make sure it can run
         * backwards on demand.  Add a Material node at the top at need.
         * 如为可滚动游标创建计划，确保它可以根据需要向后运行。
         * 在需要时在最外层添加一个物化(Material)节点。
         */
        if (cursorOptions & CURSOR_OPT_SCROLL)
        {
            if (!ExecSupportsBackwardScan(top_plan))
                top_plan = materialize_finished_plan(top_plan);
        }
    
        /*
         * Optionally add a Gather node for testing purposes, provided this is
         * actually a safe thing to do.
         * 为了测试的目的，可以选择添加一个Gather节点，前提是这样做实际上是安全的。
         */
        if (force_parallel_mode != FORCE_PARALLEL_OFF && top_plan->parallel_safe)
        {
            Gather     *gather = makeNode(Gather);
    
            /*
             * If there are any initPlans attached to the formerly-top plan node,
             * move them up to the Gather node; same as we do for Material node in
             * materialize_finished_plan.
             * 如果有任何initplan连接到formerly-top的计划节点，将它们移动到Gather节点;
             * 这与我们在materialize_finished_plan中的Material节点相同。
             */
            gather->plan.initPlan = top_plan->initPlan;
            top_plan->initPlan = NIL;
    
            gather->plan.targetlist = top_plan->targetlist;
            gather->plan.qual = NIL;
            gather->plan.lefttree = top_plan;
            gather->plan.righttree = NULL;
            gather->num_workers = 1;
            gather->single_copy = true;
            gather->invisible = (force_parallel_mode == FORCE_PARALLEL_REGRESS);
    
            /*
             * Since this Gather has no parallel-aware descendants to signal to,
             * we don't need a rescan Param.
             * 因为这个Gather没有parallel-aware的后代信号，所以我们不需要重新扫描参数。
             */
            gather->rescan_param = -1;
    
            /*
             * Ideally we'd use cost_gather here, but setting up dummy path data
             * to satisfy it doesn't seem much cleaner than knowing what it does.
             */
            gather->plan.startup_cost = top_plan->startup_cost +
                parallel_setup_cost;
            gather->plan.total_cost = top_plan->total_cost +
                parallel_setup_cost + parallel_tuple_cost * top_plan->plan_rows;
            gather->plan.plan_rows = top_plan->plan_rows;
            gather->plan.plan_width = top_plan->plan_width;
            gather->plan.parallel_aware = false;
            gather->plan.parallel_safe = false;
    
            /* use parallel mode for parallel plans. */
            //使用并行模式
            root->glob->parallelModeNeeded = true;
    
            top_plan = &gather->plan;
        }
    
        /*
         * If any Params were generated, run through the plan tree and compute
         * each plan node's extParam/allParam sets.  Ideally we'd merge this into
         * set_plan_references' tree traversal, but for now it has to be separate
         * because we need to visit subplans before not after main plan.
         * 如果生成了Params，运行计划树并计算每个计划节点的extParam/allParam集合。
         * 理想情况下，我们应该将其合并到set_plan_references的树遍历中，
         * 但是现在它必须是独立的，因为需要在主计划之前而不是之后访问子计划。
         */
        if (glob->paramExecTypes != NIL)
        {
            Assert(list_length(glob->subplans) == list_length(glob->subroots));
            forboth(lp, glob->subplans, lr, glob->subroots)
            {
                Plan       *subplan = (Plan *) lfirst(lp);
                PlannerInfo *subroot = lfirst_node(PlannerInfo, lr);
    
                SS_finalize_plan(subroot, subplan);
            }
            SS_finalize_plan(root, top_plan);
        }
    
        /* final cleanup of the plan */
        //最后的清理工作
        Assert(glob->finalrtable == NIL);
        Assert(glob->finalrowmarks == NIL);
        Assert(glob->resultRelations == NIL);
        Assert(glob->nonleafResultRelations == NIL);
        Assert(glob->rootResultRelations == NIL);
        top_plan = set_plan_references(root, top_plan);
        /* ... and the subplans (both regular subplans and initplans) */
        Assert(list_length(glob->subplans) == list_length(glob->subroots));
        forboth(lp, glob->subplans, lr, glob->subroots)
        {
            Plan       *subplan = (Plan *) lfirst(lp);
            PlannerInfo *subroot = lfirst_node(PlannerInfo, lr);
    
            lfirst(lp) = set_plan_references(subroot, subplan);
        }
    
        /* build the PlannedStmt result */
        //构建PlannedStmt结构
        result = makeNode(PlannedStmt);
    
        result->commandType = parse->commandType;//命令类型
        result->queryId = parse->queryId;
        result->hasReturning = (parse->returningList != NIL);
        result->hasModifyingCTE = parse->hasModifyingCTE;
        result->canSetTag = parse->canSetTag;
        result->transientPlan = glob->transientPlan;
        result->dependsOnRole = glob->dependsOnRole;
        result->parallelModeNeeded = glob->parallelModeNeeded;
        result->planTree = top_plan;//执行计划(这是后续执行SQL使用到的最重要的地方)
        result->rtable = glob->finalrtable;
        result->resultRelations = glob->resultRelations;
        result->nonleafResultRelations = glob->nonleafResultRelations;
        result->rootResultRelations = glob->rootResultRelations;
        result->subplans = glob->subplans;
        result->rewindPlanIDs = glob->rewindPlanIDs;
        result->rowMarks = glob->finalrowmarks;
        result->relationOids = glob->relationOids;
        result->invalItems = glob->invalItems;
        result->paramExecTypes = glob->paramExecTypes;
        /* utilityStmt should be null, but we might as well copy it */
        result->utilityStmt = parse->utilityStmt;
        result->stmt_location = parse->stmt_location;
        result->stmt_len = parse->stmt_len;
    
        result->jitFlags = PGJIT_NONE;
        if (jit_enabled && jit_above_cost >= 0 &&
            top_plan->total_cost > jit_above_cost)
        {
            result->jitFlags |= PGJIT_PERFORM;
    
            /*
             * Decide how much effort should be put into generating better code.
             */
            if (jit_optimize_above_cost >= 0 &&
                top_plan->total_cost > jit_optimize_above_cost)
                result->jitFlags |= PGJIT_OPT3;
            if (jit_inline_above_cost >= 0 &&
                top_plan->total_cost > jit_inline_above_cost)
                result->jitFlags |= PGJIT_INLINE;
    
            /*
             * Decide which operations should be JITed.
             */
            if (jit_expressions)
                result->jitFlags |= PGJIT_EXPR;
            if (jit_tuple_deforming)
                result->jitFlags |= PGJIT_DEFORM;
        }
    
        return result;
    } 
    

**pg_plan_queries &pg_plan_query**

    
    
     /*
      * Generate plans for a list of already-rewritten queries.
      *
      * For normal optimizable statements, invoke the planner.  For utility
      * statements, just make a wrapper PlannedStmt node.
      *
      * The result is a list of PlannedStmt nodes.
      */
     List *
     pg_plan_queries(List *querytrees, int cursorOptions, ParamListInfo boundParams)
     {
         List       *stmt_list = NIL;
         ListCell   *query_list;
     
         foreach(query_list, querytrees)
         {
             Query      *query = lfirst_node(Query, query_list);
             PlannedStmt *stmt;
     
             if (query->commandType == CMD_UTILITY)
             {
                 /* Utility commands require no planning. */
                 stmt = makeNode(PlannedStmt);
                 stmt->commandType = CMD_UTILITY;
                 stmt->canSetTag = query->canSetTag;
                 stmt->utilityStmt = query->utilityStmt;
                 stmt->stmt_location = query->stmt_location;
                 stmt->stmt_len = query->stmt_len;
             }
             else
             {
                 stmt = pg_plan_query(query, cursorOptions, boundParams);
             }
     
             stmt_list = lappend(stmt_list, stmt);
         }
     
         return stmt_list;
     }
    
    /*
     * Generate a plan for a single already-rewritten query.
     * This is a thin wrapper around planner() and takes the same parameters.
     */
    PlannedStmt *
    pg_plan_query(Query *querytree, int cursorOptions, ParamListInfo boundParams)
    {
        PlannedStmt *plan;
    
        /* Utility commands have no plans. */
        if (querytree->commandType == CMD_UTILITY)
            return NULL;
    
        /* Planner must have a snapshot in case it calls user-defined functions. */
        Assert(ActiveSnapshotSet());
    
        TRACE_POSTGRESQL_QUERY_PLAN_START();
    
        if (log_planner_stats)
            ResetUsage();
    
        /* call the optimizer */
        plan = planner(querytree, cursorOptions, boundParams);
    
        if (log_planner_stats)
            ShowUsage("PLANNER STATISTICS");
    
    #ifdef COPY_PARSE_PLAN_TREES
        /* Optional debugging check: pass plan tree through copyObject() */
        {
            PlannedStmt *new_plan = copyObject(plan);
    
            /*
             * equal() currently does not have routines to compare Plan nodes, so
             * don't try to test equality here.  Perhaps fix someday?
             */
    #ifdef NOT_USED
            /* This checks both copyObject() and the equal() routines... */
            if (!equal(new_plan, plan))
                elog(WARNING, "copyObject() failed to produce an equal plan tree");
            else
    #endif
                plan = new_plan;
        }
    #endif
    
    #ifdef WRITE_READ_PARSE_PLAN_TREES
        /* Optional debugging check: pass plan tree through outfuncs/readfuncs */
        {
            char       *str;
            PlannedStmt *new_plan;
    
            str = nodeToString(plan);
            new_plan = stringToNodeWithLocations(str);
            pfree(str);
    
            /*
             * equal() currently does not have routines to compare Plan nodes, so
             * don't try to test equality here.  Perhaps fix someday?
             */
    #ifdef NOT_USED
            /* This checks both outfuncs/readfuncs and the equal() routines... */
            if (!equal(new_plan, plan))
                elog(WARNING, "outfuncs/readfuncs failed to produce an equal plan tree");
            else
    #endif
                plan = new_plan;
        }
    #endif
    
        /*
         * Print plan if debugging.
         */
        if (Debug_print_plan)
            elog_node_display(LOG, "plan", plan, Debug_pretty_print);
    
        TRACE_POSTGRESQL_QUERY_PLAN_DONE();
    
        return plan;
    }
    

### 二、参考资料

[allpaths.c](https://doxygen.postgresql.org/allpaths_8c_source.html)  
[PG Document:Query
Planning](https://www.postgresql.org/docs/11/static/runtime-config-query.html)

